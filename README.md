# Ansible Role: Mattermost

Installs and configures [Mattermost](https://mattermost.com/) from either the official
Mattermost APT repository or a release tarball.

## Features

- **Two install methods** (`mattermost_install_method`):
  - `apt` — official Mattermost repository (`deb.packages.mattermost.com`) to `/opt/mattermost`,
    using the package's systemd unit (community edition).
  - `tarball` — official release from `releases.mattermost.com`, choosing the **team** or
    **enterprise** edition, installed to a versioned dir symlinked at `/opt/mattermost` with a
    role-managed systemd unit and config/data kept outside the versioned dir for clean upgrades.
- **Database**: install a **local PostgreSQL** or connect to an **external** managed PostgreSQL.
- **nginx reverse proxy** with your choice of TLS:
  - Let's Encrypt (certbot, webroot),
  - Cloudflare Origin certificate, or
  - none (plain HTTP, e.g. TLS terminated upstream).
- **Optional Cloudflare API integration**: manage the DNS record and the zone SSL mode.
- **GitLab OAuth SSO** (works in Team Edition). The OAuth endpoint URLs are configurable, so a
  generic OIDC provider (e.g. Keycloak) can be wired in through the GitLab connector.
- All settings deep-merge into `config.json`, so any Mattermost option can be overridden.

> **Note on OIDC/SAML:** Native generic OIDC and SAML are Mattermost **Enterprise** features. On
> Team Edition, the GitLab OAuth connector is the supported free SSO path; point its endpoints at an
> OIDC provider that exposes GitLab-style OAuth endpoints to approximate OIDC.

## Requirements

- Debian 11/12 or Ubuntu 20.04/22.04/24.04.
- Ansible 2.14+ and the collections in `requirements.yml`:
  `ansible-galaxy collection install -r requirements.yml`

## Role variables

See [`defaults/main.yml`](defaults/main.yml) for the full, documented list. The essentials:

| Variable | Default | Description |
| --- | --- | --- |
| `mattermost_install_method` | `apt` | `apt` (community, from the repo) or `tarball` (team/enterprise, from a release). |
| `mattermost_edition` | `community` | `community` for `apt`; `team` or `enterprise` for `tarball`. |
| `mattermost_version` | `""` | apt: version to pin (e.g. `11.8.2`, `-0` appended), empty = latest. tarball: **required** release (e.g. `9.11.9`). |
| `mattermost_checksum` | `""` | Optional `sha256:...` checksum for the tarball download. |
| `mattermost_site_url` | `""` | **Required.** Public URL, e.g. `https://chat.example.com`. |
| `mattermost_db_type` | `local` | `local` (role installs PostgreSQL) or `external`. |
| `mattermost_db_password` | `""` | **Required.** Store in Vault. |
| `mattermost_db_host` / `_port` / `_sslmode` | `127.0.0.1` / `5432` / `require` | Used for `external`. |
| `mattermost_nginx_enabled` | `true` | Deploy the nginx vhost. |
| `mattermost_tls_mode` | `letsencrypt` | `letsencrypt`, `cloudflare_origin`, or `none`. |
| `mattermost_letsencrypt_email` | `""` | Required for `letsencrypt`. |
| `mattermost_cloudflare_origin_cert` / `_key` | `""` | PEM content for `cloudflare_origin`. |
| `mattermost_cloudflare_enabled` | `false` | Enable Cloudflare API management. |
| `mattermost_gitlab_auth_enabled` | `false` | Enable GitLab/OIDC SSO. |
| `mattermost_config_extra` | `{}` | Deep-merged overrides for `config.json`. |

Secrets (`mattermost_db_password`, `mattermost_gitlab_client_secret`,
`mattermost_cloudflare_api_token`, Origin key) should live in **Ansible Vault**.

## Example playbooks

### Local database + Let's Encrypt

```yaml
- hosts: chat
  become: true
  roles:
    - role: mattermost
      vars:
        mattermost_site_url: "https://chat.example.com"
        mattermost_db_type: local
        mattermost_db_password: "{{ vault_mm_db_password }}"
        mattermost_tls_mode: letsencrypt
        mattermost_letsencrypt_email: admin@example.com
```

### External database + Cloudflare Origin cert + Cloudflare DNS/SSL + OIDC (Keycloak via GitLab connector)

```yaml
- hosts: chat
  become: true
  roles:
    - role: mattermost
      vars:
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

### Tarball install of a specific Enterprise Edition release

```yaml
- hosts: chat
  become: true
  roles:
    - role: mattermost
      vars:
        mattermost_install_method: tarball
        mattermost_edition: enterprise      # or "team"
        mattermost_version: "9.11.9"        # required for the tarball method
        mattermost_checksum: "sha256:..."   # optional but recommended
        mattermost_site_url: "https://chat.example.com"
        mattermost_db_type: local
        mattermost_db_password: "{{ vault_mm_db_password }}"
        mattermost_tls_mode: letsencrypt
        mattermost_letsencrypt_email: admin@example.com
```

## Upgrades

- **apt method**: set `mattermost_version` to a newer version (or leave it empty and run
  `apt upgrade mattermost`) and re-run. The package upgrades in place. Config
  (`/opt/mattermost/config/config.json`) and data (`/opt/mattermost/data`) persist.
- **tarball method**: bump `mattermost_version` and re-run. The new release is extracted to
  `/opt/mattermost-<version>` and the `/opt/mattermost` symlink is repointed; config
  (`/etc/mattermost`) and data (`/var/opt/mattermost`) live outside the versioned dir and are
  untouched.

## Testing

```bash
pip install "molecule-plugins[docker]" molecule ansible-core ansible-lint yamllint
ansible-galaxy collection install -r requirements.yml
yamllint . && ansible-lint
molecule test
```

## License

MIT
