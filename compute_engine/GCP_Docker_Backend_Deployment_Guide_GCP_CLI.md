# GCP VM + Docker Backend Deployment Guide

This guide walks you end-to-end through:
- Creating a Google Compute Engine (GCE) VM
- Installing Docker and required tooling
- Configuring SSH keys for GitHub and cloning your repository
- Creating a .env file
- Building and running your backend Docker container
- Opening firewall ports
- Reserving and assigning a permanent static external IP to your VM
- Optional: NGINX reverse proxy and HTTPS

Use this as a reference deployment document for a Docker-based backend on Google Cloud Platform (GCP).

---

## Prerequisites

- A GCP project with billing enabled
- Owner/Editor permissions on the project or sufficient permissions to create VM, firewall rules, and addresses
- gcloud CLI installed and authenticated on your local machine
  - Install: https://cloud.google.com/sdk/docs/install
  - Authenticate: `gcloud auth login` and `gcloud auth application-default login`
- Your backend project has:
  - A Dockerfile (or docker-compose.yaml)
  - An `.env.example` file (recommended)
  - The port the backend listens on (example: 8080)

---

## Variables

Export these variables on your local machine to reuse in commands:

```bash
export PROJECT_ID="your-gcp-project-id"
export REGION="us-central1"          # e.g. us-central1
export ZONE="us-central1-a"          # e.g. us-central1-a
export INSTANCE_NAME="backend-vm"
export MACHINE_TYPE="e2-small"       # adjust to your workload
export BOOT_DISK_SIZE="30GB"
export IP_NAME="backend-static-ip"
export APP_PORT="8080"               # the container/app port exposed by your backend
export GIT_REPO_SSH="git@github.com:owner/repo.git"  # your repo SSH URL
```

Set the active project:

```bash
gcloud config set project "$PROJECT_ID"
```

---

## 1) Enable Compute Engine API (once per project)

```bash
gcloud services enable compute.googleapis.com
```

---

## 2) Create the VM

You can use the Console or the CLI. The CLI example below creates an Ubuntu LTS instance and tags for HTTP/HTTPS.

```bash
gcloud compute instances create "$INSTANCE_NAME" \
  --zone="$ZONE" \
  --machine-type="$MACHINE_TYPE" \
  --image-family="ubuntu-2404-lts" \
  --image-project="ubuntu-os-cloud" \
  --boot-disk-size="$BOOT_DISK_SIZE" \
  --tags="http-server,https-server,$INSTANCE_NAME" \
  --scopes="https://www.googleapis.com/auth/cloud-platform"
```

Notes:
- `--tags` allow us to attach firewall rules to this instance (http-server, https-server, and instance-specific tag).
- Adjust machine type, disk size, and zone to your needs.

Get the current ephemeral external IP (for initial access):

```bash
gcloud compute instances describe "$INSTANCE_NAME" --zone="$ZONE" \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)'
```

---

## 3) Open Firewall Ports

- Allow HTTP (80) and HTTPS (443):

```bash
gcloud compute firewall-rules create allow-http-https \
  --allow=tcp:80,tcp:443 \
  --target-tags=http-server,https-server \
  --direction=INGRESS
```

- Allow your app port (e.g., 8080):

```bash
gcloud compute firewall-rules create "allow-app-port-$APP_PORT" \
  --allow="tcp:$APP_PORT" \
  --target-tags="$INSTANCE_NAME" \
  --direction=INGRESS
```

If a rule already exists, you can update it or create a new one with a different name.

---

## 4) SSH into the VM

Use gcloud to connect:

```bash
gcloud compute ssh "$INSTANCE_NAME" --zone="$ZONE"
```

Tip: This method uses OS Login/metadata-managed keys and is recommended.

---

## 5) Install Docker, Git, and Utilities on the VM (Ubuntu)

Run the following on the VM:

```bash
# Update
sudo apt-get update -y
sudo apt-get upgrade -y

# Required packages
sudo apt-get install -y ca-certificates curl gnupg lsb-release git

# Add Docker’s official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine, Buildx, Compose plugin
sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Enable & start Docker
sudo systemctl enable docker
sudo systemctl start docker

# Allow current user to run docker without sudo
sudo usermod -aG docker "$USER"
newgrp docker
```

Verify Docker:

```bash
docker version
docker run --rm hello-world
```

---

## 6) Configure SSH Keys on the VM for GitHub (Deploy Key recommended)

To clone private repos via SSH from the VM, create a dedicated key pair on the VM and add it as a Deploy Key on GitHub. This keeps secrets localized to the VM.

On the VM:

```bash
# Generate key pair (no passphrase for non-interactive deployments)
ssh-keygen -t ed25519 -C "gcp-vm-deploy-key" -f ~/.ssh/github_ed25519 -N ""

# Configure SSH to use this key for GitHub
cat >> ~/.ssh/config <<'EOF'
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/github_ed25519
  IdentitiesOnly yes
EOF

chmod 600 ~/.ssh/config
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/github_ed25519

# Show the public key to add as a Deploy Key in GitHub
echo "Public key below. Copy and add to your GitHub repo Settings > Deploy keys:"
cat ~/.ssh/github_ed25519.pub
```

In GitHub:
- Navigate to your repository > Settings > Deploy keys > Add deploy key
- Title: e.g., “GCP VM Deploy Key”
- Paste the public key content
- Enable “Allow write access” if the VM will push; otherwise read-only is sufficient

Test the connection from the VM:

```bash
ssh -T git@github.com
# You should see a success or permission message
```

---

## 7) Clone the Repository on the VM

```bash
cd ~
git clone "$GIT_REPO_SSH"
# example: git clone git@github.com:owner/repo.git
cd "$(basename "$GIT_REPO_SSH" .git)"
```

---

## 8) Create and Populate the .env File

Option A: If the repo has `.env.example`:
```bash
cp .env.example .env
nano .env   # or use your editor to adjust variables
```

Option B: Create from scratch:
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

Keep `.env` out of source control. Ensure your Dockerfile or compose config reads this `.env`.

---

## 9) Build and Run with Docker

Choose the method that matches your project.

### Option A: Docker Compose (recommended if repo includes docker-compose.yaml)

Build and run in detached mode:

```bash
docker compose up -d --build
```

Check status and logs:

```bash
docker compose ps
docker compose logs -f
```

To rebuild after changes:

```bash
docker compose up -d --build
```

To stop:

```bash
docker compose down
```

Ensure your compose service specifies:
```yaml
restart: unless-stopped
```
to survive reboots.

### Option B: Dockerfile only

Build:

```bash
docker build -t my-backend:latest .
```

Run (replace 8080 with your APP_PORT if different):

```bash
docker run -d --name my-backend \
  --env-file .env \
  -p "$APP_PORT:$APP_PORT" \
  --restart unless-stopped \
  my-backend:latest
```

Check logs:

```bash
docker logs -f my-backend
```

---

## 10) Reserve a Permanent Static External IP

A reserved static IP ensures your server’s IP doesn’t change across restarts.

Create the static IP in your region:

```bash
gcloud compute addresses create "$IP_NAME" --region="$REGION"
```

Get the reserved IP:

```bash
STATIC_IP="$(gcloud compute addresses describe "$IP_NAME" --region="$REGION" --format='get(address)')"
echo "Reserved static IP: $STATIC_IP"
```

---

## 11) Assign the Static IP to Your VM

By default, the VM has an ephemeral IP attached. We’ll replace it with the reserved static IP. First, find the access config name:

```bash
ACCESS_CONFIG_NAME="$(gcloud compute instances describe "$INSTANCE_NAME" --zone="$ZONE" \
  --format='get(networkInterfaces[0].accessConfigs[0].name)')"
echo "Current access-config name: $ACCESS_CONFIG_NAME"
```

Delete the existing (ephemeral) access config:

```bash
gcloud compute instances delete-access-config "$INSTANCE_NAME" \
  --zone="$ZONE" \
  --access-config-name="$ACCESS_CONFIG_NAME"
```

Add a new access config with the reserved static IP:

```bash
gcloud compute instances add-access-config "$INSTANCE_NAME" \
  --zone="$ZONE" \
  --address="$STATIC_IP" \
  --network-interface="nic0"
```

Verify:

```bash
gcloud compute instances describe "$INSTANCE_NAME" --zone="$ZONE" \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)'
```

You should now see the static IP assigned. Update your DNS to point to this IP (optional but recommended).

---

## 12) Test the Deployment

- From your local machine:
```bash
curl -I "http://$STATIC_IP:$APP_PORT"   # if exposed directly
# Or, if using reverse proxy on port 80:
curl -I "http://$STATIC_IP"
```

- From a browser, navigate to:
  - http://<STATIC_IP>:<APP_PORT> (direct mapping), or
  - http://<STATIC_IP> (if using NGINX reverse proxy to your container)

If the request fails, check:
- Container logs
- Firewall rules for the port you’re using
- Application bind address (ensure app binds to 0.0.0.0, not 127.0.0.1)
- Security groups/tags on the VM

---

## Optional: NGINX Reverse Proxy + HTTPS (Let’s Encrypt)

For production, it’s common to:
- Run your app on an internal port (e.g., 8080)
- Use NGINX on the host to proxy port 80/443 to the container
- Use Let’s Encrypt (Certbot) for TLS certificates

Install NGINX and Certbot on the VM:

```bash
sudo apt-get update -y
sudo apt-get install -y nginx

# Allow NGINX in UFW if you use it (GCE firewall already covers ingress)
# sudo ufw allow 'Nginx Full'
```

Create an NGINX site config (replace domain and port):

```bash
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
sudo nginx -t
sudo systemctl reload nginx
```

Point your domain’s A record to the STATIC_IP, wait for DNS to propagate, then use Certbot (one-liner via snap recommended; refer to Certbot docs if snap is not available):

```bash
sudo apt-get install -y snapd
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# Issue cert and auto-update NGINX config for HTTPS
sudo certbot --nginx -d your-domain.com

# Test auto-renew (dry run)
sudo certbot renew --dry-run
```

---

## Maintenance Commands

- View running containers:
```bash
docker ps
```

- Follow logs:
```bash
docker logs -f <container_name_or_id>
# or for compose:
docker compose logs -f
```

- Restart container:
```bash
docker restart <container_name_or_id>
# or for compose:
docker compose restart
```

- Rebuild after code changes:
```bash
docker compose up -d --build
# or (Dockerfile only)
docker build -t my-backend:latest .
docker stop my-backend && docker rm my-backend
docker run -d --name my-backend --env-file .env -p "$APP_PORT:$APP_PORT" --restart unless-stopped my-backend:latest
```

- System updates (VM):
```bash
sudo apt-get update -y && sudo apt-get upgrade -y
```

---

## Troubleshooting

- Port not reachable
  - Confirm GCE firewall rule allows that port and targets your instance’s network tag
  - Confirm app listens on 0.0.0.0 and correct port
  - Confirm container port mapping is correct (host:container)

- Cannot clone private repo on VM
  - Ensure the VM’s public key is added as a Deploy Key in the GitHub repo
  - `ssh -T git@github.com` should succeed from the VM

- Static IP not applied
  - Ensure you deleted the old access-config before adding the new one with `--address`
  - Confirm the static IP and region match the instance’s region

- Docker permission denied
  - Run `newgrp docker` or log out and back in after adding your user to the docker group

---

## References

- Compute Engine: https://cloud.google.com/compute/docs
- gcloud compute instances: https://cloud.google.com/sdk/gcloud/reference/compute/instances
- gcloud firewall rules: https://cloud.google.com/sdk/gcloud/reference/compute/firewall-rules
- gcloud addresses: https://cloud.google.com/sdk/gcloud/reference/compute/addresses
- Docker Engine on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Docker Compose: https://docs.docker.com/compose/
- Certbot: https://certbot.eff.org/

---
```bash
# Quick Start (summary):
# 1) gcloud config set project "$PROJECT_ID"
# 2) Create VM (Ubuntu) + firewall
# 3) gcloud compute ssh "$INSTANCE_NAME" --zone "$ZONE"
# 4) Install Docker & Git
# 5) Create GitHub deploy key on VM, add to repo
# 6) git clone "$GIT_REPO_SSH" && cd repo
# 7) Create .env
# 8) docker compose up -d --build  (or docker build/run)
# 9) Reserve static IP and assign it to VM
# 10) Test with curl/browser
```