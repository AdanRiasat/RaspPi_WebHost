# RaspPi_WebHost

Raspberry Pi 4 used as Web server host

- OS: Debian
- Headless setup: Connected via SSH from dev machine
- Currently hosts: [EduMeilleur](https://github.com/AdanRiasat/EduMeilleur) (Angular, ASP.NET Core, PostgreSQL)

## Deployment Workflow EduMeilleur

### Angular

Build on dev machine
```
ng build --configuration production
```

Copy the `dist/` files to the Raspberry Pi via SCP
```
scp -r dist/edu-meilleur/* adan@<pi-ip>:/home/adan/angular_tmp
```

Move the temporary files to the correct directory
```
sudo mv ~/angular_tmp/*  /var/www/angular/
```

Give permissions to the angular files so nginx can access them
```
 sudo chown -R www-data:www-data /var/www/angular
```

### ASP.Net Core

publish on dev machine
```
dotnet publish -c Release -r linux-arm64 --self-contained false -o ./publish
```

Copy the published files to the Raspberry Pi via SCP
```
scp -r ./publish/* adan@<pi-ip>:/home/adan/eduMeilleurApi
```

On the Raspberry Pi, run the backend:
```
cd /home/adan/eduMeilleurApi/
dotnet EduMeilleurAPI.dll
```
Create a systemd service to run the backend automatically
```
[Unit]
Description=EduMeilleur ASP.NET Core Web API
After=network.target

[Service]
WorkingDirectory=/home/adan/EduMeilleur
ExecStart=/usr/bin/dotnet /home/adan/EduMeilleur/EduMeilleurAPI.dll
Restart=always
RestartSec=10
SyslogIdentifier=EduMeilleurAPI
User=adan
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false
Environment=ASPNETCORE_URLS=http://0.0.0.0:5000

[Install]
WantedBy=multi-user.target
```

### PostgreSQL

Update the database on dev machine
```
dotnet ef database update
```

Export the local database
```
pg_dump -U postgres -h localhost -p 5432 -d EduMeilleurAPIContext --encoding=UTF8 -f edumeilleur_dump.sql
```

Copy the dump file to the Raspberry Pi via SCP
```
scp edumeilleur_dump.sql adan@<pi-ip>:/home/adan/
```

On the Raspberry Pi, create the database
```
sudo -u postgres createdb <db_name>
```

Once created, import the dump file
```
sudo -u postgres psql -d <db_name> < /home/adan/edumeilleur_dump.sql
```

### Nginx as Reverse Proxy & Static Server

Nginx listens on port 80 and serves static Angular files from `/var/www/angular`. It proxies API requests to the backend running on `http://localhost:5000`.

Create a configuration file on the Raspberry Pi
```
sudo nano /etc/nginx/sites-available/edumeilleur
```
```
server {
    listen 80;
    server_name _;

    root /var/www/angular/browser;
    index index.html;

    location / {
        try_files $uri /index.html;
    }

    location /api {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Reload Nginx
```
sudo systemctl reload nginx
```

### Cloudflare Tunnel (Remote Access)

To make the application accessible over the internet without port forwarding, a Cloudflare Tunnel is used.

Authenticate and create a tunnel
```
cloudflared tunnel login
cloudflared tunnel create edumeilleur
```

Create a config file at `/home/adan/.cloudflared/config.yml`
```
tunnel: 44bdd7a4-e7c4-42f2-9a93-ec6e26f2b050
credentials-file: /home/adan/.cloudflared/44bdd7a4-e7c4-42f2-9a93-ec6e26f2b050.json

ingress:
  - hostname: edumeilleur.ca
    service: http://localhost:80
  - hostname: www.edumeilleur.ca
    service: http://localhost:80
  - hostname: app.edumeilleur.ca
    service: http://localhost:80
  - service: http_status:404
```

Create a systemd service to run the cloudflared tunnel automatically
```
[Unit]
Description=Cloudflare Tunnel
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/cloudflared tunnel run
Restart=always
User=adan
Environment=HOME=/home/adan
WorkingDirectory=/home/adan/.cloudflared

[Install]
WantedBy=multi-user.target
```

Enable and start the service
```
sudo systemctl daemon-reload
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

### How a Request Flows 

1. User requests `https://edumeilleur.ca/` → Cloudflare Tunnel fowards request to Raspberry Pi and onto Nginx.
2. Nginx serves Angular static files from `/var/www/angular`.
3. Angular frontend triggers API requests to `/api`.
4. Nginx proxies API requests to ASP.NET Core backend at `http://localhost:5000`.
5. Backend processes request and returns data.
6. Nginx relays backend response → Tunnel → Cloudflare → Client browser.

### TODO

- Bash script for updating files.
- Add workflow for updating database during production.