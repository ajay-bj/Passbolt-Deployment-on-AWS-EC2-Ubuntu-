# 🚀 Passbolt on AWS EC2 (Ubuntu) — Step‑by‑Step SOP

This README is a **clean, repeatable SOP** to deploy Passbolt on a **fresh Ubuntu EC2** using **Docker + Docker Compose**.  
Every command block contains inline `#` comments explaining **what** and **why**.

> **Assumptions**
> - You launched an Ubuntu 22.04+ EC2 instance.
> - Security Group allows **SSH (22)** and **HTTP (80)**. (HTTPS 443 is optional; see notes.)
> - Your EC2 public IP: **`13.204.79.56`** (replace if different).

---

## 0) Connect to EC2
```bash
# SSH into your instance (replace key path and user if needed)
ssh -i /path/to/key.pem ubuntu@13.204.79.56
```

---

## 1) System prep
```bash
sudo apt update -y    # Refresh package lists
sudo apt upgrade -y   # Apply security/bug fixes
```

---

## 2) Install Docker Engine (official repo)
```bash
sudo apt install -y ca-certificates curl gnupg lsb-release  # Basic deps for repo + TLS

# Add Docker’s GPG key (so APT can trust Docker packages)
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add the stable Docker repository for your Ubuntu release
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin  # Install Docker + Compose v2 plugin

# Enable + start Docker daemon so it persists across reboots
sudo systemctl enable docker
sudo systemctl start docker

# Quick sanity check
docker --version        # e.g., Docker version 27.x
docker compose version  # e.g., Docker Compose version v2.x
```

> **Why not `docker-compose` (v1)?**  
> Docker Compose v1 is deprecated. We use **`docker compose`** (with a space), provided by the Docker plugin.

---

## 3) Create project structure
```bash
mkdir -p ~/passbolt       # Folder to hold compose + volumes (named volumes live in Docker)
cd ~/passbolt
```

---

## 4) Create `docker-compose.yml`
> This is a **minimal working** compose file. We start with **HTTP (80)** only for simplicity.  
> You can enable HTTPS later with a domain + certificates.

```yaml
services:
  db:
    image: mariadb:10.3
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: passbolt_root   # Root password for MariaDB
      MYSQL_DATABASE: passbolt             # App database name
      MYSQL_USER: passbolt                 # App DB user
      MYSQL_PASSWORD: passbolt             # App DB user password
    volumes:
      - db_data:/var/lib/mysql             # Persist DB data across restarts

  passbolt:
    image: passbolt/passbolt:latest
    restart: always
    depends_on:
      - db                                 # Wait for DB to be reachable
    environment:
      # Passbolt needs to know how users will reach it (used in links/emails)
      APP_FULL_BASE_URL: "http://13.204.79.56"

      # Database connection settings (match the db service above)
      DATASOURCES_DEFAULT_HOST: db
      DATASOURCES_DEFAULT_USERNAME: passbolt
      DATASOURCES_DEFAULT_PASSWORD: passbolt
      DATASOURCES_DEFAULT_DATABASE: passbolt

      # (Optional SMTP for later — Gmail example)
      # EMAIL_DEFAULT_FROM: "yourgmail@gmail.com"
      # EMAIL_TRANSPORT_DEFAULT_HOST: "smtp.gmail.com"
      # EMAIL_TRANSPORT_DEFAULT_PORT: "587"
      # EMAIL_TRANSPORT_DEFAULT_USERNAME: "yourgmail@gmail.com"
      # EMAIL_TRANSPORT_DEFAULT_PASSWORD: "your-16-char-app-password"
      # EMAIL_TRANSPORT_DEFAULT_TLS: "true"   # Must be a string, not boolean

    ports:
      - "80:80"                             # Expose HTTP; open SG port 80
      # - "443:443"                         # Optional: expose HTTPS (needs valid certs inside container)
    volumes:
      - gpg_data:/var/www/passbolt/config/gpg   # Persist GPG keys
      - jwt_data:/var/www/passbolt/config/jwt   # Persist JWT keys

volumes:
  db_data:
  gpg_data:
  jwt_data:
```

> **Why this image combo?**  
> Passbolt runs on PHP + Nginx and stores data in MariaDB. Persisting `/var/lib/mysql`, `config/gpg`, and `config/jwt` keeps your instance usable across restarts.

---

## 5) Bring the stack up
```bash
# Start containers in the background
sudo docker compose up -d

# Watch logs to ensure both services initialize cleanly
sudo docker compose logs -f passbolt
```

> **If you see permissions errors for JWT/GPG** (rare on first run), fix them with:  
> ```bash
> sudo docker exec -it passbolt-passbolt-1 bash -lc 'chown -R www-data:www-data /var/www/passbolt/config/{jwt,gpg} && chmod -R 700 /var/www/passbolt/config/{jwt,gpg}'
> ```  
> then re-run the install step below.

---

## 6) Access the Web UI
Open your browser to: **`http://13.204.79.56`**  
You should see the Passbolt welcome/setup page.

---

## 7) Create the first admin user (no SMTP required)

### A) Understand `docker exec`
```bash
docker ps                              # Lists running containers; note the Passbolt container name
# Typically it looks like: passbolt-passbolt-1

# Anatomy of docker exec:
# docker exec      -> run a command in a running container
# -i               -> keep STDIN open (interactive)
# -t               -> allocate a TTY (nice interactive shell)
# -u www-data      -> run as the "www-data" user (Passbolt web user)
# <container>      -> container name from `docker ps`
# bash             -> start a bash shell (so you can run multiple commands)
```

### B) Exec into the Passbolt container
```bash
sudo docker exec -it -u www-data passbolt-passbolt-1 bash   # Enter container as www-data user
cd /var/www/passbolt                                        # Move to the app dir (contains bin/cake)
```

### C) Run the installer (guided)
```bash
./bin/cake passbolt install --force   # Generates keys, DB schema; prompts for initial setup
```
- This will guide you and/or output a **registration link**.

### D) (Alternative) Manually register admin user
```bash
./bin/cake passbolt register_user \
  -u admin@example.com \            # <-- email (use yours)
  -f Admin \                        # <-- first name
  -l User \                         # <-- last name
  -r admin                          # <-- admin role
```
- The command prints a **registration URL** — open it in your browser to finish setup, set password, and download the recovery kit.

---

## 8) Optional — Configure SMTP (Gmail example)

> Needed to send invites / recovery emails. Use a **Gmail App Password** (not your login password).

1) Generate an App Password (Google Account → Security → 2‑Step Verification → App Passwords).  
2) Edit `docker-compose.yml` and **uncomment** the SMTP lines under `passbolt.environment`, then set values:
```yaml
      EMAIL_DEFAULT_FROM: "yourgmail@gmail.com"
      EMAIL_TRANSPORT_DEFAULT_HOST: "smtp.gmail.com"
      EMAIL_TRANSPORT_DEFAULT_PORT: "587"
      EMAIL_TRANSPORT_DEFAULT_USERNAME: "yourgmail@gmail.com"
      EMAIL_TRANSPORT_DEFAULT_PASSWORD: "bggkzenqkfmlabju"  # example format; paste without spaces
      EMAIL_TRANSPORT_DEFAULT_TLS: "true"                   # must be quoted
```
3) Apply changes:
```bash
sudo docker compose down           # Stop current containers (keeps volumes)
sudo docker compose up -d          # Recreate with new env vars
```

---

## 9) Optional — Enable HTTPS (443)

- If you have a **domain** pointing to `13.204.79.56`, add a reverse proxy (e.g., Nginx + Let’s Encrypt) in front of Passbolt to terminate TLS.  
- Exposing container port `443` alone isn’t enough unless valid certs exist inside the container.  
- For production, use a domain + Let’s Encrypt; IP-only HTTPS will show warnings or fail verification.

---

## 10) Useful lifecycle commands
```bash
# See running containers + names
docker ps

# Tail logs
sudo docker compose logs -f passbolt

# Restart services
sudo docker compose restart

# Stop and remove containers (keep volumes/data)
sudo docker compose down

# Stop and remove EVERYTHING (including volumes/data) — fresh start
sudo docker compose down -v

# Clean unused images/containers/networks
sudo docker system prune -af
```

---

## 11) Common pitfalls & fixes

- **`EMAIL_TRANSPORT_DEFAULT_TLS contains true, invalid type`**  
  → In YAML env, everything should be strings. Use `"true"` or `"false"` (with quotes).

- **`The JWT private key could not be written`** (permissions)  
  → Run:  
  ```bash
  sudo docker exec -it passbolt-passbolt-1 bash -lc 'chown -R www-data:www-data /var/www/passbolt/config/{jwt,gpg} && chmod -R 700 /var/www/passbolt/config/{jwt,gpg}'
  ```
  and then re-run the installer.

- **`No such container: passbolt` when exec**  
  → Your real name is likely `passbolt-passbolt-1`. Run `docker ps` to confirm, then:
  ```bash
  sudo docker exec -it -u www-data passbolt-passbolt-1 bash
  ```

---

## 12) Backup notes (quick)
- Backup named volumes:
  - `db_data`  → MariaDB data
  - `gpg_data` → Passbolt GPG keys
  - `jwt_data` → JWT keys
- You can snapshot these via `docker run --rm -v <volume>:/v alpine tar` or use EBS snapshots at the disk level.

---

### ✅ You’re done!
You can now log in to Passbolt at **`http://13.204.79.56`** and manage users/secrets.  
For production, add **SMTP + HTTPS** and back up your volumes regularly.

