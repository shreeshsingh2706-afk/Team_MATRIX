# 🚀 Deployment Guide — FitForge Gym (Team_MATRIX)

> **Assignment:** Deployment of Project | 10 Points | Prashant Singh · Apr 7

---

## ✅ Task 1 — Claim Free Cloud Credits (AWS)

### Step 1: Create an AWS Free Tier Account
1. Go to 👉 https://aws.amazon.com/free/
2. Click **"Create a Free Account"**
3. Enter your **email**, create a **password**, and give your AWS account a name
4. Enter your **personal/billing details** (credit/debit card required — you won't be charged for Free Tier usage)
5. Verify your phone number via OTP
6. Choose the **Basic (Free) support plan**
7. Sign in to the **AWS Management Console**

### What you get FREE for 12 months:
| Service | Free Tier Limit |
|--------|----------------|
| EC2 (t2.micro) | 750 hours/month |
| S3 Storage | 5 GB |
| RDS (MySQL) | 750 hours/month |
| Data Transfer | 15 GB outbound |

---

## ✅ Task 2 — Launch EC2 & Install Components

### Step 1: Launch an EC2 Instance
1. Open AWS Console → **EC2** → **Launch Instance**
2. Name it: `fitforge-server`
3. Choose **Ubuntu Server 22.04 LTS** (Free tier eligible)
4. Instance type: **t2.micro** (Free tier)
5. Create a **new key pair** → Download the `.pem` file (save it safely!)
6. Under **Security Group**, allow:
   - SSH (port 22)
   - HTTP (port 80)
   - HTTPS (port 443)
   - Custom TCP (port 8080) — for Tomcat
7. Click **Launch Instance**

### Step 2: Connect to Your Server
```bash
# On your local machine (Mac/Linux Terminal)
chmod 400 your-key.pem
ssh -i "your-key.pem" ubuntu@<YOUR_EC2_PUBLIC_IP>
```

### Step 3: Update the Server
```bash
sudo apt update && sudo apt upgrade -y
```

### Step 4: Install Java 21
```bash
# Install Java 21
sudo apt install -y openjdk-21-jdk

# Verify installation
java -version
# Expected: openjdk version "21.x.x"
```

### Step 5: Install Tomcat 10
```bash
# Create a dedicated tomcat user
sudo useradd -m -d /opt/tomcat -U -s /bin/false tomcat

# Download Tomcat 10
cd /tmp
wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.20/bin/apache-tomcat-10.1.20.tar.gz

# Extract and move
sudo tar xzvf apache-tomcat-10.1.20.tar.gz -C /opt/tomcat --strip-components=1

# Set permissions
sudo chown -R tomcat:tomcat /opt/tomcat/
sudo chmod -R u+x /opt/tomcat/bin

# Verify
/opt/tomcat/bin/version.sh
```

### Step 6: Install Nginx (Web Server)
```bash
sudo apt install -y nginx

# Start and enable Nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# Verify — open http://<YOUR_EC2_IP> in browser, you should see the Nginx welcome page
sudo systemctl status nginx
```

### Step 7: Install MySQL
```bash
sudo apt install -y mysql-server

# Start and enable MySQL
sudo systemctl start mysql
sudo systemctl enable mysql

# Secure the installation
sudo mysql_secure_installation
# Follow prompts: set root password, remove anonymous users, etc.

# Verify
sudo mysql -u root -p
# Type: SHOW DATABASES; then EXIT;
```

---

## ✅ Task 3 — Configuration of All Components

### 🔧 Configure Tomcat as a System Service
```bash
# Find JAVA_HOME
sudo update-java-alternatives -l
# Note the path, usually: /usr/lib/jvm/java-21-openjdk-amd64

# Create systemd service file
sudo nano /etc/systemd/system/tomcat.service
```

Paste this into the file:
```ini
[Unit]
Description=Apache Tomcat 10
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat
Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64"
Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/opt/tomcat"
ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start Tomcat
sudo systemctl daemon-reload
sudo systemctl start tomcat
sudo systemctl enable tomcat
sudo systemctl status tomcat

# Tomcat is now running on port 8080
# Visit: http://<YOUR_EC2_IP>:8080
```

### 🔧 Configure Nginx to Serve FitForge Frontend

#### Deploy the website files:
```bash
# Create website directory
sudo mkdir -p /var/www/fitforge

# Upload your project files (run this on YOUR LOCAL machine)
scp -i "your-key.pem" -r /path/to/Team_MATRIX/* ubuntu@<YOUR_EC2_IP>:/var/www/fitforge/

# On the server — fix permissions
sudo chown -R www-data:www-data /var/www/fitforge
sudo chmod -R 755 /var/www/fitforge
```

#### Configure Nginx site:
```bash
sudo nano /etc/nginx/sites-available/fitforge
```

Paste this configuration:
```nginx
server {
    listen 80;
    server_name <YOUR_EC2_IP>;   # Replace with your IP or domain

    root /var/www/fitforge;
    index contact_us.html index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    # Proxy requests to Tomcat (for future backend use)
    location /app/ {
        proxy_pass http://localhost:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Logging
    access_log /var/log/nginx/fitforge_access.log;
    error_log /var/log/nginx/fitforge_error.log;
}
```

```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/fitforge /etc/nginx/sites-enabled/

# Remove default site
sudo rm /etc/nginx/sites-enabled/default

# Test config syntax
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx

# ✅ Your website is now live at: http://<YOUR_EC2_IP>
```

### 🔧 Configure MySQL (Create a database for future use)
```bash
sudo mysql -u root -p
```

```sql
-- Create a database for FitForge
CREATE DATABASE fitforge_db;

-- Create a dedicated user
CREATE USER 'fitforge_user'@'localhost' IDENTIFIED BY 'StrongPassword@123';

-- Grant permissions
GRANT ALL PRIVILEGES ON fitforge_db.* TO 'fitforge_user'@'localhost';
FLUSH PRIVILEGES;

-- Verify
SHOW DATABASES;
EXIT;
```

---

## ✅ Task 4 (Optional) — Enable SSL / HTTPS with a Domain

> **Prerequisites:** You must have a domain name pointed to your EC2 IP via DNS A record.

### Option A: Free Domain (if you don't have one)
- Get a free domain at 👉 https://www.freenom.com or https://freedns.afraid.org
- Point it to your EC2 public IP

### Step 1: Install Certbot (Let's Encrypt — FREE SSL)
```bash
sudo apt install -y certbot python3-certbot-nginx

# Generate SSL certificate
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Follow prompts: enter email, agree to ToS, choose redirect (option 2)
```

### Step 2: Verify Auto-Renewal
```bash
sudo certbot renew --dry-run
# Certificates auto-renew every 90 days via cron
```

### Step 3: Verify HTTPS
```
https://yourdomain.com  ✅ Secure padlock should appear!
```

---

## 📋 Quick Reference — All Services

| Component | Port | Status Check Command |
|-----------|------|----------------------|
| Nginx | 80/443 | `sudo systemctl status nginx` |
| Tomcat | 8080 | `sudo systemctl status tomcat` |
| MySQL | 3306 | `sudo systemctl status mysql` |

---

## 🔥 Firewall Rules Summary (AWS Security Group)

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| SSH | TCP | 22 | Your IP only |
| HTTP | TCP | 80 | 0.0.0.0/0 |
| HTTPS | TCP | 443 | 0.0.0.0/0 |
| Custom | TCP | 8080 | 0.0.0.0/0 |
| MySQL | TCP | 3306 | This server only |

---

*Deployment guide created for Team_MATRIX | FitForge Gym Project*
