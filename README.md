#!/bin/bash

# Self-Hosting SSL-enabled n8n with Watchtower Integration on GCP

echo "Step 1: Installing Docker..."
sudo apt update -y
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker

echo "Step 2: Setting up Persistent Storage for n8n..."
sudo mkdir -p /mnt/disks/n8n-data
sudo mount /dev/sda1 /mnt/disks/n8n-data
echo '/dev/sda1 /mnt/disks/n8n-data ext4 defaults 0 2' | sudo tee -a /etc/fstab

echo "Step 2.3: Running n8n in Docker..."
sudo docker run -d --restart unless-stopped -it \
  --name n8n \
  -p 5678:5678 \
  -e N8N_HOST="your-domain.com" \
  -e WEBHOOK_TUNNEL_URL="https://your-domain.com/" \
  -e WEBHOOK_URL="https://your-domain.com/" \
  -v /mnt/disks/n8n-data:/root/.n8n \
  n8nio/n8n

echo "Step 3: Installing and Configuring Nginx..."
sudo apt install -y nginx

echo "Creating Nginx Configuration for n8n..."
cat <<EOL | sudo tee /etc/nginx/sites-available/n8n
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";
        chunked_transfer_encoding off;
        proxy_buffering off;
        proxy_cache off;
    }
}
EOL

sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx

echo "Step 4: Setting up SSL with Certbot..."
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain.com -d www.your-domain.com
sudo certbot renew --dry-run

echo "Step 5: Installing Watchtower for Automatic Updates..."
sudo docker run -d --name watchtower --restart unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  v2tec/watchtower --interval 86400 --cleanup

echo "Setup Complete. Verify all services are running as expected."
