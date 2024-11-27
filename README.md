# Self-Hosting SSL-Enabled n8n with Watchtower Integration on Google Cloud Platform (GCP)

## Step 1: Prepare Your GCP Instance

### Create an Instance
1. Log into GCP Console and navigate to Compute Engine.
2. Click **Create Instance**.
3. Choose the following settings:
   - **Machine Type**: Select the `e2-micro` type (eligible for the free tier).
   - **Boot Disk**: Select `Ubuntu 20.04 LTS` or `Ubuntu 22.04 LTS`.
   - **Firewall**: Allow `HTTP` and `HTTPS` traffic.
4. Create the Instance.

### SSH into Your Instance
Once the instance is created, connect to it via SSH using the GCP Console or your terminal:

### Update System Packages
Before you proceed, ensure your system is up to date:
```bash
sudo apt update && sudo apt upgrade -y
```

## Step 2: Install Docker and Docker-Compose

### Install Docker
Run the following commands to install Docker and Docker Compose:
```bash
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
```

### Verify Docker Installation
To confirm Docker is installed correctly, run:
```bash
docker --version
```

## Step 3: Set Up Docker Compose for n8n

### Create a Directory for n8n
Create a directory to store all your n8n files and configurations:
```bash
mkdir ~/n8n-docker && cd ~/n8n-docker
```

### Set Up Persistent Volume for n8n Data
Ensure your n8n workflows are stored persistently:
```bash
mkdir -p ~/.n8n
```

### Create docker-compose.yml File
In the `~/n8n-docker` directory, create and edit a new `docker-compose.yml` file:
```bash
nano docker-compose.yml
```

### Add the following content to the file (replace your-domain.com with your domain):
```yaml
version: "3.1"

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=your-domain.com
      - WEBHOOK_TUNNEL_URL=https://your-domain.com/
      - WEBHOOK_URL=https://your-domain.com/
    volumes:
      - ~/.n8n:/root/.n8n
```

### Start n8n Container Using Docker Compose
To start the n8n container, run:
```bash
docker-compose up -d
```

## Step 4: Install and Configure Nginx for SSL and Reverse Proxy

### Install Nginx
Install Nginx to set up a reverse proxy for n8n and to handle SSL encryption:
```bash
sudo apt install -y nginx
```

### Configure Nginx for Reverse Proxy
Create an Nginx configuration file for your domain:
```bash
sudo nano /etc/nginx/sites-available/n8n
```

### Add the following content (replace your-domain.com with your domain):
```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### Enable Nginx Configuration
Enable the Nginx configuration and reload:
```bash
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## Step 5: Set Up SSL (Cloudflare or Certbot)

### Using Cloudflare for SSL
If you're using Cloudflare for DNS and SSL:
1. Sign up for Cloudflare and add your domain.
2. Update your DNS settings to point to your GCP instance IP address.
3. Enable Full (Strict) SSL mode in the Cloudflare SSL settings.

### Using Certbot for SSL (If Not Using Cloudflare)
If you're not using Cloudflare and want to use Certbot for SSL, run:
```bash
sudo apt install -y certbot python3-certbot-nginx
```

Then, run Certbot to configure SSL for your domain:
```bash
sudo certbot --nginx -d your-domain.com
```

## Step 6: Install and Configure Watchtower for Automatic Updates

### Install Watchtower
To automatically update the n8n container, install Watchtower:
```bash
docker run -d \
    --name watchtower \
    --restart unless-stopped \
    -v /var/run/docker.sock:/var/run/docker.sock \
    containrrr/watchtower --cleanup --schedule "0 3 * * *

