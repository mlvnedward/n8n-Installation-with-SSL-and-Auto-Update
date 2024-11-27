
# Full Setup Guide for Self-Hosting SSL-enabled n8n with Watchtower Integration

## Step 1: Installing Docker

1. **Update the Package Index:**
   ```bash
   sudo apt update
   ```

2. **Install Docker:**
   ```bash
   sudo apt install docker.io
   ```

3. **Start Docker:**
   ```bash
   sudo systemctl start docker
   ```

4. **Enable Docker to Start at Boot:**
   ```bash
   sudo systemctl enable docker
   ```

---

## Step 2: Starting n8n in Docker

Run the following command to start n8n in Docker, replacing `your-domain.com` with your actual domain or subdomain:

```bash
sudo docker run -d --restart unless-stopped -it   --name n8n   -p 5678:5678   -e N8N_HOST="your-domain.com"   -e WEBHOOK_TUNNEL_URL="https://your-domain.com/"   -e WEBHOOK_URL="https://your-domain.com/"   -v ~/.n8n:/root/.n8n   n8nio/n8n
```

Or, for a subdomain:

```bash
sudo docker run -d --restart unless-stopped -it   --name n8n   -p 5678:5678   -e N8N_HOST="subdomain.your-domain.com"   -e WEBHOOK_TUNNEL_URL="https://subdomain.your-domain.com/"   -e WEBHOOK_URL="https://subdomain.your-domain.com/"   -v ~/.n8n:/root/.n8n   n8nio/n8n
```

This will expose n8n on port 5678, mount the persistent volume, and configure the environment variables.

---

## Step 3: Installing Nginx for Reverse Proxy

1. **Install Nginx:**
   ```bash
   sudo apt install nginx
   ```

---

## Step 4: Configuring Nginx

1. **Create a New Nginx Configuration File:**
   ```bash
   sudo nano /etc/nginx/sites-available/n8n
   ```

2. **Paste the Following Nginx Configuration:**
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

   Replace `your-domain.com` with your actual domain or subdomain.

3. **Enable the Nginx Configuration:**
   ```bash
   sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
   ```

4. **Test the Nginx Configuration and Restart:**
   ```bash
   sudo nginx -t
   sudo systemctl restart nginx
   ```

---

## Step 5: Installing Watchtower for Automatic Updates

To automatically update the n8n container with new images, install and run Watchtower.

1. **Run Watchtower in Docker:**
   ```bash
   sudo docker run -d      --name watchtower      --restart unless-stopped      -v /var/run/docker.sock:/var/run/docker.sock      v2tec/watchtower      --interval 86400  # 24 hours check for updates
   ```

   Explanation:
   - `-v /var/run/docker.sock:/var/run/docker.sock`: Mounts the Docker socket for Watchtower to access the Docker daemon.
   - `--interval 86400`: Sets Watchtower to check for updates every 24 hours. You can adjust the interval as needed.

2. **Optional - Enable Cleanup for Old Images:**
   If you want Watchtower to automatically remove old images after updating the container, you can add the `--cleanup` flag:

   ```bash
   sudo docker run -d      --name watchtower      --restart unless-stopped      -v /var/run/docker.sock:/var/run/docker.sock      v2tec/watchtower      --interval 86400      --cleanup
   ```

---

## Step 6: Persistent Storage and Data

You’ve already configured persistent storage using the following Docker volume mapping in the n8n container:

```bash
-v ~/.n8n:/root/.n8n
```

This ensures that the data is stored on your host machine under `~/.n8n`, and your container updates via Watchtower will not affect this data.

---

## Step 7: SSL Setup with Certbot

You can use Certbot to set up SSL for n8n with Nginx. If Certbot is not installed, you can do it with the following commands:

1. **Install Certbot and Nginx Plugin:**
   ```bash
   sudo apt install certbot python3-certbot-nginx
   ```

2. **Obtain the SSL Certificate:**
   Run the following command to obtain and install the SSL certificate for your domain:

   ```bash
   sudo certbot --nginx -d your-domain.com
   ```

   Replace `your-domain.com` with your actual domain name. Certbot will automatically configure Nginx to use SSL and will update your Nginx configuration file.

3. **Test the SSL Configuration:**
   ```bash
   sudo nginx -t
   sudo systemctl restart nginx
   ```

4. **Renew SSL Certificates Automatically:**
   Certbot sets up a cron job to renew the SSL certificate automatically. You can check the renewal status with:

   ```bash
   sudo certbot renew --dry-run
   ```

---

## Final Setup Overview

- **n8n**: Running in Docker with persistent data and SSL enabled via Nginx.
- **Watchtower**: Automatically updates `n8n` without losing any existing data by monitoring the container and pulling the latest image.
- **SSL**: Handled by Certbot with Nginx for secure communication.

You now have a fully configured self-hosted n8n setup with SSL, automatic updates via Watchtower, and persistent data storage.

---

### How to Upload to GitHub

1. **Create a new GitHub repository**.
2. **Create a new file** in the repository called `README.md`.
3. **Copy and paste the entire content** above into the `README.md` file.
4. **Commit and push** the changes to GitHub.

Now, the guide is available in your GitHub repository for easy reference and sharing.
