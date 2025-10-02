# Lobe Chat on AWS (Hobby Project)

This guide documents how we deployed **Lobe Chat** on an AWS EC2 instance using Docker and secured it with Nginx + Basic Auth. It also includes common tasks and troubleshooting tips.

---

## 🚀 Setup Steps

### 1. Launch EC2
- Use **Amazon Linux 2023** (free tier eligible).
- Open **port 22 (SSH)** and **port 80 (HTTP)** in the EC2 security group.
- Download the `.pem` key for SSH.

### 2. SSH Into Server
```bash
ssh -i your-key.pem ec2-user@your-ec2-public-ip
```

### 3. Install Docker
```bash
sudo yum update -y
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
```

### 4. Run Lobe Chat
```bash
docker run -d   --name lobe-chat   -p 127.0.0.1:3210:3210   -e OPENAI_API_KEY=your_api_key_here   lobehub/lobe-chat
```
👉 Note: We bind to **localhost only** since Nginx will handle public access.

### 5. Install Nginx
```bash
sudo yum install -y nginx httpd-tools
```

### 6. Add Basic Auth
Create username/password (`weebs:weebs` in this example):
```bash
printf "weebs\nweebs\n" | sudo htpasswd -ci /etc/nginx/.htpasswd weebs
```

### 7. Configure Nginx Reverse Proxy
Create `/etc/nginx/conf.d/lobe.conf`:
```nginx
server {
    listen 80;

    location / {
        proxy_pass http://127.0.0.1:3210;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;

        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
```

Test and restart Nginx:
```bash
sudo nginx -t
sudo systemctl restart nginx
```

Now visit your EC2 public IP — you’ll be prompted for username/password.

---

## 🔒 Authentication
- Uses **Nginx HTTP Basic Auth**.
- All users share the same login (hobby setup).
- Update password with:
```bash
sudo htpasswd /etc/nginx/.htpasswd weebs
```

---

## 💾 Persistence
By default, chat history is in the container (lost on restart).  
To persist:

```bash
docker run -d   --name lobe-chat   -p 127.0.0.1:3210:3210   -e OPENAI_API_KEY=your_api_key_here   -v /home/ec2-user/lobe-data:/app/data   lobehub/lobe-chat
```

---

## 🛠 Common Tasks

- **Check running containers**
  ```bash
  docker ps
  ```

- **Restart Lobe**
  ```bash
  docker restart lobe-chat
  ```

- **Update Lobe**
  ```bash
  docker stop lobe-chat
  docker rm lobe-chat
  docker pull lobehub/lobe-chat
  # then rerun with same command
  ```

- **Check logs**
  ```bash
  docker logs -f lobe-chat
  ```

- **Restart Nginx**
  ```bash
  sudo systemctl restart nginx
  ```

---

## ⚠️ Common Issues

- **Port 80 already in use**  
  Make sure Docker is bound only to `127.0.0.1:3210`, not port 80 directly.

- **Auth not prompting**  
  Check `/etc/nginx/conf.d/lobe.conf` for `auth_basic` lines.

- **Container won’t start**  
  Verify your `OPENAI_API_KEY` is valid and Docker is running.

---

## 🌱 Notes
- This is a hobby project (single EC2, no HTTPS).  
- For production: use HTTPS (Certbot/Let’s Encrypt) and individual accounts.  
- EC2 free tier covers **750 hours/month for t2.micro/t3.micro**.
