# semaphore_ui

Installs [Semaphore UI](https://semaphoreui.com/) on RHEL, AlmaLinux and Rocky
Linux 9 and 10.

The upstream RPM contains nothing but `/usr/bin/semaphore` — no user, no unit
file, no configuration — so the role adds those. It installs the package with a
minimal configuration, creates the service account and the systemd unit,
imports your own `config.json` when you provide one, and starts the service.

## Role variables

| Variable | Default | Description |
| --- | --- | --- |
| `semaphore_version` | `2.19.12` | Drives the download URL and the upgrade check |
| `semaphore_rpm_url` | GitHub release URL | Override for an internal mirror |
| `semaphore_user` / `semaphore_group` | `semaphore` | Service account |
| `semaphore_home` | `/var/lib/semaphore` | Working directory of the service |
| `semaphore_confdir` | `/etc/semaphore` | |
| `semaphore_config_file` | `/etc/semaphore/config.json` | |
| `semaphore_config_src` | `""` | Your own `config.json`; see below |
| `semaphore_cookie_hash` | *undefined* | Generated when unset; see below |
| `semaphore_cookie_encryption` | *undefined* | Generated when unset; see below |
| `semaphore_access_key_encryption` | *undefined* | Generated when unset; see below |

## Configuration

By default the role writes a minimal `config.json` with a SQLite database:

```json
{
  "sqlite": {
    "host": "./database.sqlite"
  },
  "dialect": "sqlite",
  "cookie_hash": "...",
  "cookie_encryption": "...",
  "access_key_encryption": "..."
}
```

The relative path resolves against the unit's `WorkingDirectory`, so the
database ends up in `/var/lib/semaphore`.

Anything beyond that — PostgreSQL or MySQL, a different port, LDAP, OIDC, TLS,
mail, runners — goes in your own file, which you point `semaphore_config_src`
at. It is then copied verbatim and the default configuration is not written.

## Secrets

Semaphore signs session cookies and encrypts stored credentials with three
keys, each 32 random bytes, base64-encoded:

```bash
openssl rand -base64 32
```

You can supply them yourself, which is what you want when several nodes share
one database, or when the keys belong in your vault alongside everything else:

```yaml
- role: semaphore_ui
  vars:
    semaphore_cookie_hash: "{{ vault_semaphore_cookie_hash }}"
    semaphore_cookie_encryption: "{{ vault_semaphore_cookie_encryption }}"
    semaphore_access_key_encryption: "{{ vault_semaphore_access_key_encryption }}"
```

Leave any of them undefined and the role generates one on the first run. It is
then kept: before writing the config the role reads the existing
`config.json` back and reuses whatever keys are already in it, so a rerun
changes nothing. There is no second file holding the same secrets.

Precedence per key, highest first:

1. the variable, when you set it
2. the value already in `config.json` on the host
3. a freshly generated one

So you can hand over a single key and let the role keep the other two. When
`semaphore_config_src` is used nothing is generated at all — your file carries
its own keys.

**Never rotate `access_key_encryption` on a live instance.** Every stored
credential is encrypted with it and becomes unreadable if it changes.

## Creating the first admin account

The role does not create any user. Semaphore starts with an empty database, so
run this once on the host after the first deploy:

```bash
sudo -u semaphore semaphore user add --admin \
  --config /etc/semaphore/config.json \
  --login admin \
  --name Administrator \
  --email admin@example.com \
  --password 'choose-a-password'
```

`semaphore user list` shows the existing accounts, and `semaphore user change`
updates one.

## Example playbook

```yaml
- name: Deploy Semaphore UI
  hosts: semaphore
  become: true
  roles:
    - role: semaphore_ui
```

With your own configuration:

```yaml
- role: semaphore_ui
  vars:
    semaphore_config_src: files/semaphore-config.json
```

## Notes

- The RPM is signed with Semaphore's own key, which is not in the RHEL keyring,
  so GPG checking is disabled for the install.
- The role does not set up a reverse proxy or TLS.
