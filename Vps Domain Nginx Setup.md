# DevOps Setup Guide: Deploying Node.js App with NGINX and Domain SSL

This document outlines all the required steps to deploy and secure a Node.js application on a VPS, using NGINX, PM2, and a custom domain with SSL. This is intended for DevOps and system administrators.

---

## 1. VPS Environment Setup

### Access VPS

```bash
ssh root@<VPS_PUBLIC_IP>
```

### Install Dependencies

```bash
sudo apt update
sudo apt install nodejs npm nginx certbot python3-certbot-nginx ufw -y
sudo npm install -g pm2
```

---

## 2. Node.js Application Setup

### Upload or Clone Your App

Structure example:

```
/project-root
 ├── index.js
 ├── package.json
 ├── public/
 └── views/
```

### Install App Dependencies

```bash
npm install
```

### Configure App to Listen Locally

In `index.js` or similar:

```js
app.listen(3000, '127.0.0.1', () => {
  console.log("Server running on localhost:3000");
});
```

### Start App with PM2

```bash
pm2 start index.js --name yourdomine-app
pm2 save
pm2 startup
```

---

## 3. NGINX Setup as Reverse Proxy

### Create NGINX Config

File: `/etc/nginx/sites-available/yourdomine`

```nginx
server {
    listen 80;
    server_name yourdomine www.yourdomine;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### Enable the Config

```bash
sudo ln -s /etc/nginx/sites-available/yourdomine /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 4. Domain Setup (DNS)

Go to your domain provider (Hostinger, GoDaddy, etc.), and add:

```
A Record:
@     ->  <VPS_PUBLIC_IP>
www   ->  <VPS_PUBLIC_IP>
```

Wait for DNS propagation (can take up to 30 mins). Check using:

```bash
dig yourdomine +short
dig www.yourdomine +short
```

---

## 5. Enable SSL with Let's Encrypt

```bash
sudo certbot --nginx
# Follow prompts: enter email, agree, select both yourdomine and www.yourdomine
```

After success, certbot will update your NGINX config to enable HTTPS.

### Test Renewal

```bash
sudo certbot renew --dry-run
```

---

## 6. Security & Port Restrictions

### UFW Firewall

```bash
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

### Block Direct Access to Port 3000

If needed:

```bash
sudo ufw deny 3000
```

---

## 7. Optional: Redirect Non-WWW to WWW or Vice Versa

To redirect all `yourdomine` to `www.yourdomine`, add this block:

```nginx
server {
    listen 80;
    server_name yourdomine;
    return 301 http://www.yourdomine$request_uri;
}
```

---

## 8. Testing

* Visit [https://yourdomine](https://yourdomine)
* Ensure PM2 shows app online:

```bash
pm2 list
```

* Check NGINX status:

```bash
sudo systemctl status nginx
```

---

## 9. Notes

* Node.js app should never bind to `0.0.0.0` unless needed.
* NGINX handles all external access, including HTTPS.
* Certbot auto-renews certificates; no manual renewal required.

---

## 10. CI/CD (Optional Future Integration)

To integrate continuous deployment:

* Use GitHub Actions or GitLab CI
* SSH deploy key setup to access VPS
* Use `scp` or `rsync` to push code
* Restart app via PM2:

```bash
pm2 restart yourdomine-app
```

---

## 11. Troubleshooting

### NGINX fails to reload:

```bash
sudo nginx -t
sudo systemctl status nginx
journalctl -xe
```

### App not reachable via domain:

* Check DNS propagation using `dig`
* Ensure NGINX is listening on port 80/443
* Check firewall rules with `sudo ufw status`

---

## Maintainer

* Email: [mohanbabi16@gmail.com](mailto:mohanbabi16@gmail.com)
* Last Updated: June 5, 2025
