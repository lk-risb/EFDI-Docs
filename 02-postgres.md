# PostgreSQL (pgEdge) Installation Instructions

- **More info:** <https://docs.pgedge.com/enterprise/>
- **Package:** Minimal / Enterprise Postgres 18

> [!IMPORTANT]
> In this and all following steps, replace the example passwords with secure,
> randomly generated values and store them in your secrets manager / password
> vault. Do not use the literal values shown here.

## 1. Install Postgres

```bash
# 1. Configure prerequisites
sudo apt-get update
sudo apt-get install -y curl gnupg2 lsb-release

# 2. Add pgEdge repository
sudo curl -sSL https://apt.pgedge.com/repodeb/pgedge-release_latest_all.deb -o /tmp/pgedge-release.deb
sudo dpkg -i /tmp/pgedge-release.deb
sudo apt-get update

# 3. Install Postgres (Minimal)
sudo apt-get install -y pgedge-enterprise-postgres-18

# 4. Initialize and start
sudo pg_ctlcluster 18 main start
sudo systemctl enable --now postgresql
```

## 2. Create root/admin user and enable remote access

```bash
sudo -u postgres psql -c "CREATE ROLE efdi WITH LOGIN SUPERUSER PASSWORD '<generated-password>';"
```

Edit `/etc/postgresql/18/main/pg_hba.conf` and add:

> [!NOTE]
> Choose the CIDR network below according to where the clients / database
> consumers actually live (e.g. the subnet of the app servers or VPN range).
> Do not open it wider than needed.

```conf
host    all             efdi            192.168.10.0/24          scram-sha-256
```

Edit `/etc/postgresql/18/main/postgres.conf` and set:

```conf
listen_addresses = '0.0.0.0'
```

Restart:

```bash
sudo systemctl restart postgresql
```

## 3. NetBird databases

```bash
sudo -u postgres psql -c "CREATE ROLE netbird WITH LOGIN SUPERUSER PASSWORD '<generated-password>';"

sudo -u postgres psql -c "CREATE DATABASE netbird_store;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE netbird_store TO netbird;"

sudo -u postgres psql -c "CREATE DATABASE netbird_auth;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE netbird_auth TO netbird;"

sudo -u postgres psql -c "CREATE DATABASE netbird_activity;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE netbird_activity TO netbird;"
```

## 4. Filestore database

```bash
sudo -u postgres psql -c "CREATE ROLE filestore WITH LOGIN SUPERUSER PASSWORD '<generated-password>';"
sudo -u postgres psql -c "CREATE DATABASE filestore;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE filestore TO filestore;"
```

## 5. CA database (optional)

```bash
sudo -u postgres psql -c "CREATE ROLE stepca WITH LOGIN SUPERUSER PASSWORD '<generated-password>';"
sudo -u postgres psql -c "CREATE DATABASE stepca;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE stepca TO stepca;"
```

## 6. Administration tool (optional)

If you prefer a GUI over `psql`, you can use [pgAdmin](https://www.pgadmin.org/)
to administer the database. Install it on your workstation and connect with the
`efdi` superuser created in step 2 (the `pg_hba.conf` entry and
`listen_addresses` already allow remote access from the configured network).
