# Connect Local PostgreSQL to Docker Container

This guide explains how to expose your **local** PostgreSQL installation (running on your host machine's Linux OS) to a **Docker container**.

> **Note**: This guide uses Port `5435` as per your configuration. Default PostgreSQL port is `5432`.

---

## ✅ Step 1: Confirm PostgreSQL Version

First, check which version of PostgreSQL you are running to edit the correct configuration files.

```bash
psql --version
# Output Example: psql (PostgreSQL) 16.1
```

*Replace `16` in the paths below with your actual version number (e.g., `14`, `15`, `17`).*

---

## ✅ Step 2: Configure PostgreSQL to Listen on All Addresses

By default, PostgreSQL only listens on `localhost`. We need to allow it to listen on the network interface that Docker talks to.

1. Edit `postgresql.conf`:
   ```bash
   sudo nano /etc/postgresql/16/main/postgresql.conf
   ```

2. Find and change `listen_addresses` and `port`:
   ```properties
   # FROM:
   #listen_addresses = 'localhost'
   
   # TO (Uncomment and change to '*'):
   listen_addresses = '*'
   
   # Ensure port is set to your desired port (default is 5432, you usually use 5435):
   port = 5435
   ```

---

## ✅ Step 3: Configure Authentication (pg_hba.conf)

Now we must tell PostgreSQL to accept incoming connections from the Docker network.

1. Edit `pg_hba.conf`:
   ```bash
   sudo nano /etc/postgresql/16/main/pg_hba.conf
   ```

2. Add the following lines to the end of the file.
   * `172.17.0.0/16` covers the default Docker bridge network.
   * `0.0.0.0/0` covers ALL networks (use with caution, ensure firewall is active).

   ```properties
   # TYPE  DATABASE        USER            ADDRESS                 METHOD
   host    all             all             172.17.0.0/16           scram-sha-256
   host    all             all             0.0.0.0/0               scram-sha-256
   ```
   
   > **Note**: PostgreSQL 14+ uses `scram-sha-256` by default. If you configured it to use `md5`, use `md5` instead. If you face "password authentication failed", check if your user's password is encrypted with md5 or scram-sha-256.

---

## ✅ Step 4: Restart & Verify

Apply the changes by restarting the service.

```bash
# heavy reload to apply config changes
sudo systemctl restart postgresql
```

**Verify it is listening on `0.0.0.0` or `*`:**

```bash
sudo ss -lntp | grep postgres
```

**Expected Output:**
```text
LISTEN 0      244          0.0.0.0:5435      0.0.0.0:*    users:(("postgres",pid=...,fd=...))
```
*If you see `127.0.0.1:5435`, it is acting incorrect. It MUST say `0.0.0.0` or `*`.*

---

## ✅ Step 5: Check Firewall (UFW)

If you have a firewall enabled, the Docker container might still be blocked.

```bash
# Check status
sudo ufw status

# If active, allow the port
sudo ufw allow 5435/tcp
```

---

## ✅ Step 6: Docker Compose Configuration

To connect from inside the container, use the special DNS name `host.docker.internal`.

**Crucial for Linux:** You must explicitly add the `extra_hosts` mapping.

```yaml
version: "3.9"

services:
  backend-user-app:
    build: ./backend-user-app
    container_name: backend-user-app
    ports:
      - "8001:8001"
    environment:
      # Use host.docker.internal and your custom port 5435
      DATABASE_URL: postgresql+asyncpg://postgres:Zaigo%4025@host.docker.internal:5435/fieldy-backend-user-app
    
    # REQUIRED for Linux Docker to resolve 'host.docker.internal'
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

## Summary Checklist
1. [ ] `listen_addresses = '*'` in `postgresql.conf`
2. [ ] `port = 5435` in `postgresql.conf`
3. [ ] `host ... 0.0.0.0/0 ...` in `pg_hba.conf`
4. [ ] `sudo systemctl restart postgresql`
5. [ ] `extra_hosts` added to `docker-compose.yml`
