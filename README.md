
# Self-Hosting SSL-Enabled n8n with Watchtower on Google Cloud Platform (GCP)

This guide details how to deploy n8n on GCP with SSL and Watchtower integration while ensuring data persistence.

---

## Step 1: Prepare Your GCP Instance

### Create an Instance
1. Go to the GCP Console → Compute Engine → VM Instances → Create Instance.
2. Use the following settings:
   - **Machine Type**: `e2-micro` (Free tier eligible).
   - **Boot Disk**: Ubuntu 22.04 LTS (recommended) or 20.04 LTS.
   - **Firewall**: Enable `Allow HTTP traffic` and `Allow HTTPS traffic`.

### Connect to the Instance via SSH
1. Use the SSH button in the GCP console or your terminal:
   ```bash
   gcloud compute ssh <instance-name>
   ```
2. Update system packages:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

---

## Step 2: Install Docker and Docker-Compose

### Install Docker
```bash
sudo apt install -y docker.io
sudo systemctl enable docker
```

### Install Docker Compose
```bash
sudo apt install -y docker-compose
```

### Verify Installation
```bash
docker --version
docker-compose --version
```

---

## Step 3: Set Up n8n with Docker Compose

### Create Directories for n8n
```bash
mkdir ~/n8n-docker && cd ~/n8n-docker
mkdir -p /home/<your-username>/n8n-data
sudo chown -R 1000:1000 /home/<your-username>/n8n-data
```

### Create a `docker-compose.yml` File
1. Create the file:
   ```bash
   nano docker-compose.yml
   ```
2. Add the following configuration:
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
         - ${N8N_DATA_PATH}:/home/node/.n8n
   ```

### Create an `.env` File
1. Create the file:
   ```bash
   nano .env
   ```
2. Add the following line:
   ```bash
   N8N_DATA_PATH=/home/<your-username>/n8n-data
   ```

---

## Step 4: Configure Nginx for SSL and Reverse Proxy

### Install Nginx
```bash
sudo apt install -y nginx
```

### Configure Nginx
1. Create a configuration file:
   ```bash
   sudo nano /etc/nginx/sites-available/n8n
   ```
2. Add the following content (replace `your-domain.com` with your domain):
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

### Enable the Configuration
```bash
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## Step 5: Set Up SSL for Your Domain

### Option 1: Using Cloudflare
1. Add your domain to Cloudflare.
2. Update your DNS settings to point to your GCP instance's external IP address.
3. Enable **Full (Strict)** SSL mode in Cloudflare.

### Option 2: Using Certbot
1. Install Certbot:
   ```bash
   sudo apt install -y certbot python3-certbot-nginx
   ```
2. Generate SSL certificates:
   ```bash
   sudo certbot --nginx -d your-domain.com
   ```
3. Verify renewal with:
   ```bash
   sudo certbot renew --dry-run
   ```

---

## Step 6: Set Up Watchtower for Automatic Updates

### Install Watchtower
```bash
docker run -d     --name watchtower     --restart unless-stopped     -v /var/run/docker.sock:/var/run/docker.sock     containrrr/watchtower --cleanup --schedule "0 3 * * *"
```

---

## Step 7: Verify and Test Your Setup

### Access n8n
Navigate to `https://your-domain.com` to access the n8n UI.

### Test Workflow Persistence
1. Create a workflow in n8n.
2. Restart the container:
   ```bash
   docker-compose restart
   ```
3. Verify that the workflow persists.

### Test Watchtower Updates
Check Watchtower logs to ensure updates are being applied:
```bash
docker logs watchtower
```

---

## Conclusion
This setup provides:
- **Persistent workflows and data** across updates.
- **SSL-secured connections** for your n8n instance.
- **Automatic updates** with Watchtower.
