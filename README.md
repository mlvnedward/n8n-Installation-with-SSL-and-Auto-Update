
# Self-Hosting SSL-enabled n8n with Watchtower Integration on Google Cloud Platform (GCP)

## Step 1: Installing Docker

### 1.1. Update the Package Index
```bash
sudo apt update
```

### 1.2. Install Docker
```bash
sudo apt install docker.io
```

### 1.3. Start Docker
```bash
sudo systemctl start docker
```

### 1.4. Enable Docker to Start at Boot
```bash
sudo systemctl enable docker
```

## Step 2: Starting n8n in Docker

### 2.1. Create Persistent Storage Directory (Assuming you're using GCP's Persistent Disk)
```bash
sudo mkdir -p /mnt/disks/n8n-data
sudo mount /dev/sda1 /mnt/disks/n8n-data
```

### 2.2. Ensure the Mount is Persistent Across Reboots
Add the mount to `/etc/fstab`:
```bash
echo '/dev/sda1 /mnt/disks/n8n-data ext4 defaults 0 2' | sudo tee -a /etc/fstab
```

### 2.3. Run n8n in Docker
Run the following command to start n8n in Docker. Replace `your-domain.com` with your actual domain or subdomain:
```bash
sudo docker run -d --restart unless-stopped -it   --name n8n   -p 5678:5678   -e N8N_HOST="your-domain.com"   -e WEBHOOK_TUNNEL_URL="https://your-domain.com/"   -e WEBHOOK_URL="https://your-domain.com/"   -v /mnt/disks/n8n-data:/root/.n8n   n8nio/n8n
```

> **Important**: This setup ensures that your n8n data is stored on the GCP Persistent Disk under `/mnt/disks/n8n-data`.

## Step 3: Installing Nginx for Reverse Proxy

### 3.1. Install Nginx
```bash
sudo apt install nginx
```

## Step 4: Configuring Nginx

### 4.1. Create a New Nginx Configuration File
```bash
sudo nano /etc/nginx/sites-available/n8n
```

### 4.2. Paste the Following Nginx Configuration:
```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        chunked_transfer_encoding off;
        proxy_buffering off;
        proxy_cache off;
    }
}
```

> Replace `your-domain.com` with your actual domain or subdomain.

### 4.3. Enable the Nginx Configuration
```bash
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
```

### 4.4. Test the Nginx Configuration and Restart
```bash
sudo nginx -t
sudo systemctl restart nginx
```

## Step 5: Installing Certbot for SSL

### 5.1. Install Certbot and Nginx Plugin
```bash
sudo apt install certbot python3-certbot-nginx
```

### 5.2. Obtain SSL Certificates with Certbot
```bash
sudo certbot --nginx -d your-domain.com -d www.your-domain.com
```

> Replace `your-domain.com` with your actual domain.

### 5.3. Set Up Automatic SSL Renewal
Certbot automatically sets up a cron job for renewal, but you can manually test it using:
```bash
sudo certbot renew --dry-run
```

## Step 6: Installing Watchtower for Automatic Updates

### 6.1. Run Watchtower in Docker
This will automatically update the n8n container every 24 hours.
```bash
sudo docker run -d   --name watchtower   --restart unless-stopped   -v /var/run/docker.sock:/var/run/docker.sock   v2tec/watchtower   --interval 86400  # Check for updates every 24 hours
```

### 6.2. Optional: Enable Cleanup of Old Images
If you want Watchtower to automatically remove old images after updating the container, use the `--cleanup` flag:
```bash
sudo docker run -d   --name watchtower   --restart unless-stopped   -v /var/run/docker.sock:/var/run/docker.sock   v2tec/watchtower   --interval 86400   --cleanup
```

---

## Troubleshooting

1. **Nginx Configuration Issues**: Ensure that the domain is correctly configured and that Nginx is able to reach the `n8n` container on `localhost:5678`.

2. **Persistent Storage Issues**: If you face issues with persistent storage, verify that the volume mount is correctly set up and that Docker has proper access permissions to the GCP disk.

3. **Watchtower Updates**: Ensure that Watchtower has permission to access Docker's socket. If you're using any firewall or restrictive network settings, make sure Docker and Watchtower can communicate correctly.

For more detailed assistance, you can refer to the official n8n and Watchtower documentation.
