# arr-stack

A self-hosted media ecosystem that combines media management and streaming.

This stack includes:

- **Radarr:** For movie management
- **Sonarr:** For TV show management
- **Prowlarr:** A torrent indexer manager for Radarr/Sonarr
- **qBittorrent:** Torrent client for downloading media
- **Jellyseerr:** To manage media requests
- **Jellyfin:** Open-source media streamer

## Requirements

- Docker version 28.0.1 or later
- Docker compose version v2.33.1 or later
- Older versions may work, but they have not been tested.

## Install media stack

> **⚠️ Warning for ARMv7 Users:**  
> Jellyseerr **v2.0.x** introduces breaking changes, dropping support for the **ARMv7** container image. However, **ARM64** support remains available. You will not be able to run Jellyseerr in ARMv7 CPU.

Before deploying the stack, you must first create a Docker network:  

```bash
docker network create --subnet 172.20.0.0/16 mynetwork
# Update the CIDR range based on your available IP range
```

### Create External Volumes

Create external volumes that will persist independently of the containers. These volumes will bind to your host directories for media storage:

```bash
# Create external volume for torrent downloads
docker volume create --driver local --opt type=none --opt o=bind --opt device=/path/to/your/torrent/directory windows-torrent-downloads

# Create external volume for media library 
docker volume create --driver local --opt type=none --opt o=bind --opt device=/path/to/your/media/library windows-library
```

**Windows Example:**
```bash
docker volume create --driver local --opt type=none --opt o=bind --opt device=D:\sebas\Torrent windows-torrent-downloads
docker volume create --driver local --opt type=none --opt o=bind --opt device=D:\sebas\Videos\Jellyfin windows-library
```

**Linux Example:**
```bash
docker volume create --driver local --opt type=none --opt o=bind --opt device=/mnt/torrents windows-torrent-downloads  
docker volume create --driver local --opt type=none --opt o=bind --opt device=/mnt/media windows-library
```

> **Note:** Replace the device paths with your actual directory paths where you want to store torrents and media files.


```bash
docker compose -d

# OPTIONAL: Use Nginx as a reverse proxy
# docker compose -f docker-compose-nginx.yml up -d
```

## Configure qBittorrent

- Open qBitTorrent at http://localhost:5080. Default username is `admin`. Temporary password can be collected from container log `docker logs qbittorrent`
- Go to Tools --> Options --> WebUI --> Change password
- Run below commands on the server

```bash
docker exec -it qbittorrent bash # Get inside qBittorrent container

# Above command will get you inside qBittorrent interactive terminal, Run below command in qbt terminal
mkdir /downloads/movies /downloads/tvshows
chown 1000:1000 /downloads/movies /downloads/tvshows
```

## Configure Radarr

- Open Radarr at http://localhost:7878
- Settings --> Media Management --> Check mark "Movies deleted from disk are automatically unmonitored in Radarr" under File management section --> Save
- Settings --> Media Management --> Scroll to bottom --> Add Root Folder --> Browse to /downloads/movies --> OK
- Settings --> Download clients --> qBittorrent --> Add Host (qbittorrent) and port (5080) --> Username and password --> Test --> Save **Note: If VPN is enabled, then qbittorrent is reachable on vpn's service name. In this case use `vpn` in Host field.**
- Settings --> General --> Enable advance setting --> Select Authentication and add username and password
- Indexer will get automatically added during configuration of Prowlarr. See 'Configure Prowlarr' section.

Sonarr can also be configured in similar way.

**Add a movie** (After Prowlarr is configured)

- Movies --> Search for a movie --> Add Root folder (/downloads/movies) --> Quality profile --> Add movie
- All queued movies download can be checked here, Activities --> Queue 
- Go to qBittorrent (http://localhost:5080) and see if movie is getting downloaded (After movie is queued. This depends on availability of movie in indexers configured in Prowlarr.)

## Configure Jellyfin

- Open Jellyfin at http://localhost:8096
- When you access the jellyfin for first time using browser, A guided configuration will guide you to configure jellyfin. Just follow the guide.
- Add media library folder and choose /data/movies/

## Configure Jellyseerr

- Open Jellyfin at http://localhost:5055
- When you access the jellyseerr for first time using browser, A guided configuration will guide you to configure jellyseerr. Just follow the guide and provide the required details about sonarr and Radarr.
- Follow the Overseerr document (Jellyseerr is fork of overseerr) for detailed setup - https://docs.overseerr.dev/ 

## Configure Prowlarr

- Open Prowlarr at http://localhost:9696
- Settings --> General --> Authentications --> Select Authentication and add username and password
- Add Indexers, Indexers --> Add Indexer --> Search for indexer --> Choose base URL --> Test and Save
- Add application, Settings --> Apps --> Add application --> Choose Radarr --> Prowlarr server (http://prowlarr:9696) --> Radarr server (http://radarr:7878) --> API Key --> Test and Save
- Add application, Settings --> Apps --> Add application --> Choose Sonarr --> Prowlarr server (http://prowlarr:9696) --> Sonarr server (http://sonarr:8989) --> API Key --> Test and Save
- This will add indexers in respective apps automatically.


## Configure Nginx

- Get inside Nginx container
- `cd /etc/nginx/conf.d`
- Add proxies for all tools.

`docker cp nginx.conf nginx:/etc/nginx/conf.d/default.conf && docker exec -it nginx nginx -s reload`
- Close ports of other tools in firewall/security groups except port 80 and 443.


## Apply SSL in Nginx

- Open port 80 and 443.
- Get inside Nginx container and install certbot and certbot-nginx `apk add certbot certbot-nginx`
- Add URL in server block. e.g. `server_name  localhost mediastack.example.com;` in /etc/nginx/conf.d/default.conf
- Run `certbot --nginx` and provide details asked.

## Radarr Nginx reverse proxy

- Settings --> General --> URL Base --> Add base (/radarr)
- Add below proxy in nginx configuration

```
location /radarr {
    proxy_pass http://radarr:7878;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
  }
```

- Restart containers.

## Sonarr Nginx reverse proxy

- Settings --> General --> URL Base --> Add base (/sonarr)
- Add below proxy in nginx configuration

```
location /sonarr {
    proxy_pass http://sonarr:8989;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
  }
```

## Prowlarr Nginx reverse proxy

- Settings --> General --> URL Base --> Add base (/prowlarr)
- Add below proxy in nginx configuration

This may need to change configurations in indexers and base in URL.

```
location /prowlarr {
    proxy_pass http://prowlarr:9696; # Comment this line if VPN is enabled.
    # proxy_pass http://vpn:9696; # Uncomment this line if VPN is enabled.
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
  }
```

- Restart containers.

**Note: If VPN is enabled, then Prowlarr is reachable on vpn's service name**

## qBittorrent Nginx proxy

```
location /qbt/ {
    proxy_pass         http://qbittorrent:5080/; # Comment this line if VPN is enabled.
    # proxy_pass         http://vpn:5080/; # Uncomment this line if VPN is enabled.
    proxy_http_version 1.1;

    proxy_set_header   Host               http://qbittorrent:5080; # Comment this line if VPN is enabled.
    # proxy_set_header   Host               http://vpn:5080; # Uncomment this line if VPN is enabled.
    proxy_set_header   X-Forwarded-Host   $http_host;
    proxy_set_header   X-Forwarded-For    $remote_addr;
    proxy_cookie_path  /                  "/; Secure";
}
```

**Note: If VPN is enabled, then qbittorrent is reachable on vpn's service name**

## Jellyfin Nginx proxy

- Add base URL, Admin Dashboard -> Networking -> Base URL (/jellyfin)
- Add below config in Ngix config

```
 location /jellyfin {
        return 302 $scheme://$host/jellyfin/;
    }

    location /jellyfin/ {

        proxy_pass http://jellyfin:8096/jellyfin/;

        proxy_pass_request_headers on;

        proxy_set_header Host $host;

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $http_host;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $http_connection;

        # Disable buffering when the nginx proxy gets very resource heavy upon streaming
        proxy_buffering off;
    }
```

## Jellyseerr Nginx proxy

**Currently Jellyseerr/Overseerr doesnot officially support the subfolder/path reverse proxy. They have a workaround documented here without an official support. Find it [here](https://docs.overseerr.dev/extending-overseerr/reverse-proxy)**

```
location / {
        proxy_pass http://127.0.0.1:5055;

        proxy_set_header Referer $http_referer;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Real-Port $remote_port;
        proxy_set_header X-Forwarded-Host $host:$remote_port;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-Port $remote_port;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Ssl on;
    }
```

- Restart containers


## Disclaimer  

> Neither the author nor the developers of the code in this repository **condone or encourage** downloading, sharing, seeding, or peering of **copyrighted material**.  
> Such activities are **illegal** under international laws.  
>
> This project is intended for **educational purposes only**.  
