# Ansible in audited environments

Additional practice for automation that is reviewed by auditors as well as
engineers. Everything in `SKILL.md` still applies — this layers evidence and
traceability on top.

Treat the control mappings below as a starting point for a conversation with
your auditor, not as a certification. No configuration file makes a system
compliant on its own.

## Control mapping

| Framework | What Ansible is usually asked to demonstrate |
|---|---|
| HIPAA | Encrypted secrets at rest, access logging, least-privilege execution, change audit trail |
| PCI-DSS | Network segmentation reflected in inventory, file integrity, access control, change management |
| FedRAMP | FIPS-validated crypto modules, continuous monitoring, STIG automation, POA&M tracking |
| SOC 2 | Change management (PR + CI), monitoring callbacks, access reviews, incident response runbooks |
| NIST 800-53 | AC (access control), AU (audit), CM (configuration management), IA, SC |

## Controls worth enforcing in the repo

1. **Run records** — every production run produces a structured entry: play name, inventory, user, start/end, changed/failed counts, and the Git SHA of the playbook source.
2. **Encrypted secrets** — every credential, token, and key under `ansible-vault`, with `no_log` on the tasks that touch them.
3. **Narrow escalation** — `become` per task or per role, never per play.
4. **Key-based SSH only** — no password auth, `StrictHostKeyChecking=yes` against production, keys rotated on a schedule.
5. **Segmented inventory** — one group per security zone; a play never spans zones unless that is the explicit intent.
6. **Reviewed changes** — everything through Git, PR-reviewed, CI-validated (lint + molecule + `--check`), tagged for production.

## Run records

Fact gathering must be on for `ansible_date_time`.

```yaml
- name: Record deployment metadata
  ansible.builtin.set_fact:
    deployment_metadata:
      git_sha: "{{ lookup('pipe', 'git rev-parse HEAD') }}"
      git_branch: "{{ lookup('pipe', 'git rev-parse --abbrev-ref HEAD') }}"
      deployer: "{{ lookup('env', 'USER') }}"
      timestamp: "{{ ansible_date_time.iso8601 }}"
      playbook: "{{ ansible_play_name }}"
      inventory: "{{ inventory_file }}"
  run_once: true
  delegate_to: localhost

- name: Write the deployment record
  ansible.builtin.lineinfile:
    path: /var/log/ansible/deployment-audit.log
    line: "{{ deployment_metadata | to_json }}"
    create: true
    owner: ansible
    group: audit
    mode: '0640'
  delegate_to: localhost
  run_once: true
  become: true
```

The `git rev-parse` lookups run on the control node and fail outside a
checkout — guard them with `| default('unknown')` if the playbook may run from
an unpacked artifact.

For per-play logging, enable `community.general.log_plays` via
`callbacks_enabled` (see `patterns.md`) rather than replacing the stdout
callback.

## Vault strategy

```bash
# Encrypt one value inline — best for files that mix secret and non-secret vars
ansible-vault encrypt_string --vault-id prod@prompt 'SuperSecretPassword' --name 'vault_db_password'

# Encrypt a whole file
ansible-vault encrypt inventories/production/group_vars/all/vault.yml

# Distinct vault IDs per environment, so a dev key cannot decrypt production
ansible-playbook playbooks/site.yml \
  -i inventories/production/hosts.yml \
  --vault-id prod@~/.vault_prod
```

Keep the vault password file outside the repository. Better still, pull secrets
at run time so they never land on disk:

```yaml
- name: Retrieve database credentials from HashiCorp Vault
  ansible.builtin.set_fact:
    db_password: "{{ lookup('community.hashi_vault.hashi_vault',
      'secret/data/production/database:password',
      url='https://vault.internal.example.com:8200',
      auth_method='approle',
      role_id=lookup('env', 'VAULT_ROLE_ID'),
      secret_id=lookup('env', 'VAULT_SECRET_ID')) }}"
  no_log: true

- name: Retrieve an API key from AWS Secrets Manager
  ansible.builtin.set_fact:
    api_key: "{{ lookup('amazon.aws.aws_secret', 'production/api-key', region='us-east-1') }}"
  no_log: true
```

`set_fact` values persist for the rest of the play and appear in
`--extra-vars` dumps. Scope them to the tasks that need them and avoid
registering them into host facts.

## SSH hardening template

```jinja
{# roles/common/templates/sshd_config.j2 #}
# Managed by Ansible — do not edit manually
# Applied: {{ ansible_date_time.iso8601 }}
# Source: {{ role_name }}

PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
KbdInteractiveAuthentication no

MaxAuthTries 3
MaxSessions 5
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2

SyslogFacility AUTH
LogLevel VERBOSE

AllowGroups {{ sshd_allowed_groups | join(' ') }}

{% if sshd_fips_mode | default(false) %}
{# These algorithm lists restrict SSH to FIPS-approved primitives. They do   #}
{# NOT by themselves make the host FIPS-compliant — that requires the OS to  #}
{# be running a FIPS-validated crypto module (e.g. `fips-mode-setup          #}
{# --enable` on RHEL). Verify with `fips-mode-setup --check`.                #}
Ciphers aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes128-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512,hmac-sha2-256
KexAlgorithms ecdh-sha2-nistp384,ecdh-sha2-nistp256,diffie-hellman-group16-sha512,diffie-hellman-group14-sha256
{% endif %}
```

Do not emit a `Protocol 2` line. OpenSSH removed the directive in 7.6 and
sshd refuses to start with it present.

Always pair the template with `validate: '/usr/sbin/sshd -t -f %s'` so a bad
render cannot lock you out, and verify the result with `sshd -T` rather than
grepping the file.

## File permission discipline

Every `copy`, `template`, and `file` task states `owner`, `group`, and `mode`
explicitly. Without `mode`, the result depends on the remote umask and drifts
between hosts.

```yaml
- name: Deploy application configuration
  ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/myapp/app.conf
    owner: myapp
    group: myapp
    mode: '0640'
    validate: '/opt/myapp/bin/validate-config %s'
    backup: true
  become: true
  notify: Restart myapp
```

## Kernel parameter baselines

```yaml
- name: Apply security kernel parameters
  ansible.posix.sysctl:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    sysctl_set: true
    state: present
    reload: true
    sysctl_file: /etc/sysctl.d/99-security-hardening.conf
  loop:
    - { name: 'net.ipv4.conf.all.accept_redirects', value: '0' }
    - { name: 'net.ipv4.conf.default.accept_redirects', value: '0' }
    - { name: 'net.ipv4.conf.all.send_redirects', value: '0' }
    - { name: 'net.ipv4.conf.all.accept_source_route', value: '0' }
    - { name: 'net.ipv4.conf.all.log_martians', value: '1' }
    - { name: 'net.ipv4.icmp_echo_ignore_broadcasts', value: '1' }
    - { name: 'net.ipv4.tcp_syncookies', value: '1' }
    - { name: 'net.ipv6.conf.all.accept_redirects', value: '0' }
    - { name: 'net.ipv6.conf.default.accept_redirects', value: '0' }
    - { name: 'kernel.randomize_va_space', value: '2' }
    - { name: 'kernel.dmesg_restrict', value: '1' }
    - { name: 'kernel.kptr_restrict', value: '2' }
    - { name: 'fs.suid_dumpable', value: '0' }
  loop_control:
    label: "{{ item.name }} = {{ item.value }}"
  become: true
```

Rather than maintaining a hand-rolled baseline, prefer an audited upstream role
and override only what your environment requires — see the ansible-lockdown
STIG and CIS roles in the references below.

## Supply chain

- Pin every collection to an exact version in `requirements.yml`; treat an upgrade as a reviewable change.
- Vendor collections into `./collections/` and set `collections_path` so a run cannot silently pick up a system-wide plugin.
- Install from a controlled mirror (Automation Hub, an internal Galaxy proxy, or a checked-in tarball) where the environment requires provenance.
- Verify signatures with `ansible-galaxy collection install --verify` where the publisher signs releases.

## Evidence checklist

Before calling an automation change ready for an audited environment:

- [ ] `ansible-lint --profile production --strict` passes.
- [ ] `molecule test` passes for every role touched, including the idempotence step.
- [ ] `--check --diff` against staging shows only intended changes.
- [ ] No plaintext credential in the diff, the repo, or the run output.
- [ ] `become` appears per task or per role, never per play.
- [ ] The change is on a PR with a reviewer, and the merge commit SHA is recorded in the run log.
- [ ] Rollback path is written down and has been executed at least once in staging.

## References

- [ansible-lockdown RHEL 9 STIG role](https://github.com/ansible-lockdown/RHEL9-STIG)
- [ansible-lockdown RHEL 9 CIS role](https://github.com/ansible-lockdown/RHEL9-CIS)
- [NIST SP 800-53 Rev 5](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- [Ansible Vault guide](https://docs.ansible.com/ansible/latest/vault_guide/index.html)
