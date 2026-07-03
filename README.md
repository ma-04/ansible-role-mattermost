# Ansible Role: Mattermost

Installs and configures [Mattermost](https://mattermost.com/) on Debian/Ubuntu — from
either the official APT repository or a release tarball — with PostgreSQL (local or
external), an nginx reverse proxy with TLS, optional Cloudflare API management, and
optional GitLab/OIDC SSO.

## Table of contents

- [How it works](#how-it-works)
- [Requirements](#requirements)
- [Quick start](#quick-start)
- [The `deploy/` harness](#the-deploy-harness)
- [Defaults and how to change them](#defaults-and-how-to-change-them)
- [Configuration recipes](#configuration-recipes)
- [Upgrades](#upgrades)
- [Secrets](#secrets)
- [Testing](#testing)
- [License](#license)

## How it works

The role runs, in order: preflight assertions → install → database → render
`config.json` → systemd unit → nginx (if enabled) → Cloudflare (if enabled) →
firewall (if enabled). Each stage is gated by a variable, so you enable only what you
need.

**Install methods** (`mattermost_install_method`):

| Method | Source | Editions | Layout |
| --- | --- | --- | --- |
| `tarball` (default) | `releases.mattermost.com` | `team` (free) or `enterprise` | Versioned dir symlinked at `/opt/mattermost`; config in `/etc/mattermost`, data in `/var/opt/mattermost` (kept outside the versioned dir for clean upgrades). Role-managed systemd unit. |
| `apt` | `deb.packages.mattermost.com` | enterprise server package | Package owns `/opt/mattermost` with config/data underneath; uses the package's systemd unit. |

**config.json** is built from a role base (site URL, Postgres DSN, file/log/plugin
paths, GitLab settings when SSO is on) deep-merged with your
`mattermost_config_extra`, so any Mattermost setting can be overridden without
templating the whole file.

## Requirements

- Debian 11/12 or Ubuntu 20.04/22.04/24.04, reachable over SSH with `become`.
- Ansible 2.14+ on the control machine, plus the collections in `requirements.yml`:
  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```

## Quick start

Minimal direct use of the role:

```yaml
- hosts: chat
  become: true
  roles:
    - role: ansible-role-mattermost
      vars:
        mattermost_site_url: "https://chat.example.com"
        mattermost_db_type: local
        mattermost_db_password: "{{ vault_mm_db_password }}"
        mattermost_tls_mode: letsencrypt
        mattermost_letsencrypt_email: admin@example.com
```

For a ready-made playbook with inventory/vars/vault scaffolding, use the `deploy/`
harness below.

## The `deploy/` harness

`deploy/` is a runnable playbook. Its `ansible.cfg` points `roles_path` at the repo
root, so the role resolves by directory name. Only the `*.example` templates are
committed; your real `inventory.ini`, `group_vars/chat/vars.yml`, and
`group_vars/chat/vault.yml` are git-ignored so environment values and secrets never
land in git.

```bash
cd deploy

# 1. Inventory — host IP + SSH details
cp inventory.ini.example inventory.ini && $EDITOR inventory.ini

# 2. Non-secret role config
cp group_vars/chat/vars.yml.example group_vars/chat/vars.yml && $EDITOR group_vars/chat/vars.yml

# 3. Secrets — fill in, then encrypt
cp group_vars/chat/vault.yml.example group_vars/chat/vault.yml && $EDITOR group_vars/chat/vault.yml
ansible-vault encrypt group_vars/chat/vault.yml

# 4. Deploy
ansible-galaxy collection install -r ../requirements.yml
ansible-playbook site.yml --ask-vault-pass
```

Verify on the host:

```bash
systemctl status mattermost
curl -s localhost:8065/api/v4/system/ping    # -> {"status":"OK"}
```

Then browse `https://<your-domain>` and create the first admin account.

## Defaults and how to change them

Every variable lives in [`defaults/main.yml`](defaults/main.yml) with inline notes.
Override any of them in your playbook `vars:`, in `group_vars`/`host_vars`, or with
`-e`. The essentials:

| Variable | Default | Notes |
| --- | --- | --- |
| `mattermost_install_method` | `tarball` | `tarball` (team/enterprise) or `apt` (enterprise package). |
| `mattermost_edition` | `team` | `team` (free) or `enterprise` for tarball; effectively enterprise for apt. |
| `mattermost_version` | `11.8.2` | tarball: **required** exact release. apt: version to pin (empty = latest). |
| `mattermost_checksum` | `""` | Optional `sha256:...` for the tarball download. |
| `mattermost_site_url` | `""` | **Required.** e.g. `https://chat.example.com`. |
| `mattermost_bind_address` / `_listen_port` | `127.0.0.1` / `8065` | App listen address (localhost behind the proxy). |
| `mattermost_max_file_size` | `50M` | Upload size proxied by nginx. |
| `mattermost_db_type` | `local` | `local` (role installs PostgreSQL) or `external`. |
| `mattermost_db_name` / `_user` | `mattermost` / `mattermost` | Database and role names. |
| `mattermost_db_password` | `""` | **Required.** Store in Vault. |
| `mattermost_db_host` / `_port` / `_sslmode` | `127.0.0.1` / `5432` / `require` | Used when `external` (local forces `sslmode=disable`). |
| `mattermost_nginx_enabled` | `true` | Deploy the nginx vhost. |
| `mattermost_tls_mode` | `letsencrypt` | `letsencrypt`, `cloudflare_origin`, or `none`. |
| `mattermost_letsencrypt_email` | `""` | Required for `letsencrypt`. |
| `mattermost_cloudflare_origin_cert` / `_key` | `""` | PEM content for `cloudflare_origin` (via Vault). |
| `mattermost_configure_firewall` | `false` | Open 80/443 with ufw (skip if a cloud firewall handles it). |
| `mattermost_cloudflare_enabled` | `false` | Manage DNS record + zone SSL mode via the Cloudflare API. |
| `mattermost_cloudflare_api_token` / `_zone` / `_record_content` / `_ssl_mode` | `""` / `""` / `""` / `full` | Cloudflare API settings. |
| `mattermost_gitlab_auth_enabled` | `false` | Enable GitLab OAuth / OIDC SSO. |
| `mattermost_gitlab_client_id` / `_client_secret` | `""` / `""` | OAuth app credentials (secret via Vault). |
| `mattermost_gitlab_auth_endpoint` / `_token_endpoint` / `_user_api_endpoint` | gitlab.com URLs | Point at any provider exposing GitLab-style OAuth endpoints. |
| `mattermost_config_extra` | `{}` | Deep-merged last over everything in `config.json`. |

> **OIDC/SAML note:** native generic OIDC and SAML are Mattermost **Enterprise**
> features. On Team Edition, the GitLab OAuth connector is the supported free SSO path
> — set the three endpoints above to an OIDC provider (e.g. Keycloak) that exposes
> GitLab-style OAuth URLs.

## Configuration recipes

**External database + Cloudflare Origin cert + Cloudflare DNS/SSL + OIDC via GitLab connector:**

```yaml
mattermost_site_url: "https://chat.example.com"

mattermost_db_type: external
mattermost_db_host: db.internal.example.com
mattermost_db_password: "{{ vault_mm_db_password }}"
mattermost_db_sslmode: require

mattermost_tls_mode: cloudflare_origin
mattermost_cloudflare_origin_cert: "{{ vault_mm_origin_cert }}"
mattermost_cloudflare_origin_key: "{{ vault_mm_origin_key }}"

mattermost_cloudflare_enabled: true
mattermost_cloudflare_api_token: "{{ vault_cf_token }}"
mattermost_cloudflare_zone: example.com
mattermost_cloudflare_record_content: "203.0.113.10"
mattermost_cloudflare_ssl_mode: full

mattermost_gitlab_auth_enabled: true
mattermost_gitlab_client_id: "mattermost"
mattermost_gitlab_client_secret: "{{ vault_oidc_secret }}"
mattermost_gitlab_auth_endpoint: "https://keycloak.example.com/realms/main/protocol/openid-connect/auth"
mattermost_gitlab_token_endpoint: "https://keycloak.example.com/realms/main/protocol/openid-connect/token"
mattermost_gitlab_user_api_endpoint: "https://keycloak.example.com/realms/main/protocol/openid-connect/userinfo"
```

**Tarball install of a specific Enterprise release:**

```yaml
mattermost_install_method: tarball
mattermost_edition: enterprise      # or "team"
mattermost_version: "9.11.9"        # required for tarball
mattermost_checksum: "sha256:..."   # optional but recommended
mattermost_site_url: "https://chat.example.com"
mattermost_db_type: local
mattermost_db_password: "{{ vault_mm_db_password }}"
mattermost_tls_mode: letsencrypt
mattermost_letsencrypt_email: admin@example.com
```

## Upgrades

- **apt:** set `mattermost_version` to a newer version (or leave it empty and
  `apt upgrade mattermost`) and re-run. Config and data under `/opt/mattermost` persist.
- **tarball:** bump `mattermost_version` and re-run. The new release extracts to
  `/opt/mattermost-<version>` and the `/opt/mattermost` symlink is repointed; config
  (`/etc/mattermost`) and data (`/var/opt/mattermost`) are untouched.
- **Switching apt → tarball** removes the APT package first (its data under
  `/opt/mattermost/data` is purged) — back up the database and file store first.

## Secrets

Keep `mattermost_db_password`, `mattermost_gitlab_client_secret`,
`mattermost_cloudflare_api_token`, and the Cloudflare Origin key in **Ansible Vault**.
Secret-bearing tasks run with `no_log: true`; tarball downloads are checksum-verified
when `mattermost_checksum` is set.

## Testing

```bash
pip install "molecule-plugins[docker]" molecule ansible-core ansible-lint yamllint
ansible-galaxy collection install -r requirements.yml
yamllint . && ansible-lint
molecule test
```

## License

MIT
