# Deploying a Flask Application with Nginx and Let's Encrypt SSL on Ubuntu

This guide provides a refined setup for deploying a Flask application on an Ubuntu server using Nginx as a reverse proxy and Let's Encrypt for SSL certificates. It addresses common errors such as Nginx configuration issues, Certbot failures, and static file access problems.

## Prerequisites
- Ubuntu server (e.g., Ubuntu 24.04)
- Python 3.12 and `venv` installed
- Domain names configured (e.g., `i3chatbot.duckdns.org`, `chatbottest.duckdns.org`) pointing to your server's IP
- Root access or `sudo` privileges

## Flask Application (`app.py`)

The Flask application below handles a chatbot interface with session management and CORS support. It remains unchanged from the original but is included for completeness.

```python

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=5000)
```

**Note**: The `app.run()` is for development only. Use Gunicorn for production, as described below.

## Nginx Configuration

The Nginx configuration below sets up a reverse proxy to the Flask app, serves static files, and enables HTTPS with Let's Encrypt certificates.

**File: `/etc/nginx/sites-available/chatbot`**

```nginx
server {
    listen 80;
    server_name i3chatbot.duckdns.org chatbottest.duckdns.org;
    return 301 https://$host$request_uri; # Redirect HTTP to HTTPS
}

server {
    listen 443 ssl;
    server_name i3chatbot.duckdns.org chatbottest.duckdns.org;

    ssl_certificate /etc/letsencrypt/live/i3chatbot.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/i3chatbot.duckdns.org/privkey.pem;
    ssl_certificate /etc/letsencrypt/live/chatbottest.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/chatbottest.duckdns.org/privkey.pem;

    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        alias /root/test_chat_bot/static/;
    }
}
```

**Explanation**:
- **Port 80**: Redirects all HTTP traffic to HTTPS.
- **Port 443**: Handles HTTPS traffic with Let's Encrypt certificates for both domains.
- **ACME Challenge**: Serves Certbot's challenge files for certificate renewal.
- **Proxy**: Forwards requests to the Flask app running on `127.0.0.1:5000`.
- **Static Files**: Serves static files directly from `/root/test_chat_bot/static/`.

## Setup Instructions

Follow these steps to deploy the Flask application without errors:

### 1. Install Dependencies
Ensure Nginx and Certbot are installed:
```bash
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx
```

Install Gunicorn in your virtual environment:
```bash
cd ~/test_chat_bot
source venv/bin/activate
pip install gunicorn
```

### 2. Configure Nginx
Replace the existing Nginx configuration:
```bash
sudo nano /etc/nginx/sites-available/chatbot
```
Copy and paste the Nginx configuration above, then save and exit.

Enable the configuration:
```bash
sudo ln -sf /etc/nginx/sites-available/chatbot /etc/nginx/sites-enabled/chatbot
```

Test and reload Nginx:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 3. Run Flask with Gunicorn
Instead of `python app.py`, use Gunicorn:
```bash
cd ~/test_chat_bot
source venv/bin/activate
gunicorn --bind 0.0.0.0:5000 app:app
```

### 4. Set Up Gunicorn as a Systemd Service
Create a systemd service to run Gunicorn persistently:

**File: `/etc/systemd/system/gunicorn.service`**
```bash
sudo nano /etc/systemd/system/gunicorn.service
```
Add:
```ini
[Unit]
Description=Gunicorn instance to serve Flask app
After=network.target

[Service]
User=root
Group=www-data
WorkingDirectory=/root/test_chat_bot
Environment="PATH=/root/test_chat_bot/venv/bin"
ExecStart=/root/test_chat_bot/venv/bin/gunicorn --bind 0.0.0.0:5000 app:app

[Install]
WantedBy=multi-user.target
```

Enable and start the service:
```bash
sudo systemctl enable gunicorn
sudo systemctl start gunicorn
```

### 5. Obtain SSL Certificates
Run Certbot to obtain certificates for both domains:
```bash
sudo certbot --nginx -d i3chatbot.duckdns.org -d chatbottest.duckdns.org
```
Follow the prompts to provide an email and agree to the terms. Certbot will configure SSL automatically.

### 6. Fix DNS Configuration
Ensure your domains point to your server's IP (`178.128.108.121`):
```bash
nslookup i3chatbot.duckdns.org
nslookup chatbottest.duckdns.org
```
Update DNS records at DuckDNS if necessary.

### 7. Fix Static File Issues
The log showed a 404 error for a Google Fonts URL. Check `index.html` or `chatbot.js` for malformed URLs (e.g., `https:/` instead of `https://`):
```html
<!-- Example in index.html -->
<link href="https://fonts.googleapis.com/css2?family=Inter:opsz,wght@14..32,100..900&display=swap" rel="stylesheet">
```

Ensure static files are accessible:
```bash
sudo chown -R www-data:www-data /root/test_chat_bot/static
sudo chmod -R 755 /root/test_chat_bot/static
```

### 8. Verify the Setup
- Check Nginx status:
  ```bash
  sudo systemctl status nginx
  ```
- Check Gunicorn status:
  ```bash
  sudo systemctl status gunicorn
  ```
- Test the application at `https://i3chatbot.duckdns.org` and `https://chatbottest.duckdns.org`.

### 9. Firewall Configuration
Allow HTTP and HTTPS traffic:
```bash
sudo ufw allow 80
sudo ufw allow 443
```

## Troubleshooting

- **Nginx Syntax Error**: Verify the configuration with `sudo nginx -t`. Check for stray characters or incorrect indentation.
- **Certbot 404 Error**: Ensure `/var/www/certbot/.well-known/acme-challenge/` is accessible and DNS points to the correct IP.
- **Flask Not Accessible**: Verify Gunicorn is running (`ss -tulpn | grep 5000`) and Nginx is proxying correctly.
- **Static File 404**: Confirm the `static` directory path and permissions.
- **Certificate Renewal**: Test renewal with `sudo certbot renew --dry-run`.

## Security Recommendations
- **Non-Root User**: Avoid running Gunicorn as `root`:
  ```bash
  sudo adduser --system --group --no-create-home flaskuser
  sudo chown -R flaskuser:flaskuser /root/test_chat_bot
  ```
  Update `gunicorn.service` to use `User=flaskuser` and `Group=flaskuser`.
- **Environment Variables**: Store `SECRET_KEY` in an environment variable instead of hardcoding.

## Conclusion
This setup ensures a secure, production-ready Flask application with Nginx and Let's Encrypt SSL. If you encounter errors, share the specific logs or error messages for further assistance.
```
