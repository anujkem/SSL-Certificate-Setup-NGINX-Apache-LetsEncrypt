# SSL-Certificate-Setup-NGINX-Apache-LetsEncrypt


SSL Setup

Description:
This repository provides step-by-step instructions to set up SSL certificates for web servers using Let's Encrypt with NGINX and Apache2.

---

Prerequisites:
- A domain name (e.g., yourdomain.com).
- A server running Ubuntu.
- Access to your server's terminal.

---

Steps:

1. Allow HTTP and HTTPS through Firewall (UFW):

    sudo ufw allow 80
    sudo ufw allow 443

2. Install Let's Encrypt:

    sudo apt install letsencrypt

---

NGINX Setup:

1. Install Certbot for NGINX:

    sudo apt install python3-certbot-nginx

2. Generate SSL Certificate for NGINX:

    Replace domain-name.com with your actual domain name.

    sudo certbot --nginx --agree-tos --preferred-challenges http -d domain-name.com

---

Apache2 Setup:

1. Install Certbot for Apache2:

    sudo apt install python3-certbot-apache

2. Generate SSL Certificate for Apache2:

    Replace domain-name.com with your actual domain name.

    sudo certbot --apache --agree-tos --preferred-challenges http -d domain-name.com

---

Additional Notes:
- Ensure your domain is pointing to your server's IP address.
- Certbot will automatically renew your SSL certificate before it expires.
- To check the renewal process, you can run:

    sudo certbot renew --dry-run

---

License:
This repository is licensed under the MIT License.
