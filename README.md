# RaspPi_WebHost

Raspberry Pi 4 used as Web server host

- OS: Raspbian lite 64 bit
- Headless setup: Connected via tailscale SSH
- Currently hosts: [EduMeilleur](https://github.com/AdanRiasat/EduMeilleur) (Angular, ASP.NET Core, PostgreSQL)

## Architecture Overview
 
All services run as Docker containers managed by Docker Compose.
 
```
GitHub Actions
    │
    ├── Builds multi-platform images (linux/amd64 + linux/arm64)
    ├── Pushes to GHCR (ghcr.io/adanriasat/edumeilleur/*)
    └── SSHs into Pi via Tailscale → docker compose pull && up -d
                                            │
                                       Pi (Docker)
                              ┌────────────┴────────────┐
                         cloudflared              nginx:alpine
                         (Cloudflare               (port 80)
                          Tunnel)                      │
                                              ┌────────┴────────┐
                                          frontend            api
                                         (nginx:alpine)   (ASP.NET 8)
                                                               │
                                                           postgres:18
```

### How a Request Flows
 
1. User requests `https://edumeilleur.ca` → Cloudflare routes through the tunnel to the cloudflared container on the Pi
2. cloudflared forwards to the nginx container on port 80
3. Nginx serves Angular static files directly from the frontend container
4. Angular triggers API calls to `/api/`
5. Nginx proxies `/api/` to the ASP.NET Core api container on port 8080
6. API queries PostgreSQL, returns response back through the chain

### Docker containers

```
# Frontend

FROM node:18-alpine AS build

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build -- --configuration production

FROM nginx:alpine
COPY --from=build /app/dist/edu-meilleur/browser /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

```
# API

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

RUN dotnet tool install --global dotnet-ef
ENV PATH="$PATH:/root/.dotnet/tools"

COPY EduMeilleurAPI/EduMeilleurAPI.csproj EduMeilleurAPI/
RUN --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    dotnet restore EduMeilleurAPI/EduMeilleurAPI.csproj

COPY . .
WORKDIR /src/EduMeilleurAPI

ARG TARGETARCH
RUN dotnet publish -c Release -o /app/publish -r linux-$TARGETARCH --self-contained false

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "EduMeilleurAPI.dll"]
```

```
# compose.yaml

name: 'edumeilleur'

services:
  db:
    image: postgres:18-alpine
    env_file: '.env'
    environment:
      POSTGRES_USER: '${EDUMEILLEUR_POSTGRES_USER:?Add to .env}'
      POSTGRES_PASSWORD: '${EDUMEILLEUR_POSTGRES_PASSWORD:?Add to .env}'
      POSTGRES_DB: '${EDUMEILLEUR_POSTGRES_DATABASE:?Add to .env}'
    volumes:
      - pg-data:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${EDUMEILLEUR_POSTGRES_USER} -d ${EDUMEILLEUR_POSTGRES_DATABASE}"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped
  
  api:
    image: ghcr.io/adanriasat/edumeilleur/api:latest
    env_file: '.env'
    environment:
      ConnectionStrings__EduMeilleurAPIContext: "Host=${EDUMEILLEUR_POSTGRES_HOST};Port=5432;Database=${EDUMEILLEUR_POSTGRES_DATABASE};Username=${EDUMEILLEUR_POSTGRES_USER};Password=${EDUMEILLEUR_POSTGRES_PASSWORD}"
    depends_on:
      db:
        condition: service_healthy
    command: >
      sh -c "
        dotnet EduMeilleurAPI.dll
      "
    restart: unless-stopped

  frontend:
    image: ghcr.io/adanriasat/edumeilleur/frontend:latest
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports: 
      - 80:80
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro,Z
    depends_on:
      - api
      - frontend
    restart: unless-stopped

  cloudflared:
    image: cloudflare/cloudflared:latest
    command: tunnel run
    restart: unless-stopped
    depends_on:
      - nginx
    volumes:
      - ./cloudflared:/etc/cloudflared

volumes:
  pg-data:
```

## Nginx as Reverse Proxy

Nginx runs as a container and acts as a reverse proxy. Static files are served by the frontend container, not by Nginx.

```
# nginx.congf

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://frontend:80;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_intercept_errors on;
        error_page 404 = /index.html;
    }

    location /api/ {
        proxy_pass http://api:8080/api/;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

## Cloudflare Tunnel (Remote Access)

To make the application accessible over the internet without port forwarding, a Cloudflare Tunnel is used.

```
# cloudflared/config.yaml

tunnel: TUNNEL_ID
credentials-file: /etc/cloudflared/TUNNEL_ID.json

ingress:
  - hostname: yourdomain.com
    service: http://nginx:80

  - hostname: api.yourdomain.com
    service: http://nginx:80

  - service: http_status:404
```

To create the tunnel credentials for the first time:
```
cloudflared tunnel login
cloudflared tunnel create edumeilleur
cp ~/.cloudflared/<tunnel-id>.json ~/EduMeilleur/cloudflared/
```

## CI/CD (GitHub Actions)
 
On every push to `main`, the workflow:
 
1. Builds multi-platform Docker images (`linux/amd64` + `linux/arm64`) using QEMU and buildx
2. Pushes images to GHCR (`ghcr.io/adanriasat/edumeilleur/api:latest` and `frontend:latest`)
3. Connects to the Pi via Tailscale using an auth key
4. SSHs in and runs `docker compose pull && docker compose up -d`
The Pi currently requires the files in `~/EduMeilleur/`: compose, env, nginx config, cloudflared config.
 
Required GitHub secrets:
```
GITHUB_TOKEN       ← automatic
TS_AUTHKEY         ← Tailscale auth key (ephemeral)
PI_HOST            ← Pi's Tailscale IP (100.x.x.x)
PI_USER            ← SSH username
PI_SSH_KEY         ← private SSH key
```

## Boot Behaviour
 
All containers have `restart: unless-stopped`. The Docker daemon is enabled as a systemd service. On reboot, Docker starts automatically and brings all containers back up.

## TODO

- WIFI recovery with USB script