# Deploying a Flask Application on DigitalOcean

This guide will walk you through the process of deploying a Flask application on a DigitalOcean Droplet using **Gunicorn** as the WSGI server and **Nginx** as the reverse proxy. It also includes optional PostgreSQL setup and HTTPS configuration.

---

## 1. Connect to Your Droplet

Open a terminal and connect to your server via SSH:

```bash
ssh username@203.0.113.0
```

- Replace `username` with your Droplet username (default: `root`).
- Replace `203.0.113.0` with your Droplet’s IP address.
- If prompted, type `yes` to confirm. If not using SSH keys, enter your password when prompted.

---

## 2. Prepare the Flask Application

Navigate to your application directory:

```bash
cd ~/myapps/weather/
```

Edit your environment file:

```bash
vim .env
```

To save and exit in `vim`, press `ESC` then type:

```
:wq
```

Verify your `.env` file:

```bash
cat .env
```

Activate the virtual environment:

```bash
source env/bin/activate
```

Install dependencies:

```bash
pip install -r weather_app/requirements.txt
```

If not already cloned, clone your application:

```bash
git clone https://github.com/CaptainBett/weather_app.git
```

---

## 3. Configure the Firewall

Enable and configure the firewall:

```bash
sudo ufw enable
sudo ufw allow 5000
sudo ufw allow ssh
sudo ufw reload
sudo ufw status
```

---

## 4. Set Up Gunicorn

Create a WSGI entry file:

```bash
touch wsgi.py
sudo nano wsgi.py
```

Add the following:

```python
from app import app

if __name__ == "__main__":
    app.run()
```

Test Gunicorn:

```bash
gunicorn --bind 0.0.0.0:5000 wsgi:app
```

Deactivate your environment when done:

```bash
deactivate
```

### Create a systemd Service

Create a service file:

```bash
sudo nano /etc/systemd/system/weather_app.service
```

Add:

```ini
[Unit]
Description=Gunicorn instance to serve weather app
After=network.target

[Service]
User=bett
Group=www-data
WorkingDirectory=/home/bett/myapps/weather_app
Environment="PATH=/home/bett/myapps/weather_app/env/bin"
ExecStart=/home/bett/myapps/weather_app/env/bin/gunicorn --workers 3 --bind unix:/home/bett/myapps/weather_app/weather_app.sock -m 007 wsgi:app

[Install]
WantedBy=multi-user.target
```

Reload systemd and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl start weather_app
sudo systemctl enable weather_app
sudo systemctl status weather_app
```

---

## 5. Set Up Nginx

Create a new configuration file:

```bash
sudo nano /etc/nginx/sites-available/weather_app.conf
```

Add:

```nginx
server {
    server_name captaincodes.co.ke www.captaincodes.co.ke;
    listen 80;

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/bett/myapps/weather_app/weather_app.sock;
    }
}
```

Enable the config:

```bash
sudo ln -s /etc/nginx/sites-available/weather_app.conf /etc/nginx/sites-enabled/
```

Test and restart Nginx:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

Update firewall:

```bash
sudo ufw delete allow 5000
sudo ufw allow "Nginx Full"
sudo ufw status
```

Check Nginx:

```bash
sudo systemctl status nginx
```

---

## 6. Configure PostgreSQL (Optional)

Install PostgreSQL:

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
```

Log in as the postgres user:

```bash
sudo -i -u postgres
```

Create a database and user:

```sql
CREATE DATABASE weather_db;
CREATE USER weather_user WITH PASSWORD 'secure_password';
GRANT ALL PRIVILEGES ON DATABASE weather_db TO weather_user;
```

Update Flask config:

```python
SQLALCHEMY_DATABASE_URI = 'postgresql://weather_user:secure_password@localhost/weather_db'
```

---

## 7. Update Domain Nameservers

At your domain registrar (e.g., Truehost), update nameservers to:

```
ns1.digitalocean.com
ns2.digitalocean.com
ns3.digitalocean.com
```

Check propagation (24–48 hrs):

```bash
whois captaincodes.co.ke | grep "Name Server"
```

Expected output:

```
Name Server: ns1.digitalocean.com
Name Server: ns2.digitalocean.com
Name Server: ns3.digitalocean.com
```

---

## 8. Troubleshooting Logs

Check Nginx logs:

```bash
sudo tail /var/log/nginx/error.log
sudo tail -f /var/log/nginx/error.log
```

Check Gunicorn logs:

```bash
sudo systemctl status weather_app.service
```

Restart services if needed:

```bash
sudo systemctl restart nginx
```

---

## 9. Security and HTTPS (Recommended)

Make sure your `.env` file is **not** committed to GitHub.

Enable HTTPS with Let’s Encrypt:

```bash
sudo apt update && sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d captaincodes.co.ke -d www.captaincodes.co.ke
sudo systemctl status snap.certbot.renew.service
sudo certbot renew --dry-run
```

---

## 10. PostgreSQL Role Attributes (Reference)

- **Superuser**: Can override all restrictions.
- **Create role**: Can create users.
- **Create DB**: Can create databases.
- **Replication**: Can run replication and backups.
- **Bypass RLS**: Can bypass row-level security.

Example superuser role:

```sql
CREATE ROLE captain WITH SUPERUSER CREATEROLE CREATEDB REPLICATION BYPASSRLS LOGIN PASSWORD 'your_password_here';
```

