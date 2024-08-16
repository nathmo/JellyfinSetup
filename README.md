# Introduction
This tutorial contains instruction to install your own video streaming server using jellyfin and instruction to create TV "topBox" using a raspberry pi they can plug in their TV to access all your videos that you can distribute to your less tech savy family and friends :)
# Feature list
- Docker stack (run on your own server)
- Jellyfin (for the video streaming)
- sonarr/radarr/bazarr/jackett/transmission (for automatically finding torrents and downloading them at the wanted quality)
- jellyseer (to have an interface netflix style with the whole internet as a catalog to download as a torrent)
- tdarr (for transcoding unsopported codec of your end device)
- wireguard (allow a secure access to jellyfin and all other service in the stack)
- kodi client on a raspberry pi (with wireguard support and jellyfin client)

# Caveat
the only problem that I'm currently aware of is that the CEC is not working on some of my TV so you need your smartphone to control kodi/the raspberry pi

# Server setup

I assume you have a server with at least 16 Gb of RAM and at least 2TB of storage with portainer installed (if not, read this : )

this is the config that you can paste into portainer.
the things you might want to know about / tune :

`/pool/container/` is the path to a folder on the server containing the config of each container. (have it on a fast drive) if your folder has another name, please replace it everywhere in the config.
`/terapool/media/` is the path to another folder the server with sufficient storage (2-20TB) were the video file will be stored. if your folder has another name, please replace it everywhere in the config.
`192.168.7.0/24` is the subnet for the stack network. MAKE SURE IT'S NOT THE SAME AS YOUR LOCAL NETWORK (usually 192.168.1.0/24), if you edit it, you also need to change it for each container.


once you chose where to store your data we must create each subfolder (replace the path to your folder in theses commands):
```
mkdir /pool/container/jellyfin
mkdir /pool/container/sonarr
mkdir /pool/container/radarr
mkdir /pool/container/bazarr
mkdir /pool/container/jellyseerr
mkdir /pool/container/jackett
mkdir /pool/container/transmission
mkdir /pool/container/tdarr/server
mkdir /pool/container/tdarr/logs
mkdir /pool/container/tdarr/config
mkdir /pool/container/tdarr/transcode
mkdir /pool/container/tdarr-node/configs
mkdir /pool/container/tdarr-node/logs
mkdir /pool/container/wireguard_jellyfin/configs
mkdir /pool/container/wireguard_jellyfin/wireguard

mkdir /terapool/media/tvshows
mkdir /terapool/media/movies
mkdir /terapool/media/music
```
once done go to portainer and click on Add Stack from then stack menu.
![image](https://github.com/user-attachments/assets/922be697-96c8-46b8-98c2-56bbdf014264)
give it a name (jellyfin media stack for instance) and paste the following content (with your modification for the path) in the Web editor console.
![image](https://github.com/user-attachments/assets/08ad4417-e881-4ced-961b-13aa0ba74f64)

```
version: "3"

PATHCONTAINER=/pool/container
PATHMEDIA=/terapool/media

services:
  jellyfin:
    image: linuxserver/jellyfin:latest
    container_name: jellyfin
    mem_limit: 800m
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Rome
    volumes:
      - ${PATHCONTAINER}/jellyfin:/config
      - ${PATHMEDIA}/tvshows:/data/tvshows
      - ${PATHMEDIA}/movies:/data/movies
      - ${PATHMEDIA}/music:/data/music
    ports:
      - 8096:8096
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jellyfin.rule=Host(`jellyfin.nas.local`)"
      - "traefik.http.services.jellyfin.loadbalancer.server.port=8096"
      - "traefik.http.routers.jellyfin.entrypoints=web"
      - "traefik-home.alias=jellyfin"
    restart: unless-stopped
    networks:
      wireguard_network:
        ipv4_address: 192.168.7.3
      
  sonarr:
    image: linuxserver/sonarr:latest
    container_name: sonarr
    mem_limit: 800m
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Rome
    volumes:
      - ${PATHCONTAINER}/sonarr:/config
      - ${PATHMEDIA}/tvshows:/tvshows
      - ${PATHMEDIA}/complete:/downloads/complete
    ports:
      - 8989:8989
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.sonarr.rule=Host(`sonarr.nas.local`)"
      - "traefik.http.services.sonarr.loadbalancer.server.port=8989"
      - "traefik.http.routers.sonarr.entrypoints=web"
      - "traefik-home.alias=sonarr"
    restart: unless-stopped
    networks:
      wireguard_network:
        ipv4_address: 192.168.7.4
      
  radarr:
    image: linuxserver/radarr:latest
    container_name: radarr
    mem_limit: 800m
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Rome
    volumes:
      - ${PATHCONTAINER}/radarr:/config
      - ${PATHMEDIA}/complete:/downloads/complete
      - ${PATHMEDIA}/movies:/movies
    ports:
      - 7878:7878
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.radarr.rule=Host(`radarr.nas.local`)"
      - "traefik.http.services.radarr.loadbalancer.server.port=7878"
      - "traefik.http.routers.radarr.entrypoints=web"
      - "traefik-home.alias=radarr"
    restart: unless-stopped
    networks:
      wireguard_network:
        ipv4_address: 192.168.7.5
      
  bazarr:
    image: linuxserver/bazarr:latest
    container_name: bazarr
    mem_limit: 800m
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Rome
    volumes:
      - ${PATHCONTAINER}/bazarr:/config
      - ${PATHMEDIA}/movies:/movies #optional
      - ${PATHMEDIA}/tvshows:/tvshows #optional
    ports:
      - 6767:6767
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.bazarr.rule=Host(`bazarr.nas.local`)"
      - "traefik.http.services.bazarr.loadbalancer.server.port=6767"
      - "traefik.http.routers.bazarr.entrypoints=web"
      - "traefik-home.alias=bazarr"
    restart: unless-stopped
    networks:
      wireguard_network:
        ipv4_address: 192.168.7.6

  jellyseerr:
    image: fallenbagel/jellyseerr:develop
    container_name: jellyseerr
    mem_limit: 800m
    environment:
      - LOG_LEVEL=debug
      - TZ=Europe/Rome
    ports:
      - 5055:5055
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jellyseerr.rule=Host(`jellyseerr.nas.local`)"
      - "traefik.http.services.jellyseerr.loadbalancer.server.port=5055"
      - "traefik.http.routers.jellyseerr.entrypoints=web"
      - "traefik-home.alias=jellyseerr"
    volumes:
      - ${PATHCONTAINER}/jellyseerr:/app/config
    restart: unless-stopped
    depends_on:
      - radarr
      - sonarr
    networks:
      wireguard_network:
        ipv4_address: 192.168.7.7
      
  jackett:
    image: linuxserver/jackett:latest
    container_name: jackett
    mem_limit: 800m
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Rome
      - AUTO_UPDATE=true #optional
    volumes:
      - ${PATHCONTAINER}/jackett:/config
      - ${PATHMEDIA}:/downloads
    ports:
      - 9117:9117
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jackett.rule=Host(`jackett.nas.local`)"
      - "traefik.http.services.jackett.loadbalancer.server.port=9117"
      - "traefik.http.routers.jackett.entrypoints=web"
      - "traefik-home.alias=jackett"
    restart: unless-stopped
    networks:
      wireguard_network:
        ipv4_address: 192.168.7.8

  transmission:
    image: linuxserver/transmission
    container_name: transmission
    mem_limit: 800m
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Rome
    volumes:
      - ${PATHCONTAINER}/transmission:/config
      - ${PATHMEDIA}:/downloads
    ports:
      - 9091:9091
      - 51413:51413
      - 51413:51413/udp
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.transmission.rule=Host(`transmission.nas.local`)"
      - "traefik.http.services.transmission.loadbalancer.server.port=9091"
      - "traefik.http.routers.transmission.entrypoints=web"
      - "traefik-home.alias=transmission"
    restart: unless-stopped
    networks:
      wireguard_network:
        ipv4_address: 192.168.7.9
      
  tdarr:
    container_name: tdarr
    mem_limit: 1000m
    image: ghcr.io/haveagitgat/tdarr:latest
    restart: unless-stopped
    ports:
      - 8265:8265 # webUI port
      - 8266:8266 # server port
      - 8267:8267 # Internal node port
      - 8268:8268 # Example extra node port
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.tdarr.rule=Host(`tdarr.nas.local`)"
      - "traefik.http.services.tdarr.loadbalancer.server.port=8265"
      - "traefik.http.routers.tdarr.entrypoints=web"
      - "traefik-home.alias=tdarr"
    environment:
      - TZ=Europe/Rome
      - PUID=1000
      - PGID=1000
      - UMASK_SET=002
      - serverIP=0.0.0.0
      - serverPort=8266
      - webUIPort=8265
      - internalNode=true
      - nodeID=tdarrMain
    volumes:
      - ${PATHCONTAINER}/tdarr/server:/app/server
      - ${PATHCONTAINER}/tdarr/config:/app/configs
      - ${PATHCONTAINER}/tdarr/logs:/app/logs
      - ${PATHMEDIA}:/main_media
      - ${PATHCONTAINER}/tdarr/transcode:/temp
    devices:
      - /dev/dri:/dev/dri
    networks:
      wireguard_network:
        ipv4_address: 192.168.7.10

  tdarr-node:
    mem_limit: 1500m
    container_name: tdarr-node
    image: ghcr.io/haveagitgat/tdarr_node:latest
    restart: unless-stopped
    environment:
      - TZ=Europe/Rome
      - PUID=1000
      - PGID=1000
      - UMASK_SET=002
      - nodeID=tdarrNode
      - serverIP=0.0.0.0
      - serverPort=8266
    volumes:
      - ${PATHCONTAINER}/tdarr-node/configs:/app/configs
      - ${PATHCONTAINER}/tdarr-node/logs:/app/logs
      - ${PATHMEDIA}:/main_media
      - ${PATHCONTAINER}/tdarr/transcode:/temp
    networks:
      wireguard_network:
        ipv4_address: 192.168.7.11
        
  wireguard:
    image: archef2000/pivpn:latest
    container_name: pivpn
    hostname: pivpn
    ports:
      - 51820:51820/udp
    volumes:
      - ${PATHCONTAINER}/wireguard_jellyfin/configs:/home/pivpn/configs
      - ${PATHCONTAINER}/wireguard_jellyfin/wireguard:/etc/wireguard
    environment:
      - HOST=dyn.aperture.blue
      - VPN=wireguard
      - PORT=51820
# optional
      - CLIENT_NAME=pivpn
      - NET=10.8.0.0
      - DNS1=1.1.1.1 # Client DNS
      - DNS2=9.9.9.9 # Client DNS
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    restart: always
    networks:
      wireguard_network:
        ipv4_address: 192.168.7.2


networks:
  wireguard_network:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 192.168.7.0/24 # Adjust the subnet as needed
```



# Client setup

