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

## Configure Cleanuparr

Cleanuparr is optional, but an important set for my *arr stack, as it automatically cleans off the download folder and doesn't let it bloat with all the copied files from Sonarr and Radarr.

For example you can setup a few rules so that it removes downloads that are stalled or slow from Deluge, you can also give it a rule so it seeds the downloads for only a set number of time and then remove the downloaded file from the download folder.

When you first log into Cleanuparr (at http://your.server.ip:11011) you will be greeted with an initial setup request,
Enter your credentials as you wish.

We don't have a plex account since we are using Jellyfin, so we can skip the Plex setup.
Next you can sign into the instance with the credentials you gave earlier.

You will see a dashboard as follow:

<img src="../../assets/cleanuparr-home.png" width=50% height=50% alt="cleanuparr's home screen">

The first few things to do is to setup the media apps.
Add an instance to Sonarr and Radarr with `sonarr` and `radarr` as URL's hostname instead of `localhost` (you can leave the external URL empty if you like) along with their respective API Keys.

This will connect Cleanuparr to Sonarr and Radarr.
Next you need to add the download client, use Deluge as client type and use `http://deluge:8112` as the URL for this setup (instead of `localhost`).

This will connect Cleanuparr to the download client so that it can remove torrents that aren't needed.
Next you can add the rules as you wish.
Personally i use the Queue Cleaner with a stalled rule along with a slow download rule:

<img src="../../assets/cleanuparr-stalled-rule.png" width=50% height=50% alt="cleanuparr's stalled rule">

For the Download Cleaner, we enabled the `Label` plugin for Deluge in order to use this specifically.
You can set a two different rules for the `tv-sonarr` category and `radarr` category and give them each a seed time as you wish, so that after a certain amount of time Cleanuparr will remove the torrent automatically.




## Disclaimer  

> Neither the author nor the developers of the code in this repository **condone or encourage** downloading, sharing, seeding, or peering of **copyrighted material**.  
> Such activities are **illegal** under international laws.  
>
> This project is intended for **educational purposes only**.  
