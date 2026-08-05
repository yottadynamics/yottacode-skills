---
name: ansible-automation
description: Write, review, and test Ansible playbooks, roles, and inventories so they stay idempotent, least-privilege, and safe to run against production. Use when authoring or reviewing a playbook or role, debugging a non-idempotent task, designing inventory and variable layout, setting up molecule or ansible-lint, or hardening automation for an audited environment.
license: MIT
metadata:
  author: yottacode
---

# Ansible automation

Ansible's failure mode is not a crash — it is a playbook that reports `changed` forever, escalates more than it needs, or applies a config nobody reviewed. Work the checklist below in order; each step ends in something you can verify.

## 1. Establish the ground truth before writing a task

Never guess at a module's interface. `ansible-doc` is authoritative for the version actually installed on the control node:

```bash
ansible-doc --list | grep -i lineinfile    # confirm the module exists, get its FQCN
ansible-doc ansible.builtin.lineinfile     # required params, defaults, check_mode support, RETURN
ansible-doc ansible.builtin.lineinfile --snippet   # minimal task block to start from
```

Read the `RETURN` section before writing any `changed_when` or `failed_when` — it tells you which fields actually exist on the registered result. If a module is missing, the collection probably is too: `ansible-galaxy collection list <namespace.name>`.

See `references/patterns.md` for the fuller `ansible-doc` workflow, including plugin-type lookups.

## 2. Make every task idempotent

The bar: a second run reports `changed=0`. Prefer a module with state management over `command`/`shell`.

```yaml
- name: Ensure nginx is installed
  ansible.builtin.dnf:
    name: nginx
    state: present
  become: true
```

When no module exists, make the raw command idempotent explicitly — `creates:`, or `changed_when` derived from real output:

```yaml
- name: Apply database migrations
  ansible.builtin.command:
    cmd: /opt/myapp/bin/migrate up
  register: migrate_result
  become: true
  become_user: myapp
  changed_when: "'Migrated' in migrate_result.stdout"
  failed_when: migrate_result.rc != 0
```

Verify by running the playbook twice against staging. Second run must show `changed=0 unreachable=0 failed=0`. In CI, molecule's default sequence does this for you via its `idempotence` step.

## 3. Escalate narrowly

Set `become: true` on the tasks that need root, not on the play. A play-level `become` grants root to every task including the ones that only read state or call an API from `localhost`.

Where a whole role genuinely needs root, put `become: true` on the role include rather than the play, so the boundary stays visible in the playbook.

## 4. Name tasks so `--check --diff` reads like a change request

Every task needs a `name:` describing the end state, not the action. Dry-run output is the artifact a reviewer approves:

```bash
ansible-playbook playbooks/site.yml -i inventories/production/hosts.yml --check --diff
```

Read the diff line by line. Unexpected changes mean configuration drift on the target, not a bug in your playbook — investigate before applying.

Use `validate:` on any task that writes a config file a daemon parses, so a bad template can never take the service down:

```yaml
- name: Ensure sshd permits only key-based authentication
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PasswordAuthentication'
    line: 'PasswordAuthentication no'
    validate: '/usr/sbin/sshd -t -f %s'
  become: true
  notify: Restart sshd
```

## 5. Keep secrets out of output as well as out of Git

- Encrypt with `ansible-vault`, and keep encrypted values in a separate `vault.yml` from the plain-text file that references them. That way `grep -r db_password` still finds usage without decrypting anything.
- Put `no_log: true` on any task handling key material. Remember it also hides the failure message — remove it temporarily when debugging, never leave it off.
- For `loop`, set `loop_control.label` so the loop item does not print.

```yaml
# group_vars/all/vault.yml  — encrypted
vault_db_password: "…"

# group_vars/all/main.yml   — plain text, greppable
db_password: "{{ vault_db_password }}"
```

## 6. Structure roles so they can be tested in isolation

One role, one concern (`sshd`, `auditd`, `firewalld` — not `server_setup`). Namespace every variable with the role name. `defaults/main.yml` holds the knobs a consumer may override; `vars/main.yml` holds internal constants that should be hard to override.

Handler names are static strings — **they cannot be templated**. `notify: Restart {{ service_name }}` silently matches nothing and the service never restarts. For a role that manages a parameterised service, use a static handler name and `listen`:

```yaml
# tasks/main.yml
- name: Deploy service TLS certificate
  ansible.builtin.copy:
    content: "{{ vault_tls_certificate }}"
    dest: "/etc/pki/tls/certs/{{ service_name }}.crt"
    owner: root
    group: "{{ service_name }}"
    mode: '0644'
  become: true
  notify: Restart managed service

# handlers/main.yml
- name: Restart managed service
  listen: Restart managed service
  ansible.builtin.systemd:
    name: "{{ service_name }}"     # templating the module arg is fine
    state: restarted
  become: true
```

## 7. Lint and test before it reaches a host

```bash
ansible-lint --profile production --strict
molecule test        # run from the role directory
```

`references/patterns.md` has working `.ansible-lint`, `molecule.yml`, `verify.yml`, and CI pipeline configurations.

When writing assertions in `verify.yml`, remember **a list under `failed_when` is ANDed**. Two conditions in one list only fail when both hold — usually not what you want:

```yaml
# WRONG — passes when the file exists with the wrong mode
failed_when:
  - not sshd_stat.stat.exists
  - sshd_stat.stat.mode != '0600'

# RIGHT — one assertion per condition
- name: Verify sshd_config exists with restrictive permissions
  ansible.builtin.assert:
    that:
      - sshd_stat.stat.exists
      - sshd_stat.stat.mode == '0600'
    fail_msg: "sshd_config missing or mode is {{ sshd_stat.stat.mode | default('absent') }}"
```

## 8. Fail early on missing inputs

Assert required variables at the top of the play rather than letting a half-applied change fail three tasks in:

```yaml
- name: Validate required deployment variables
  ansible.builtin.assert:
    that:
      - deploy_version is defined
      - deploy_version is match('^[0-9]+\.[0-9]+\.[0-9]+$')
      - deploy_environment in ['staging', 'production']
    fail_msg: "deploy_version and deploy_environment must be set (got '{{ deploy_version | default('UNDEFINED') }}' / '{{ deploy_environment | default('UNDEFINED') }}')"
  run_once: true
```

Use `| mandatory` for values with no sensible default, `| default(x)` where there is one, and `| default(omit)` to skip an optional module parameter entirely.

## Output format for playbook and role reviews

```markdown
## Ansible review

- Verdict: ready / needs changes
- Idempotency: <tasks that will report changed on every run>
- Privilege: <play-level become, unnecessary root>
- Secrets: <plaintext values, missing no_log>
- Structure: <role scope, variable namespacing, defaults vs vars>
- Testing: <lint profile, molecule coverage, check-mode safety>

## Required changes
- `path/to/file.yml:LINE` — <what to change and why>
```

## Pitfalls

1. **Templated handler names** — `notify: Restart {{ svc }}` never fires. Static name, template the module argument.
2. **A `failed_when` list is ANDed** — for OR semantics, use `or` explicitly or split into separate assertions.
3. **`shell`/`command` where a module exists** — you lose idempotency, change detection, and check-mode support.
4. **`become: true` at play level** — grants root to `delegate_to: localhost` tasks and read-only checks.
5. **`ignore_errors: true` without a comment** — silently converts a failed change into a green run. Prefer `failed_when` with a precise condition.
6. **Disabling SELinux to make something work** — use `seboolean`, `sefcontext`, and `audit2allow` to find the real policy gap.
7. **Vault password files inside the repo** — keep them outside the working tree or fetch from a secrets manager.
8. **Unpinned collections** — pin exact versions in `requirements.yml`; an upgraded collection can change module behaviour under you.
9. **`gather_facts: true` when nothing uses facts** — it costs a round trip per host. But check first: `ansible_date_time`, `ansible_hostname`, and most `ansible_*` variables all require it.
10. **Hardcoded IP addresses** — use inventory hostnames; IPs change and are invisible to `grep` during a migration.

## References

- `references/patterns.md` — production-ready task, role, inventory, testing, and CI patterns.
- `references/compliance.md` — audit logging, vault strategy, SSH/kernel hardening, and control mappings for regulated environments.
- [Ansible variable precedence](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable)
- [Ansible Vault guide](https://docs.ansible.com/ansible/latest/vault_guide/index.html)
- [ansible-lint profiles](https://ansible.readthedocs.io/projects/lint/profiles/)
- [Molecule](https://ansible.readthedocs.io/projects/molecule/)
