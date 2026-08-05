# Ansible patterns

Working configurations and task patterns referenced from `SKILL.md`.

## Project layout

```text
project-root/
├── ansible.cfg                    # project-scoped; never rely on /etc/ansible/ansible.cfg
├── requirements.yml               # pinned collection + role dependencies
├── inventories/
│   ├── dev/
│   │   ├── hosts.yml
│   │   ├── group_vars/
│   │   │   ├── all.yml
│   │   │   └── webservers.yml
│   │   └── host_vars/
│   ├── staging/
│   └── production/
│       ├── hosts.yml
│       ├── group_vars/
│       │   └── all/
│       │       ├── main.yml       # plain-text references
│       │       └── vault.yml      # ansible-vault encrypted values
│       └── host_vars/
├── roles/
│   └── common/
│       ├── tasks/main.yml
│       ├── handlers/main.yml
│       ├── templates/
│       ├── defaults/main.yml      # consumer-overridable knobs
│       ├── vars/main.yml          # role-internal constants
│       ├── meta/main.yml
│       └── molecule/default/
│           ├── molecule.yml
│           ├── converge.yml
│           └── verify.yml
├── playbooks/
│   ├── site.yml
│   └── rollback.yml
├── collections/                   # installed locally, gitignored
└── .ansible-lint
```

## ansible.cfg

```ini
[defaults]
collections_path = ./collections
roles_path = ./roles
retry_files_enabled = False
nocows = True

# Timing and per-play logs. Leave stdout_callback at its default (`default`)
# for interactive runs — the `json` and `yaml` callbacks suppress the
# profile_tasks/timer output below and buffer everything until the run ends,
# which makes a long production play look hung and makes --diff unreadable.
callbacks_enabled = ansible.posix.profile_tasks, ansible.posix.timer, community.general.log_plays

[callback_log_plays]
log_folder = /var/log/ansible/plays

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=yes
pipelining = True
```

For machine-readable output in CI, set the callback per-invocation instead of globally:

```bash
ANSIBLE_STDOUT_CALLBACK=ansible.posix.json ansible-playbook playbooks/site.yml -i inventories/staging/hosts.yml
```

## requirements.yml

```yaml
---
collections:
  # Pin exact versions — an upgraded collection can change module behaviour.
  - name: ansible.posix
    version: "1.5.4"
  - name: community.general
    version: "8.6.0"
  - name: community.crypto
    version: "2.18.0"
```

Install with `ansible-galaxy collection install -r requirements.yml`. Where the
collection publisher signs releases, add `--verify` to check the signature.

## ansible-doc workflow

```bash
# Full documentation: parameters, defaults, check_mode support, RETURN values
ansible-doc ansible.builtin.copy

# Just the example block — fastest way to a working snippet
ansible-doc ansible.builtin.template --snippet

# Everything in a collection
ansible-doc --list ansible.posix

# Non-module plugin types
ansible-doc -t callback ansible.posix.json
ansible-doc -t lookup file
ansible-doc -t filter regex_replace
ansible-doc -t inventory amazon.aws.aws_ec2

# Find a module when you don't know its name
ansible-doc --list | grep -i firewall
```

Reach for it when: using an unfamiliar module; a task fails with a parameter
error; checking whether a module supports `--check`; reviewing someone else's
playbook against the installed version; writing `changed_when`/`failed_when`
(read `RETURN` first); or upgrading ansible-core and auditing deprecations.

## Error handling with rollback

```yaml
- name: Deploy application with rollback safety
  block:
    - name: Deploy new application version
      ansible.builtin.copy:
        src: "app-{{ app_version }}.tar.gz"
        dest: /opt/app/
        owner: app
        group: app
        mode: '0640'
      become: true

    - name: Run database migration
      ansible.builtin.command:
        cmd: /opt/app/bin/migrate up
      become: true
      become_user: app
      register: migration_result
      changed_when: "'migrated' in migration_result.stdout"
      failed_when: migration_result.rc != 0

  rescue:
    - name: Roll back application to previous version
      ansible.builtin.copy:
        src: "app-{{ app_previous_version | mandatory }}.tar.gz"
        dest: /opt/app/
        owner: app
        group: app
        mode: '0640'
      become: true

    - name: Notify operations of the failed deployment
      ansible.builtin.uri:
        url: "{{ deploy_webhook_url }}"
        method: POST
        body_format: json
        body:
          text: "FAILED: {{ app_version }} on {{ inventory_hostname }} — rolled back to {{ app_previous_version }}"
      delegate_to: localhost
      no_log: true

  always:
    - name: Record the deployment attempt
      ansible.builtin.lineinfile:
        path: /var/log/deployment-audit.log
        line: "{{ ansible_date_time.iso8601 }} | {{ app_version }} | {{ inventory_hostname }} | {{ 'FAILED' if ansible_failed_result is defined else 'SUCCESS' }}"
        create: true
        mode: '0640'
      become: true
```

`ansible_date_time` comes from fact gathering — this play needs
`gather_facts: true`.

## Rolling deployment

`become` sits on the role, not the play, so the `localhost` load-balancer calls
never run as root.

```yaml
---
- name: Rolling deployment with health checks
  hosts: webservers
  serial: "25%"
  max_fail_percentage: 10
  order: sorted

  pre_tasks:
    - name: Remove host from the load balancer
      ansible.builtin.uri:
        url: "https://lb.internal/api/v1/backends/{{ inventory_hostname }}"
        method: DELETE
        headers:
          Authorization: "Bearer {{ vault_lb_api_token }}"
        status_code: [200, 204]
      delegate_to: localhost
      no_log: true

    - name: Wait for in-flight connections to drain
      ansible.builtin.wait_for:
        timeout: 30
      delegate_to: localhost

  roles:
    - role: webserver
      become: true
      vars:
        app_version: "{{ deploy_version | mandatory }}"

  post_tasks:
    - name: Wait for the application health check to pass
      ansible.builtin.uri:
        url: "http://{{ inventory_hostname }}:{{ app_port }}/health"
        status_code: 200
      register: health_check
      retries: 5
      delay: 10
      until: health_check.status == 200
      delegate_to: localhost

    - name: Re-add host to the load balancer
      ansible.builtin.uri:
        url: "https://lb.internal/api/v1/backends"
        method: POST
        body_format: json
        body:
          hostname: "{{ inventory_hostname }}"
          port: "{{ app_port }}"
        headers:
          Authorization: "Bearer {{ vault_lb_api_token }}"
        status_code: [200, 201]
      delegate_to: localhost
      no_log: true
```

## Canary deployment

```yaml
---
- name: Canary deployment — phase 1
  hosts: webservers_canary
  serial: 1
  roles:
    - role: webserver
      become: true
      vars:
        app_version: "{{ deploy_version | mandatory }}"
  post_tasks:
    - name: Hold for the canary observation window
      ansible.builtin.pause:
        minutes: 15
        prompt: "Canary deployed. Check dashboards. Enter to continue, Ctrl-C A to abort."
      run_once: true
      delegate_to: localhost

- name: Full deployment — phase 2
  hosts: webservers:!webservers_canary
  serial: "33%"
  max_fail_percentage: 5
  roles:
    - role: webserver
      become: true
      vars:
        app_version: "{{ deploy_version | mandatory }}"
```

`ansible.builtin.pause` is interactive — it blocks forever under a CI runner
with no TTY. Gate the phase on a CI approval step instead when running headless.

## Variable precedence

Ansible defines 22 precedence levels. The ones that decide most arguments,
lowest to highest:

| Source | Use for |
|---|---|
| `role/defaults/main.yml` | Values a consumer is expected to override |
| Inventory `group_vars/all` | Environment-wide settings |
| Inventory `group_vars/<group>` | Per-group settings |
| Inventory `host_vars/<host>` | Per-host overrides |
| Play `vars:` | Values scoped to one play |
| `role/vars/main.yml` | Role internals that should be hard to override |
| `set_fact` / `register` | Runtime-computed values |
| `--extra-vars` / `-e` | Deploy-time parameters; wins over everything |

## Variable naming and guards

```yaml
# Namespace every role variable — generic names collide silently across roles.
nginx_worker_processes: auto
nginx_listen_port: 443
nginx_tls_cert_path: /etc/pki/tls/certs/nginx.crt

# | mandatory for values with no sensible default
dest: "/etc/{{ app_name | mandatory }}/app.conf"

# | default() where there is one
log_retention_days: "{{ app_log_retention | default(90) }}"

# | default(omit) to skip an optional module parameter entirely
- name: Create user account
  ansible.builtin.user:
    name: "{{ item.username }}"
    groups: "{{ item.groups | default(omit) }}"
    shell: "{{ item.shell | default('/bin/bash') }}"
    expires: "{{ item.expires | default(omit) }}"
  loop: "{{ users }}"
  loop_control:
    label: "{{ item.username }}"
  become: true
```

## Handlers

Handlers fire once, at the end of the play, only if a notifying task reported
`changed`. Handler names are literal strings and cannot be templated.

```yaml
# tasks/main.yml
- name: Deploy nginx main configuration
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: nginx
    mode: '0640'
    validate: '/usr/sbin/nginx -t -c %s'
  become: true
  notify: Restart nginx

- name: Deploy nginx virtual host
  ansible.builtin.template:
    src: vhost.conf.j2
    dest: /etc/nginx/conf.d/myapp.conf
    owner: root
    group: nginx
    mode: '0640'
  become: true
  notify: Reload nginx      # reload is less disruptive than restart

# handlers/main.yml
- name: Restart nginx
  ansible.builtin.systemd:
    name: nginx
    state: restarted
    daemon_reload: true
  become: true

- name: Reload nginx
  ansible.builtin.systemd:
    name: nginx
    state: reloaded
  become: true
```

To make handlers run mid-play — for example before a health check that depends
on the new config — flush them explicitly:

```yaml
- name: Apply pending restarts before the health check
  ansible.builtin.meta: flush_handlers
```

## Role metadata

```yaml
# meta/main.yml
---
galaxy_info:
  role_name: nginx
  author: platform-team
  description: Production nginx with TLS and hardening
  min_ansible_version: "2.15"
  platforms:
    - name: EL
      versions: ["9"]

dependencies:
  # Hard dependencies only — roles that MUST run first.
  # Optional ordering belongs in the playbook, not here.
  - role: common
  - role: firewalld
```

## ansible-lint

```yaml
# .ansible-lint
---
profile: production
strict: true

# Add skips only with a comment explaining why.
skip_list: []

exclude_paths:
  - collections/
  - .cache/
```

## Molecule

```yaml
# molecule/default/molecule.yml
---
dependency:
  name: galaxy
  options:
    requirements-file: requirements.yml

driver:
  name: podman        # rootless; no daemon required

platforms:
  - name: el9-instance
    image: "quay.io/centos/centos:stream9"
    command: /usr/sbin/init
    systemd: true
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    capabilities:
      - SYS_ADMIN

provisioner:
  name: ansible
  config_options:
    defaults:
      callbacks_enabled: ansible.posix.profile_tasks

verifier:
  name: ansible
```

The driver sets the connection plugin — do not also set `ansible_connection` in
`provisioner.inventory`, it will fight the driver.

```yaml
# molecule/default/converge.yml
---
- name: Converge
  hosts: all
  become: true
  roles:
    - role: "{{ lookup('env', 'MOLECULE_PROJECT_DIRECTORY') | basename }}"
      vars:
        sshd_allowed_groups:
          - ssh-users
```

```yaml
# molecule/default/verify.yml
---
- name: Verify
  hosts: all
  become: true
  gather_facts: true
  tasks:
    - name: Collect sshd_config metadata
      ansible.builtin.stat:
        path: /etc/ssh/sshd_config
      register: sshd_stat

    # One assert per fact. A list under failed_when is ANDed, so bundling
    # these into a single failed_when would pass when the file exists with
    # the wrong mode.
    - name: Verify sshd_config exists with restrictive permissions
      ansible.builtin.assert:
        that:
          - sshd_stat.stat.exists
          - sshd_stat.stat.mode == '0600'
        fail_msg: "sshd_config missing or mode is {{ sshd_stat.stat.mode | default('absent') }}"

    - name: Verify SELinux is enforcing
      ansible.builtin.command:
        cmd: getenforce
      register: selinux_status
      changed_when: false
      failed_when: selinux_status.stdout != 'Enforcing'

    - name: Verify password authentication is disabled
      ansible.builtin.command:
        cmd: sshd -T
      register: sshd_effective
      changed_when: false
      failed_when: "'passwordauthentication no' not in sshd_effective.stdout | lower"

    - name: Verify auditd is running
      ansible.builtin.service_facts:

    - name: Assert auditd is active
      ansible.builtin.assert:
        that: ansible_facts.services['auditd.service'].state == 'running'
        fail_msg: "auditd is not running"
```

`sshd -T` reports the *effective* configuration, which is what you actually
care about — a `grep` of the file misses `Include` directives and later
overrides.

## CI pipeline

```yaml
# .github/workflows/ansible-ci.yml
---
name: Ansible CI
on:
  pull_request:
    paths:
      - 'roles/**'
      - 'playbooks/**'
      - 'inventories/**'
      - 'requirements.yml'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ansible/ansible-lint@v24
        with:
          args: "--profile production --strict"

  molecule:
    runs-on: ubuntu-latest
    needs: lint
    strategy:
      fail-fast: false
      matrix:
        role: [common, webserver, database]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - name: Install dependencies
        run: |
          pip install 'molecule' 'molecule-plugins[podman]' 'ansible-core'
          ansible-galaxy collection install -r requirements.yml
      - name: Run molecule test
        run: molecule test
        working-directory: "roles/${{ matrix.role }}"

  check-staging:
    runs-on: self-hosted
    needs: molecule
    environment: staging      # GitHub environment gate = manual approval
    steps:
      - uses: actions/checkout@v4
      - name: Dry-run against staging
        run: |
          ansible-playbook playbooks/site.yml \
            -i inventories/staging/hosts.yml \
            --check --diff \
            --vault-password-file "$VAULT_PASSWORD_FILE"
        env:
          VAULT_PASSWORD_FILE: ${{ runner.temp }}/vault-pass
```

## Common task patterns

### Package pinning

```yaml
- name: Install packages at pinned versions
  ansible.builtin.dnf:
    name:
      - "nginx-{{ nginx_version }}"
      - "openssl-{{ openssl_version }}"
    state: present
  become: true
```

`dnf` matches on the `name-version` prefix, so `nginx-1.24.0` pins the version
but not the release. Where the exact build matters, give the full NEVR
(`nginx-1.24.0-1.el9`) and confirm it exists in the repo with
`dnf list --showduplicates nginx`.

### Service hardening

```yaml
- name: Deploy nginx configuration
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: nginx
    mode: '0640'
    validate: '/usr/sbin/nginx -t -c %s'
    backup: true
  become: true
  notify: Restart nginx

- name: Ensure nginx is enabled and running
  ansible.builtin.systemd:
    name: nginx
    state: started
    enabled: true
  become: true

- name: Open HTTPS in the firewall
  ansible.posix.firewalld:
    port: 443/tcp
    permanent: true
    immediate: true
    state: enabled
    zone: "{{ firewall_zone | default('public') }}"
  become: true

- name: Allow nginx outbound network connections
  ansible.posix.seboolean:
    name: httpd_can_network_connect
    state: true
    persistent: true
  become: true
```

### SELinux

```yaml
- name: Ensure SELinux is enforcing
  ansible.posix.selinux:
    policy: targeted
    state: enforcing
  become: true

- name: Set the SELinux file context for the application directory
  community.general.sefcontext:
    target: '/opt/myapp(/.*)?'
    setype: httpd_sys_content_t
    state: present
  become: true
  notify: Restore SELinux contexts

# handlers/main.yml
- name: Restore SELinux contexts
  ansible.builtin.command:
    cmd: restorecon -Rv /opt/myapp
  become: true
  changed_when: true
```

`sefcontext` only writes the policy rule. Without the `restorecon` handler the
files on disk keep their old labels.

### Certificates

```yaml
- name: Deploy the TLS certificate
  ansible.builtin.copy:
    content: "{{ vault_tls_certificate }}"
    dest: "/etc/pki/tls/certs/{{ service_name }}.crt"
    owner: root
    group: "{{ service_name }}"
    mode: '0644'
  become: true
  notify: Restart managed service

- name: Deploy the TLS private key
  ansible.builtin.copy:
    content: "{{ vault_tls_private_key }}"
    dest: "/etc/pki/tls/private/{{ service_name }}.key"
    owner: root
    group: "{{ service_name }}"
    mode: '0440'
  become: true
  no_log: true
  notify: Restart managed service

- name: Verify the certificate and key match
  ansible.builtin.shell:
    cmd: |
      set -o pipefail
      cert=$(openssl x509 -noout -pubkey -in "/etc/pki/tls/certs/{{ service_name }}.crt")
      key=$(openssl pkey -pubout -in "/etc/pki/tls/private/{{ service_name }}.key")
      [ "$cert" = "$key" ]
    executable: /bin/bash
  become: true
  changed_when: false
```

Compare public keys with `openssl pkey`, not moduli with `openssl rsa` —
`rsa` fails outright on ECDSA and Ed25519 keys.

### Service accounts and operator keys

```yaml
- name: Create the application service account
  ansible.builtin.user:
    name: "{{ app_user }}"
    comment: "{{ app_name }} service account — managed by Ansible"
    system: true
    shell: /sbin/nologin
    home: "/opt/{{ app_name }}"
    create_home: true
  become: true

- name: Deploy authorised SSH keys for operators
  ansible.posix.authorized_key:
    user: "{{ item.username }}"
    key: "{{ item.public_key }}"
    state: present
    exclusive: true       # removes any key not managed here
    comment: "{{ item.username }}@{{ item.team }}"
  loop: "{{ authorized_operators }}"
  loop_control:
    label: "{{ item.username }}"
  become: true
```

`exclusive: true` deletes unmanaged keys. Confirm the inventory list is
complete before the first run, or you will lock out live operators.

## Inventory

### Static, segmented by zone

```yaml
---
all:
  children:
    dmz:
      children:
        load_balancers:
          hosts:
            lb-01.dmz.prod.example.com:
            lb-02.dmz.prod.example.com:
    internal:
      children:
        webservers:
          hosts:
            web-01.internal.prod.example.com:
            web-02.internal.prod.example.com:
    pci_zone:
      children:
        payment_processors:
          hosts:
            pay-01.pci.prod.example.com:
```

### Dynamic (AWS)

```yaml
# inventories/production/aws_ec2.yml
---
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  tag:Environment: production
  instance-state-name: running
keyed_groups:
  - key: tags.ComplianceZone
    prefix: zone
    separator: "_"
  - key: tags.Role
    prefix: role
    separator: "_"
hostnames:
  - tag:Name
  - private-dns-name
compose:
  ansible_host: private_ip_address
  ansible_user: "'ec2-user'"     # literal string needs the inner quotes
```

Verify with `ansible-inventory -i inventories/production/aws_ec2.yml --graph`
before pointing a playbook at it.
