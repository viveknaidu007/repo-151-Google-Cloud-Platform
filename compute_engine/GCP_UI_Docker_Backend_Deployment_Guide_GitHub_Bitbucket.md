# GCP Console (UI): Deploy a Docker Backend on a Compute Engine VM
This end-to-end guide uses the Google Cloud Console (web UI) to:
- Create a Compute Engine VM (e2-medium) and reserve a permanent external IP
- Open firewall for HTTP/HTTPS and your app port (allow from 0.0.0.0/0 if desired)
- SSH into the VM (browser SSH)
- Install Docker, Docker Compose plugin, and Git
- Configure SSH and clone from GitHub or Bitbucket (both supported)
- Create a .env file
- Build and run your backend with Docker or Docker Compose
- Optional: NGINX reverse proxy and HTTPS via Let’s Encrypt

Note: “EC2” is AWS terminology. In GCP, pick the E2 series and the e2-medium type.

---

## Prerequisites
- GCP project with billing enabled
- Permissions to create VM instances, firewall rules, and reserve external IPs
- Repository with Dockerfile (or docker-compose.yaml)
- Your app’s listening port (APP_PORT, e.g., 8080)

---

## Part A — Create the VM in the GCP Console
1) Open the Console → Navigation menu → Compute Engine → VM instances → Create instance  
2) Basic:
- Name: backend-vm (or as you prefer)
- Region/Zone: close to your users (e.g., us-central1/us-central1-a)
- Machine family and type:
  - Series: E2
  - Machine type: e2-medium (2 vCPU, 4 GB)
- Boot disk:
  - Change → OS: Ubuntu → Version: Ubuntu 24.04 LTS
  - Size: 30 GB (or more)
  - Select

3) Firewall (on the VM creation page)
- Check “Allow HTTP traffic”
- Check “Allow HTTPS traffic”

4) Networking (reserve static IP and add a network tag)
- Advanced options → Networking → Network interfaces → default → Edit
  - External IP → Create IP address → Name: backend-static-ip → Reserve
  - Done
- Network tags: add a tag to target custom firewall rules (e.g., backend-app)

5) Create the VM
- Click Create and wait for provisioning. The External IP shown is your permanent IP because you reserved it during creation.

---

## Part B — Open Firewall for Your App Port (UI)
If your app listens on APP_PORT (e.g., 8080), create a firewall rule:

1) Navigation menu → VPC network → Firewall → Create firewall rule  
2) Fill:
- Name: allow-backend-app-port
- Network: default (or your chosen VPC)
- Targets: Specified target tags
  - Target tags: backend-app (the same tag on your VM)
- Source filter: IPv4 ranges
  - Source IPv4 ranges: 0.0.0.0/0
  - Note: 0.0.0.0/0 allows traffic from anywhere. Restrict in production if possible.
- Protocols and ports: Specified protocols and ports → tcp:8080 (replace with your APP_PORT)
3) Create

HTTP/HTTPS are already allowed by the VM creation checkboxes; if needed, you can also create explicit firewall rules targeting the same tag with ports 80 and 443.

---

## Part C — Connect to the VM (Browser SSH)
- Compute Engine → VM instances → Click SSH next to your VM to open a browser terminal.

---

## Part D — Install Docker and Git (on the VM)
Run inside the VM SSH session (Ubuntu 24.04):

```bash
# Update and basics
sudo apt-get update -y
sudo apt-get upgrade -y
sudo apt-get install -y ca-certificates curl gnupg lsb-release git

# Docker’s official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Docker apt repo (Ubuntu 24.04 = noble)
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine + Buildx + Compose plugin
sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Enable and start Docker
sudo systemctl enable docker
sudo systemctl start docker

# Use docker without sudo
sudo usermod -aG docker "$USER"
newgrp docker

# Verify
docker version
docker run --rm hello-world
```

---

## Part E — Configure SSH for GitHub and Bitbucket
You can:
- Use one SSH key for both, or
- Create separate keys (recommended for separation).

Below shows separate keys and per-host SSH config.

1) Generate keys on the VM
```bash
# GitHub key
ssh-keygen -t ed25519 -C "gcp-vm-github" -f ~/.ssh/github_ed25519 -N ""

# Bitbucket key
ssh-keygen -t ed25519 -C "gcp-vm-bitbucket" -f ~/.ssh/bitbucket_ed25519 -N ""
```

2) SSH config to map each host to its key
```bash
cat >> ~/.ssh/config <<'EOF'
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/github_ed25519
  IdentitiesOnly yes

Host bitbucket.org
  HostName bitbucket.org
  User git
  IdentityFile ~/.ssh/bitbucket_ed25519
  IdentitiesOnly yes
EOF
chmod 600 ~/.ssh/config

# Load the keys (optional — SSH will auto-load when used)
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/github_ed25519
ssh-add ~/.ssh/bitbucket_ed25519
```

3) Add the public keys where appropriate

GitHub (Deploy Key on a single repo):
- Copy the public key:
  ```bash
  echo "=== GitHub Deploy Key (public) ==="
  cat ~/.ssh/github_ed25519.pub
  ```
- In GitHub: Repository → Settings → Deploy keys → Add deploy key
  - Title: GCP VM Deploy Key (GitHub)
  - Key: paste the public key
  - Allow write access: check only if the VM needs to push; otherwise leave read-only

GitHub (Personal account key — for multiple repos via your user):
- GitHub → Profile menu (top-right avatar) → Settings → SSH and GPG keys → New SSH key
- Paste the above public key
- This grants your user access based on your repo permissions

Bitbucket Cloud (Repository Access Key = Deploy Key):
- Copy the public key:
  ```bash
  echo "=== Bitbucket Access Key (public) ==="
  cat ~/.ssh/bitbucket_ed25519.pub
  ```
- In Bitbucket Cloud: Repository → Repository settings → Access keys → Add key
  - Label: GCP VM Access Key (Bitbucket)
  - Key: paste the public key
  - Note: Repository access keys are read-only in Bitbucket Cloud. If you need write access (push), use a Personal SSH key on the user account instead.

Bitbucket Cloud (Personal SSH Key on your account):
- Bitbucket → Avatar (bottom-left) → Personal settings → SSH keys → Add key
- Paste the public key
- Your ability to access repos depends on the permissions of your Bitbucket user.

4) Test SSH access
```bash
ssh -T git@github.com
# You should see a success/permission message

ssh -T git@bitbucket.org
# You should see a success/permission message
```

---

## Part F — Clone Your Repository (GitHub or Bitbucket)

- GitHub SSH URL format: git@github.com:OWNER/REPO.git  
- Bitbucket Cloud SSH URL format: git@bitbucket.org:WORKSPACE/REPO.git

Examples:
```bash
# Example: GitHub
REPO_SSH="git@github.com:owner/repo.git"

# Example: Bitbucket Cloud
# REPO_SSH="git@bitbucket.org:workspace/repo.git"

cd ~
git clone "$REPO_SSH"
cd "$(basename "$REPO_SSH" .git)"
```

---

## Part G — Create and Populate .env
If your repo has .env.example:
```bash
cp .env.example .env
nano .env    # set production values; ensure PORT/APP_PORT and bind address 0.0.0.0
```

Or create from scratch:
```bash
cat > .env <<'EOF'
# Example environment variables
NODE_ENV=production
PORT=8080
DATABASE_URL=postgres://user:pass@host:5432/dbname
JWT_SECRET=replace_me
# Add other backend-specific variables here
EOF
```

Ensure your app listens on 0.0.0.0 and the APP_PORT so it’s reachable externally.

---

## Part H — Build and Run with Docker

Option A: Docker Compose (if docker-compose.yaml exists)
```bash
docker compose up -d --build
docker compose ps
docker compose logs -f
```
Tip in your compose service:
```yaml
restart: unless-stopped
```

Option B: Dockerfile only
```bash
APP_PORT=8080

# Build
docker build -t my-backend:latest .

# Run
docker run -d --name my-backend \
  --env-file .env \
  -p ${APP_PORT}:${APP_PORT} \
  --restart unless-stopped \
  my-backend:latest

# Logs
docker logs -f my-backend
```

---

## Part I — Verify Static External IP and Test
- Compute Engine → VM instances → find the External IP for your VM (it’s static if you reserved it during creation)
- Test from your machine:
  ```bash
  # Direct exposure (no proxy)
  curl -I http://YOUR_STATIC_IP:8080

  # If you later use NGINX on 80/443:
  curl -I http://YOUR_STATIC_IP
  ```

If unreachable, check:
- Firewall rule allows tcp:APP_PORT and targets your VM’s network tag
- Container is running and the app binds 0.0.0.0:APP_PORT
- Docker port mapping matches container port
- Use docker compose logs or docker logs for errors

---

## Optional — NGINX Reverse Proxy + HTTPS
1) Point your domain’s A record to the VM’s static IP  
2) Install NGINX:
```bash
sudo apt-get update -y
sudo apt-get install -y nginx
```
3) Simple reverse proxy to your app:
```bash
APP_PORT=8080
sudo bash -c 'cat > /etc/nginx/sites-available/backend <<EOF
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://127.0.0.1:'"$APP_PORT"';
        proxy_http_version 1.1;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        proxy_read_timeout 300;
    }
}
EOF'
sudo ln -s /etc/nginx/sites-available/backend /etc/nginx/sites-enabled/backend
sudo nginx -t && sudo systemctl reload nginx
```
4) HTTPS with Let’s Encrypt (Certbot) after DNS resolves:
```bash
sudo apt-get install -y snapd
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

sudo certbot --nginx -d your-domain.com
sudo certbot renew --dry-run
```
Ensure firewall allows tcp:80, tcp:443.

---

## Maintenance Cheatsheet
```bash
# Containers
docker ps
docker logs -f <container>
docker restart <container>
docker stop <container> && docker rm <container>

# Compose
docker compose ps
docker compose logs -f
docker compose restart
docker compose up -d --build
docker compose down

# Rebuild (Dockerfile-only)
docker build -t my-backend:latest .
docker stop my-backend && docker rm my-backend
docker run -d --name my-backend --env-file .env -p ${APP_PORT}:${APP_PORT} --restart unless-stopped my-backend:latest

# VM updates
sudo apt-get update -y && sudo apt-get upgrade -y
```

---

## Security Notes
- Source IPv4 0.0.0.0/0 allows traffic from anywhere. Prefer restricting to trusted CIDRs or use a reverse proxy with standard ports and additional protection.
- Keep .env out of git; store secrets only on the VM (or use a secret manager).
- Use separate SSH keys per provider; rotate if compromised.
- Consider enabling OS Login and IAM roles to control SSH access.

---

## Appendix A — Common SSH Clone URLs
- GitHub: git@github.com:OWNER/REPO.git
- Bitbucket Cloud: git@bitbucket.org:WORKSPACE/REPO.git

## Appendix B — Example ~/.ssh/config (separate keys)
```sshconfig
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/github_ed25519
  IdentitiesOnly yes

Host bitbucket.org
  HostName bitbucket.org
  User git
  IdentityFile ~/.ssh/bitbucket_ed25519
  IdentitiesOnly yes
```

---

## Quick Start Summary
1) Console → Compute Engine → Create VM (Ubuntu 24.04, E2 → e2-medium, 30 GB), allow HTTP/HTTPS, reserve static IP, add tag backend-app  
2) VPC → Firewall → allow tcp:APP_PORT from 0.0.0.0/0 targeting backend-app  
3) SSH (browser) → install Docker + Git (commands above)  
4) Create SSH keys and add:
   - GitHub: Repo → Settings → Deploy keys OR Profile → Settings → SSH keys
   - Bitbucket: Repo → Repository settings → Access keys (read-only) OR Avatar → Personal settings → SSH keys  
5) git clone via SSH → create .env  
6) docker compose up -d --build OR docker build/run  
7) Test http://STATIC_IP:APP_PORT or set up NGINX + HTTPS