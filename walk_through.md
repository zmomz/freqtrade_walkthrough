## **Freqtrade Deployment Walkthrough on Ubuntu 20.04**

---

###  **Step 1: Initial Server Setup & Security**

####  Create a Non-Root User
```bash
sudo adduser username
sudo usermod -aG sudo username
su - username
```

####  Disable Root Access
```bash
sudo usermod -L root
```

####  Set Host Timezone (Optional but Recommended)
```bash
sudo timedatectl set-timezone Etc/UTC  # Or your actual timezone
sudo timedatectl set-ntp true
```

####  Enable Automatic Updates
```bash
sudo apt update && sudo apt dist-upgrade -y
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

---

###  **Step 2: Harden SSH & Firewall**

####  Set up SSH Key Authentication
From your **local machine**:
```bash
ssh-keygen -b 4096
ssh-copy-id username@your_server_ip -p 22
```

On your **server**:
```bash
sudo nano /etc/ssh/sshd_config
# Change or add:
Port 1970
PermitRootLogin no
PasswordAuthentication no
AllowUsers username
```

```bash
sudo ufw allow 1970/tcp
sudo systemctl restart sshd
```

####  Configure UFW Firewall
```bash
sudo apt install ufw
sudo ufw allow 1970/tcp
sudo ufw enable
```

---

###  **Step 3: Install Docker & Docker Compose**

```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
sudo apt update
sudo apt install docker-ce -y
```

####  Install Docker Compose
```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

####  Optional: Run Docker without `sudo`
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```

---

###  **Step 4: Freqtrade Setup**

####  Create Bot Workspace
```bash
mkdir ~/ft_userdata && cd ~/ft_userdata
curl https://raw.githubusercontent.com/freqtrade/freqtrade/stable/docker-compose.yml -o docker-compose.yml
docker-compose pull
docker-compose run --rm freqtrade create-userdir --userdir user_data
```

####  Configure the Bot
```bash
docker-compose run --rm freqtrade new-config --config user_data/config.json
```

Edit:
```bash
nano user_data/config.json
# Set pair_whitelist, pairlists to StaticPairList, and heartbeat_interval
```

---

###  **Step 5: Backtest & Run the Bot**

####  Download Historical Data
```bash
docker-compose run --rm freqtrade download-data --config user_data/config.json --days 30 -t 5m
```

####  Backtest
```bash
docker-compose run --rm freqtrade backtesting --config user_data/config.json --strategy SampleStrategy
```

####  Edit Compose File to Run the Bot
```bash
nano docker-compose.yml
```

Update the `command:` section:
```yaml
command: >
  trade
  --logfile /freqtrade/user_data/logs/freqtrade.log
  --db-url sqlite:////freqtrade/user_data/tradesv3.sqlite
  --config /freqtrade/user_data/config.json
  --strategy SampleStrategy
```

####  Start the Bot
```bash
docker-compose up -d
```

####  Monitor Logs
```bash
docker ps
docker logs <container_id>
tail -f user_data/logs/freqtrade.log
```

---

### 📡 **Step 6: Secure Web UI (Optional)**

####  Enable API Access in `config.json`
```json
"api_server": {
  "enabled": true,
  "listen_ip_address": "127.0.0.1",
  "listen_port": 8080,
  "username": "admin",
  "password": "StrongPasswordHere"
}
```

####  Install & Configure Nginx
```bash
sudo apt install nginx -y
sudo ufw allow http/tcp
```

Create Proxy Config:
```bash
sudo nano /etc/nginx/conf.d/freq_proxy.conf
```

```nginx
server {
    listen 80;
    server_name your_domain_or_ip;

    location / {
        proxy_pass http://localhost:8080/;
    }
}
```

####  Secure with Self-Signed SSL (optional)
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/ssl/private/nginx-selfsigned.key \
-out /etc/ssl/certs/nginx-selfsigned.crt
sudo openssl dhparam -out /etc/nginx/dhparam.pem 4096
```

Configure Nginx:
```bash
# Add SSL snippets:
sudo nano /etc/nginx/snippets/self-signed.conf
ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

sudo nano /etc/nginx/snippets/ssl-params.conf
# Add strong SSL settings here (as given in your source)
```

Update Nginx config to use SSL:
```nginx
server {
    listen 443 ssl;
    include snippets/self-signed.conf;
    include snippets/ssl-params.conf;
    server_name your_domain_or_ip;

    location / {
        proxy_pass http://localhost:8080/;
    }
}

server {
    listen 80;
    return 301 https://$host$request_uri;
}
```

Reload:
```bash
sudo nginx -t && sudo nginx -s reload
sudo ufw allow https/tcp
```

---

###  **Step 7: Let’s Encrypt (For Public Domain)**

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com
```

---

###  **Optional: Run Freqtrade as a Systemd Service**

```bash
sudo nano /etc/systemd/system/freqtrade_docker.service
```

```ini
[Unit]
Description=Freqtrade Docker Bot
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=true
WorkingDirectory=/home/username/ft_userdata
ExecStart=/usr/local/bin/docker-compose up -d
ExecStop=/usr/local/bin/docker-compose down

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable freqtrade_docker
sudo systemctl start freqtrade_docker
```

---

Let me know if you'd like this exported as a PDF or markdown file.
