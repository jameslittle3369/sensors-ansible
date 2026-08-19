# sensors-ansible

Ansible deployment for the LilLittleSensors Raspberry Pi fleet:

- **rpi4-db** — PostgreSQL server (implemented)
- **rpi4-api** — nginx + FastAPI via uv-managed Python (placeholder role only)
- **rpi4-app** — Grafana + uv-managed Python + standalone systemd services/timers (placeholder role only)

## Connectivity model

- SSH as **root**, password auth (no SSH keys).
- **No sudo anywhere.** The one exception: the PostgreSQL role/db/extension
  tasks in `roles/postgres/tasks/roles_db.yml` use Ansible `become` with
  `become_method: su` (not sudo) to run as the OS `postgres` user, since
  `community.postgresql.*` modules need peer auth over the local socket.
  Root can `su` to any user without a password, so this needs no sudo
  package or config.
- Secrets (root SSH password, Postgres `api` user password) are stored in
  an `ansible-vault`-encrypted file, never in plaintext.
- The control node (this Mac) connects via Ansible's `paramiko` plugin so
  password auth works without needing the `sshpass` CLI installed.

## Setup

```bash
# 1. Install the required collection
ansible-galaxy collection install -r requirements.yml

# 2. Create the vault file (see vault.yml.example for the keys it needs)
ansible-vault create inventory/group_vars/all/vault.yml
#   vault_root_ssh_password: "<actual root SSH password>"
#   vault_postgres_api_password: "<actual password for the Postgres 'api' role>"
```

`inventory/hosts.yml` already has `rpi4-db` pinned to its static IP
(`10.0.0.11`). `api`/`app` groups exist with no hosts yet -- add entries
there once rpi4-api/rpi4-app are ready.

## Running

```bash
# rpi4-db only
ansible-playbook site.yml --limit db --ask-vault-pass

# rpi4-db + rpi4-api once rpi4-api has an inventory entry and role content
ansible-playbook site.yml --limit db,api --ask-vault-pass

# everything currently defined
ansible-playbook site.yml --ask-vault-pass
```

## What the rpi4-db run does

- **common role**: installs and enables `fail2ban` (with an `sshd` jail
  enabled -- Debian ships none enabled by default) and `chrony` as the NTP
  client (default public pool).
- **postgres role**:
  - Adds the official PGDG apt repository for the host's Debian release
    (`trixie-pgdg` on Debian 13) using the current deb822 `.sources` format.
  - Installs PostgreSQL (version pinned via `postgres_major_version`,
    currently 18) plus `postgresql-contrib` (needed for the `citext`
    extension) and `python3-psycopg2`.
  - Sets `listen_addresses` to `localhost` plus the host's own private IP
    on `10.0.0.0/24` (derived from facts, not `0.0.0.0`).
  - Configures `pg_hba.conf` to allow `scram-sha-256` connections from
    `10.0.0.0/24` (for rpi4-api/rpi4-app later) plus loopback.
  - Creates the `api` role (`LOGIN`, `SUPERUSER`, vaulted password) and the
    `api` database, idempotently -- it will not touch an existing database,
    so a `pg_restore` done afterwards is safe on repeat playbook runs.
  - Creates the `citext` extension inside the `api` database.

## Verification

On rpi4-db:

```bash
ss -tlnp | grep 5432                    # listening on loopback + 10.0.0.11 only
su postgres -c "psql -c '\du'"           # role "api" shows Superuser
su postgres -c "psql -c '\l'"             # database "api" owned by api
su postgres -c "psql -d api -c '\dx'"      # extension "citext" present
apt-cache policy postgresql-18            # origin should be apt.postgresql.org
systemctl status postgresql fail2ban chrony
fail2ban-client status sshd
chronyc tracking
```

From another host on `10.0.0.0/24`:

```bash
psql -h 10.0.0.11 -U api -d api -W
```
