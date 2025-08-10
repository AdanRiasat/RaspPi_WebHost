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
scp -r dist/edu-meilleur/* adan@<pi-ip>:/var/www/angular/
```

### ASP.Net Core

publish on dev machine
```
dotnet publish -c Release -r linux-arm --self-contained false -o ./publish
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
(Optional) Create a systemd service to run the backend automatically
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
pg_dump -U <db_user> -h localhost -p 5432 <db_name> > edumeilleur_dump.sql
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

### How a Request Flows (local network)

1. User requests `http://<pi-ip>/` → Nginx serves Angular static files.
2. The user interacts with the frontend, triggering an API request that starts with `http://<pi-ip>/api/`.
3. Nginx forwards API request to ASP.NET Core backend at localhost:5000.
4. Backend processes request and returns data.
5. Nginx relays backend response to the client browser.

### TODO

- Configure for HTTPS
- Make accessible outside of local network
- Add extra security mesures with nginx