Here’s a complete docker-compose.yml setup with:

n8n

PostgreSQL

NGINX reverse proxy with HTTPS

🧩 Folder Structure
~/n8n-docker/
├── docker-compose.yml
├── n8n_data/
└── nginx/
    └── conf.d/

🐳 1. docker-compose.yml
version: "3.8"

services:
  postgres:
    image: postgres:15
    container_name: n8n-postgres
    restart: always
    environment:
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=n8npassword
      - POSTGRES_DB=n8n
    volumes:
      - ./postgres_data:/var/lib/postgresql/data

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: always
    depends_on:
      - postgres
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=n8npassword

      # Base URLs and domain config
      - N8N_HOST=automation.example.com
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://automation.example.com/
      - N8N_EDITOR_BASE_URL=https://automation.example.com/
      - NODE_ENV=production

    volumes:
      - ./n8n_data:/home/node/.n8n
    expose:
      - "5678"

  nginx:
    image: nginx:latest
    container_name: n8n-nginx
    restart: always
    depends_on:
      - n8n
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - /etc/letsencrypt:/etc/letsencrypt
      - ./nginx/logs:/var/log/nginx

🧱 2. NGINX Configuration

Create file:
nginx/conf.d/n8n.conf
```
server {
    listen 80;
    server_name automation.example.com;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name automation.example.com;

    ssl_certificate /etc/letsencrypt/live/automation.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/automation.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://n8n:5678/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    access_log /var/log/nginx/n8n.access.log;
    error_log /var/log/nginx/n8n.error.log;
}
```
🔒 3. Get SSL Certificates

Use Certbot on the host (not in Docker):

```
sudo apt install certbot -y
sudo certbot certonly --standalone -d automation.example.com
```

Certificates will be saved to /etc/letsencrypt/live/automation.example.com/.

🚀 4. Start Everything
docker-compose up -d


Check logs:

docker logs -f n8n


You should see it connecting to PostgreSQL:

Migrations complete
n8n ready on port 5678


Then visit:
👉 https://automation.example.com
