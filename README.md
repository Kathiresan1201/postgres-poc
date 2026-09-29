# PostgreSQL Ansible Role for Ubuntu

To offer this as a Morpheus self-service catalog item (vSphere VM + PostgreSQL in one order), see [morpheus/README.md](morpheus/README.md).

## Preparation
1. Edit `inventory.ini` with the Ubuntu server IP and SSH user.
2. Edit `group_vars/postgresql_servers/main.yml` with the allowed application subnet, database name, user, PostgreSQL version, and sizing.
3. Change passwords in `group_vars/postgresql_servers/vault.yml`.
4. Encrypt the vault file:

```bash
ansible-vault encrypt group_vars/postgresql_servers/vault.yml
```

## Install requirements

```bash
ansible-galaxy collection install -r requirements.yml
```

## Validate

```bash
ansible all -m ping
ansible-playbook site.yml --syntax-check --ask-vault-pass
ansible-playbook site.yml --check --diff --ask-vault-pass
```

## Deploy

```bash
ansible-playbook site.yml --ask-vault-pass
```

If sudo needs a password:

```bash
ansible-playbook site.yml --ask-vault-pass --ask-become-pass
```

## Verification

```bash
sudo systemctl status postgresql --no-pager
sudo pg_lsclusters
sudo -u postgres psql -c "SELECT version();"
sudo -u postgres psql -c "\\l"
sudo -u postgres psql -c "\\du"
sudo ss -lntp | grep 5432
```

Important: If `postgresql_enable_ufw` is enabled, allow SSH before enabling UFW.
