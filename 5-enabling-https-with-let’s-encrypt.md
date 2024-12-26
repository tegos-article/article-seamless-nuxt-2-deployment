### 5. Enabling HTTPS with Let’s Encrypt

Secure your application with HTTPS by setting up Let’s Encrypt:

1. Install Certbot:
   ```bash
   apt install certbot python3-certbot-nginx -y
   ```
2. Obtain an SSL certificate:
   ```bash
   certbot --nginx -d your_domain
   ```
3. Test the renewal process:
   ```bash
   certbot renew --dry-run
   ```
4. Reload Nginx:
   ```bash
   systemctl reload nginx
   ```