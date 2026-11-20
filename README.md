# Ansible Nginx Deployment

**A role-based Ansible project for repeatable Nginx installation and configuration on Ubuntu.**

This repository demonstrates how to deploy a small web server using declarative infrastructure automation rather than manual package installation and configuration edits. The playbook selects a target group, applies an Nginx role, renders configuration and content from Jinja2 templates, and validates configuration changes before reloading the service.

The scope is deliberately focused: one Nginx site serving a configurable static welcome page over HTTP. It is a practical automation exercise, not a claim of a production-hardened web platform.

## Engineering Highlights

- **Reusable role:** Nginx implementation is separated from the entry-point playbook.
- **Configurable deployment:** Port, server name, site name, document root, and welcome message are role variables.
- **Change-aware operations:** Handlers validate and reload Nginx only when relevant configuration tasks report changes.
- **Idempotent design:** Ansible's package, file, template, and service modules converge the host toward a declared state.
- **Secret separation:** SSH keys remain on the control node; a local Ansible Vault file can supply the sudo password without embedding it in the playbook.

## Deployment Architecture

```text
Ansible control node
  ├── inventory/hosts              selects the nginx target group and SSH settings
  ├── site.yml                     applies the nginx role with become: true
  └── inventory/group_vars/nginx/  optional local Vault-managed group secrets
                  |
                  | SSH + privilege escalation
                  v
Ubuntu managed host
  ├── APT: nginx package
  ├── /etc/nginx/sites-available/nginx-site   (rendered configuration)
  ├── /etc/nginx/sites-enabled/nginx-site     (symlink to configuration)
  ├── /var/www/html/index.html               (rendered welcome page)
  └── nginx service                          (enabled and running)
```

## Repository Layout

```text
ansible-nginx-lab/
├── .gitignore
├── README.md
├── site.yml
├── inventory/
│   ├── hosts
│   └── group_vars/
│       └── nginx/
│           └── vault.yml              # LOCAL ONLY — ignored by Git
└── roles/
    └── nginx/
        ├── defaults/main.yml
        ├── tasks/main.yml
        ├── handlers/main.yml
        └── templates/
            ├── site.conf.j2
            └── index.html.j2
```

The tree illustrates the deployment layout, including the **local-only** Vault file. `vault.yml` must not appear in the published GitHub repository. A future `evidence/` directory may contain redacted command outputs and screenshots; it should be added to the public tree only after those files actually exist.

## File-by-File Implementation

### `site.yml` — Entry-point playbook

Selects hosts in the `nginx` inventory group, enables privilege escalation with `become: true`, and applies the `nginx` role. Keeping orchestration separate from implementation makes the role easier to maintain and reuse.

### `inventory/hosts` — Target selection and connectivity

Defines the managed host or hosts in the `[nginx]` group. The current lab inventory also provides:

| Setting | Responsibility |
| --- | --- |
| `ansible_user` | SSH account used by Ansible on the managed host. |
| `ansible_ssh_private_key_file` | Path to a private key **on the control node**; the key itself is not part of this repository. |
| `ansible_become` | Enables privileged execution; the playbook also sets `become: true`. |

The working inventory contains lab-specific addresses, an account name, and an absolute SSH-key path. Replace these with non-sensitive example values before publishing a public repository. An illustrative inventory is:

```ini
[nginx]
web01 ansible_host=192.0.2.10

[nginx:vars]
ansible_user=automation
ansible_ssh_private_key_file=/home/automation/.ssh/ansible_ed25519
ansible_become=true
```

`192.0.2.10` is a documentation-only example, **not** the deployment host. The managed account must have the necessary sudo permissions.

### `inventory/group_vars/nginx/vault.yml` — Local encrypted credentials

For the current Vault-based workflow, this location is intended to contain the group-scoped variable `ansible_become_password`. Ansible can automatically load group variables from `inventory/group_vars/nginx/` when the corresponding inventory is used. `--ask-vault-pass` supplies the **decryption** password; the decrypted `ansible_become_password` supplies the **sudo** password. These are distinct credentials.

Keep the real Vault file on the control node and exclude it from the public repository using `.gitignore`. To create a Vault file for a new environment:

```bash
mkdir -p inventory/group_vars/nginx
ansible-vault create inventory/group_vars/nginx/vault.yml
```

Enter the variable name and the environment's sudo password **inside the encrypted Vault editor**:

```yaml
ansible_become_password: "YOUR_ENVIRONMENT_SUDO_PASSWORD"
```

This YAML is an illustrative variable name and placeholder, **not** a credential to commit. Do not create a plaintext `vault.example.yml` inside the automatically loaded `group_vars/nginx/` directory: Ansible could load its placeholder as an active variable. If a shareable sample is added later, place it outside that directory (for example, `examples/vault.example.yml`) and ensure no real secrets are present.

> Path note: the Vault location above follows the intended project layout; confirm the actual local path before publishing documentation as a final deployment record.

### `roles/nginx/defaults/main.yml` — Configurable role defaults

| Variable | Value in supplied role | Used for |
| --- | --- | --- |
| `nginx_port` | `80` | Nginx `listen` port. |
| `nginx_welcome_message` | `welcome to nginx on Ubuntu` | Welcome page heading. |
| `nginx_server_name` | Lab-specific IP address | Nginx `server_name` directive; substitute a host-specific value. |
| `nginx_site_name` | `nginx-site` | Filename under `sites-available` and symlink under `sites-enabled`. |
| `nginx_web_root` | `/var/www/html` | Nginx document root and location of `index.html`. |

Role defaults have low precedence and can be overridden by inventory/group variables or other higher-precedence inputs without modifying the templates. Note that the supplied `nginx_server_name` value is environment-specific and should be sanitized in the published role defaults.

### `roles/nginx/tasks/main.yml` — Deployment sequence

| Task | Module | Exact behavior |
| --- | --- | --- |
| Install nginx package | `ansible.builtin.apt` | Ensures `nginx` is present; updates the APT cache when needed (`cache_valid_time: 3600`). |
| Remove default nginx site | `ansible.builtin.file` | Removes `/etc/nginx/sites-enabled/default` if present; notifies `nginx_config_changed`. |
| Deploy configuration | `ansible.builtin.template` | Renders `site.conf.j2` to `/etc/nginx/sites-available/{{ nginx_site_name }}` with root ownership and `0644` mode; notifies the handler. |
| Enable Nginx site | `ansible.builtin.file` | Ensures a symlink exists in `/etc/nginx/sites-enabled/` pointing to the managed config; notifies the handler. |
| Deploy welcome | `ansible.builtin.template` | Renders `index.html.j2` to `{{ nginx_web_root }}/index.html`, owned by root with `0644` mode. |
| Ensure nginx service is running | `ansible.builtin.service` | Ensures the Nginx service is started and enabled at boot. |

The HTML task does **not** notify the Nginx reload handlers: changing a static page does not require reloading the web-server configuration.

### `roles/nginx/templates/site.conf.j2` — Nginx server block

Renders `listen` from `nginx_port`, `server_name` from `nginx_server_name`, and `root` from `nginx_web_root`. The server serves `index.html` as an index document, while `try_files $uri $uri/ =404` returns 404 when a requested path does not match a file or directory.

### `roles/nginx/templates/index.html.j2` — Static content

Creates a minimal HTML page with an `<h1>` element populated from `nginx_welcome_message`. Changing the variable changes the rendered page without editing HTML source.

### `roles/nginx/handlers/main.yml` — Validation and reload

Both handlers listen for `nginx_config_changed` and run in their declaration order:

1. **Validate:** `/usr/sbin/nginx -t` via `ansible.builtin.command`. `changed_when: false` prevents a read-only validation from being reported as a host change.
2. **Reload:** `ansible.builtin.service` with `state: reloaded` runs only if validation and the preceding work succeed.

Notifications are coalesced: multiple notifying tasks in a play do not cause repeated handler runs for every notification. **Important limitation:** the configuration is written before this handler validates it; if validation fails, the play stops before reload, but the file changes are **not automatically rolled back**.

## Requirements

- **Control node:** Linux with Ansible installed, network connectivity to the target, and access to the configured SSH private key.
- **Managed node:** Ubuntu with SSH access, Python for Ansible modules, working APT repositories, sudo privileges for the automation account, and the usual Debian/Ubuntu Nginx directory layout.
- **Secrets:** A locally managed encrypted Vault variable (`ansible_become_password`) if password-based sudo is used. Other supported sudo credential workflows may be substituted.

The role currently assumes `/var/www/html` exists. If `nginx_web_root` is changed, create that directory before deploying: the supplied tasks do not create arbitrary document roots.

## Running the Playbook

Run commands from the repository root after adapting the inventory and role settings to your environment:

```bash
# Check SSH connectivity
ansible -i inventory/hosts nginx -m ansible.builtin.ping --ask-vault-pass

# Validate Ansible syntax
ansible-playbook -i inventory/hosts site.yml --syntax-check --ask-vault-pass

# Deploy Nginx
ansible-playbook -i inventory/hosts site.yml --ask-vault-pass
```

In the documented Vault workflow, the password prompt decrypts the local Vault file; Ansible then reads `ansible_become_password` for privileged tasks. If the file is not used, adjust the authentication flags to the actual environment rather than supplying `--ask-vault-pass` unnecessarily.

## Verification

After deployment, check syntax and HTTP behavior:

```bash
# Validate the Nginx configuration on the target
ansible -i inventory/hosts nginx --become --ask-vault-pass \
  -m ansible.builtin.command -a "/usr/sbin/nginx -t"

# Replace the documentation IP with the reachable address of your host
curl -i http://192.0.2.10/

# Check style and Ansible best practices
ansible-lint site.yml

# Rerun unchanged inputs to assess idempotency
ansible-playbook -i inventory/hosts site.yml --ask-vault-pass
```

Successful Nginx validation prints `syntax is ok` and `test is successful`; the HTTP test should return a successful response with the configured `<h1>` text. For an unchanged target, the second playbook run should report `changed=0`. If published, report exact test outcomes only alongside real, redacted output captured from the target environment. The project history includes successful playbook and lint runs, but supporting evidence files have not been verified as part of this README revision.

## GitHub Publication and Secret Handling

The project's `.gitignore` should exclude the local Vault path as well as any local Vault password files and Ansible retry artifacts, such as:

```gitignore
/inventory/group_vars/nginx/vault.yml
.vault_pass
.vault_password
*.retry
```

A `.gitignore` rule does not remove content that Git already tracks and does not protect a key intentionally copied into the repository. Keep SSH private keys outside the project directory, keep real passwords out of public files, and review the staged changes before committing. In particular, review `inventory/hosts` and `roles/nginx/defaults/main.yml` for local IP addresses, account names, and key paths.

Check ignore rules and staged content with:

```bash
git check-ignore -v inventory/group_vars/nginx/vault.yml
git status --short --untracked-files=all
git diff --cached --name-only
```

`git diff --cached --name-only` is useful **after staging** files; before staging, its output may be empty. `git status` alone does not reveal secrets inside tracked files, so inspect file contents before pushing.

## Scope and Limitations

This is a single-site HTTP deployment. It does not configure TLS/HTTPS, firewall rules, DNS, monitoring, automatic rollback, or multiple separate virtual hosts. The role also removes the default site symlink and overwrites the configured `index.html`, so review its effects before using it on a shared or existing production web server. A further hardening step would be validating a candidate configuration before replacing the live file.

## Summary

This project illustrates practical configuration management with Ansible: clear separation of inventory and role logic, reusable variables, Jinja2 rendering, change-triggered handlers, secret isolation, and repeatable host configuration. It is intended to be understandable, reproducible, and reviewable by engineers assessing DevOps fundamentals.
