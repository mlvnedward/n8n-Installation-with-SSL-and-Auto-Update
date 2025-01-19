# n8n Deployment on Google Cloud Platform

This guide outlines the steps to deploy n8n on a GCP instance using Docker, Docker Compose, and Nginx with SSL. The setup ensures data persistence, auto-updates, and secure access.

---

## Prerequisites

- A Google Cloud Platform (GCP) account.
- A registered domain name.
- Basic familiarity with SSH and Linux commands.

---

## Step 1: Prepare Your GCP Instance

### 1. Create a Compute Engine Instance
1. Log into the GCP Console and navigate to **Compute Engine > VM Instances**.
2. Click **Create Instance** and configure the following:
   - **Name**: `n8n-instance` (or your preferred name).
   - **Machine Type**: `e2-micro` (free tier eligible).
   - **Boot Disk**: `Ubuntu 22.04 LTS`.
   - **Firewall**: Check both **Allow HTTP** and **Allow HTTPS**.
3. Click **Create** to start the instance.

### 2. SSH into Your Instance
1. Connect via SSH using the GCP Console:
   - Click the **SSH** button next to your instance.
2. Update system packages:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

---

## Step 2: Install Docker and Docker Compose

### 1. Install Docker
1. Install Docker:
   ```bash
   sudo apt install -y docker.io
   ```
2. Enable Docker to start on boot:
   ```bash
   sudo systemctl enable docker
   ```
3. Add your user to the Docker group:
   ```bash
   sudo usermod -aG docker $USER
   ```
4. Log out and back in to apply the group changes or run:
   ```bash
   newgrp docker
   ```
5. Verify Docker installation:
   ```bash
   docker --version
   ```

### 2. Install Docker Compose
1. Download Docker Compose:
   ```bash
   sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   ```
2. Apply executable permissions:
   ```bash
   sudo chmod +x /usr/local/bin/docker-compose
   ```
3. Verify Docker Compose installation:
   ```bash
   docker-compose --version
   ```

---

## Step 3: Configure n8n with Docker Compose

### 1. Set Up Directories
1. Create the necessary directories:
   ```bash
   mkdir ~/n8n-docker
   mkdir ~/n8n-data
   ```
2. Ensure the `n8n-data` directory has the correct permissions:
   ```bash
   sudo chown 1000:1000 ~/n8n-data
   ```

### 2. Create the `docker-compose.yml` File
1. Navigate to the `n8n-docker` directory:
   ```bash
   cd ~/n8n-docker
   ```
2. Open the file:
   ```bash
   nano docker-compose.yml
   ```
3. Add the following content (replace `your-domain.com` with your domain):
   ```yaml
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
      - GENERIC_TIMEZONE=Asia/Kolkata
      - N8N_PROTOCOL=https
      - N8N_PORT=443
      - N8N_TRUST_PROXY=true
    volumes:
      - ~/n8n-data:/home/node/.n8n

   ```
4. Save and close the file.

### 3. Start the n8n Container
1. Launch the container:
   ```bash
   docker-compose up -d
   ```
2. Verify the container is running:
   ```bash
   docker ps
   ```

---

## Step 4: Configure Nginx Reverse Proxy

### 1. Install Nginx
1. Install Nginx:
   ```bash
   sudo apt install -y nginx
   ```

### 2. Set Up Reverse Proxy
1. Create an Nginx configuration file for your domain:
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
3. Enable the configuration:
   ```bash
   sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl reload nginx
   ```

---

## Step 5: Enable SSL

### Option 1: Using Cloudflare
1. Add your domain to Cloudflare and update DNS records to point to your instance's IP address.
2. Enable **Full (Strict)** SSL mode in Cloudflare settings.

### Option 2: Using Certbot
1. Install Certbot:
   ```bash
   sudo apt install -y certbot python3-certbot-nginx
   ```
2. Obtain and configure an SSL certificate:
   ```bash
   sudo certbot --nginx -d your-domain.com
   ```
3. Verify automatic renewal:
   ```bash
   sudo certbot renew --dry-run
   ```

---

## Step 6: Set Up Watchtower for Auto-Updates

1. Install Watchtower:
   ```bash
   docker run -d \
   --name watchtower \
   --restart unless-stopped \
   -v /var/run/docker.sock:/var/run/docker.sock \
   containrrr/watchtower --cleanup --schedule "0 3 * * *"
   ```

---

## Step 7: Validate Your Setup

1. **Check n8n Logs**:
   ```bash
   docker logs n8n
   ```
2. **Verify Data Persistence**:
   - Create a workflow in n8n.
   - Ensure workflow data is saved in `~/n8n-data`:
     ```bash
     ls ~/n8n-data
     ```
3. **Test Updates**:
   - Stop and remove the n8n container:
     ```bash
     docker stop n8n && docker rm n8n
     ```
   - Restart using `docker-compose up -d` and verify data persistence.

---

## Final Notes

- **Backups**: Regularly back up the `~/n8n-data` directory to prevent data loss.
- **Monitoring**: Use tools like UptimeRobot to monitor service availability.
- **Support**: If issues occur, inspect container logs (`docker logs n8n`) and check the configuration files.

This setup ensures a reliable, secure, and auto-updating n8n deployment on GCP.


# Automatic System Update, Upgrade, and Reboot with Cron

This guide outlines how to automatically update, upgrade, and restart your system using cron jobs on Ubuntu. The cron job will run weekly without requiring manual intervention.

## Steps to Set Up

### Step 1: Configure the Cron Job
1. Open the cron jobs file for editing:
   ```bash
   sudo crontab -e
   ```

2. Add the following cron job to run the updates, upgrades, and restart your system. This example runs it every day at 2 AM:
   ```bash
   0 2 * * 0 sudo apt update && sudo apt upgrade -y && sudo reboot
   ```

3. **Save and exit** the crontab editor:
   - Press `Ctrl + X`
   - Press `Y` to confirm
   - Press `Enter` to save

### Step 2: Allow `sudo` to Run Without Password
Since the cron job uses `sudo`, you need to allow the system to run the `apt` and `reboot` commands without requiring a password.

1. Edit the sudoers file to allow passwordless execution of the update, upgrade, and reboot commands:
   ```bash
   sudo visudo
   ```

2. Add the following line to the sudoers file to allow passwordless execution of `apt` and `reboot` commands:
   ```bash
   your_username ALL=NOPASSWD: /usr/bin/apt update, /usr/bin/apt upgrade, /sbin/reboot
   ```

   Replace `your_username` with your actual username (you can check your username by running `whoami`).

3. **Save and exit** the editor:
   - Press `Ctrl + X`
   - Press `Y` to confirm
   - Press `Enter` to save

### Step 3: Test the Cron Job
To verify the configuration works without requiring a password, manually run the following command:
```bash
sudo apt update && sudo apt upgrade -y && sudo reboot
```

If the update and upgrade commands run, and the system restarts without prompting for a password, your configuration is correct.

## Final Setup Summary
- The cron job will automatically:
  - Run `sudo apt update` and `sudo apt upgrade -y`.
  - Restart the system with `sudo reboot` if necessary.
- The system will execute the cron job weekly at 2 AM without requiring password input because `sudo` permissions have been configured to allow these commands without a password.

This setup ensures your system is regularly updated and rebooted without manual intervention.

