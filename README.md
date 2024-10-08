# Introduction
This tutorial contains instruction to install your own video streaming server using jellyfin and instruction to create TV "topBox" using a raspberry pi 3 or 4 that can be plugged in a TV to access all your videos that you can distribute to your less tech savy family and friends :)
# Feature list
- Docker stack (run on your own server)
- Jellyfin (for the video streaming)
- sonarr/radarr/bazarr/jackett/transmission (for automatically finding torrents and downloading them at the wanted quality)
- jellyseer (to have an interface netflix style with the whole internet as a catalog to download as a torrent)
- tdarr (for transcoding unsopported codec of your end device)
- wireguard (allow a secure access to jellyfin and all other service in the stack)
- kodi client on a raspberry pi (with wireguard support and jellyfin client)

# Caveat
the first problem that I'm currently aware of is that the CEC is not working on some of my TV so you need your smartphone to control kodi/the raspberry pi

# Server setup

I assume you have a server with at least 16 Gb of RAM and at least 2TB of storage with portainer installed (and a static IP address of a domain name with dynDNS) (if not, read this : https://github.com/nathmo/Quartz64_NAS)

this is the config that you can paste into portainer.
the things you might want to know about / tune :

`/pool/container/` is the path to a folder on the server containing the config of each container. (have it on a fast drive) if your folder has another name, please replace it everywhere in the config.
`/terapool/media/` is the path to another folder the server with sufficient storage (2-20TB) were the video file will be stored. if your folder has another name, please replace it everywhere in the config.
`192.168.7.0/24` is the subnet for the stack network. MAKE SURE IT'S NOT THE SAME AS YOUR LOCAL NETWORK (usually 192.168.1.0/24), if you edit it, you also need to change it for each container.

here is a table with the port and IP address of each container. (if you access the services over the VPN, you need to specify the IP from the table. if you want to access it directly from your LAN. use the IP of the server where docker is running (192.168.1.10 for instance)

| Title         | Port      | URL                           |
|---------------|-----------|-------------------------------|
| jellyfin      | 8096      | http://192.168.7.3:8096/      |
| sonarr        | 8989      | http://192.168.7.4:8989/      |
| radarr        | 7878      | http://192.168.7.5:7878/      |
| bazarr        | 6767      | http://192.168.7.6:6767/      |
| jellyseerr    | 5055      | http://192.168.7.7:5055/      |
| jackett       | 9117      | http://192.168.7.8:9117/      |
| transmission  | 9091      | http://192.168.7.9:9091/      |
| tdarr         | 8265      | http://192.168.7.10:8265/     |
| tdarr-node    | N/A       | N/A                           |
| wireguard     | 51820/udp | N/A                           |

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

this is what you should copy paste. Note the PATHCONTAINER=/pool/container and PATHMEDIA=/terapool/media variable that you should edit if needed.

```
version: "3"

PATHCONTAINER=/pool/container
PATHMEDIA=/terapool/media
SUBNET=192.168.7
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
    restart: unless-stopped
    networks:
      wireguard_network:
        ipv4_address: ${SUBNET}.3
      
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
    restart: unless-stopped
    networks:
      wireguard_network:
        ipv4_address: ${SUBNET}.4
      
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
    restart: unless-stopped
    networks:
      wireguard_network:
        ipv4_address: ${SUBNET}.5
      
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
    restart: unless-stopped
    networks:
      wireguard_network:
        ipv4_address: ${SUBNET}.6

  jellyseerr:
    image: fallenbagel/jellyseerr:develop
    container_name: jellyseerr
    mem_limit: 800m
    environment:
      - LOG_LEVEL=debug
      - TZ=Europe/Rome
    ports:
      - 5055:5055
    volumes:
      - ${PATHCONTAINER}/jellyseerr:/app/config
    restart: unless-stopped
    depends_on:
      - radarr
      - sonarr
    networks:
      wireguard_network:
        ipv4_address: ${SUBNET}.7
      
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
    restart: unless-stopped
    networks:
      wireguard_network:
        ipv4_address: ${SUBNET}.8

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
    restart: unless-stopped
    networks:
      wireguard_network:
        ipv4_address: ${SUBNET}.9
      
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
        ipv4_address: ${SUBNET}.10

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
        ipv4_address: ${SUBNET}.11
        
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
        ipv4_address: ${SUBNET}.2


networks:
  wireguard_network:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 192.168.7.0/24 # Adjust the subnet as needed
```

once this is done you should just go to your router and expose your public port 51820 to the host running the container so that external people can access your jellyfin over the VPN.

now to create VPN config to share to your friend / use in the next part of this tutorial do the following :

click on the shell icon of the pivpn container from the container list.
![image](https://github.com/user-attachments/assets/bf4dcb79-6f06-489b-a921-19ae0c1cdfe4)
click connect
![image](https://github.com/user-attachments/assets/0ff88913-0abc-4164-9de2-cb306425326b)

then to create new config execute the following command : 
`pivpn -a -n theNameOfTheClient`

![image](https://github.com/user-attachments/assets/863a16c9-a5bf-49f3-8153-dec682caa7a6)


then run `cat config/theNameOfTheClient.conf`
and copy paste the config in wireguard / a textfile with the .conf exitention that you can import into wireguard.

# Client setup

you need a raspberry pi with a powersupply, HDMI cable /micro HDMI and a micro SD card.

install Raspberry Pi imager (https://www.raspberrypi.com/software/) if you dont have it.

then select the raspberry pi model (RPI 4 in my case)

for the OS, go under the Media player OS and Select LibreELEC 
![image](https://github.com/user-attachments/assets/ae75a90a-9be0-4f32-b63d-ffd0f1cc09d5)

the select your SD card and click next.

now plug the SD card in your raspberry pi, connect it to your TV and power it.
(if the remote of your tv fail to control the raspberry pi, use a USB keyboard for the setup)

now you can do the innitial setup

![image](https://github.com/user-attachments/assets/5a47c60f-ad3b-4aaf-a321-67ff7ad5bda2)
feel free to change the hostname to something different
![image](https://github.com/user-attachments/assets/11c81f70-85de-4541-9b88-43d63db9f382)
configure the network. (for ease of configuration you can set a static IP address but ensure it not occupied by something else)
![image](https://github.com/user-attachments/assets/1b8b125d-587c-4f6f-92d8-b575795505d0)
![image](https://github.com/user-attachments/assets/0011c2fe-66b9-49cf-a6dc-c1a50cca687f)
![image](https://github.com/user-attachments/assets/cdcd2dc7-1b2f-44f1-9666-61e7671a803d)
![image](https://github.com/user-attachments/assets/991d560c-e5c5-464d-97fa-b45228b4cbb3)

then you want to enable SSH and SMB (SSH is required in the next step)

![image](https://github.com/user-attachments/assets/4b2bb500-f905-4713-bf88-5588e8d26823)

![image](https://github.com/user-attachments/assets/dda4bb12-2996-41d0-88d8-c66f547facf9)

(make sure that the password is what you expect. the default keyboard layout is US so double check)

now we can install jellyfin client :
first we add the jellyfin repo 
![image](https://github.com/user-attachments/assets/619730ab-d22a-411a-acc0-b74117e22776)

![image](https://github.com/user-attachments/assets/ab4e7043-87dd-4d28-b6ee-8210248c4e4f)
click enter when <None> is selected and enter the following URL : 
![image](https://github.com/user-attachments/assets/f2d8af31-94d3-4f03-a892-fd4f912951e0)
`https://kodi.jellyfin.org`
![image](https://github.com/user-attachments/assets/2f477c95-17d6-4ef9-a2b0-95e3ebef60b0)
dont mix up the name with the url. the name can be anything. set it to jellyfin or something like that
![image](https://github.com/user-attachments/assets/aa476e4e-ae1b-4642-b32b-0f658c52dcfc)


now you can go to the addond manager in the settings :

![image](https://github.com/user-attachments/assets/f4f52dcb-1cce-4f8a-ae53-59e4ccafa14e)
and install the repository from a ZIP file
![image](https://github.com/user-attachments/assets/2f255880-a74b-4384-a8a3-ab925829be72)

modify the settings regarding the zip plugins

![image](https://github.com/user-attachments/assets/6d24caba-a04c-4509-8232-ab91799b303e)

![image](https://github.com/user-attachments/assets/94fb6c0b-086d-4b24-949b-0c13dadb0cdf)

now you can come back to the menu and install jellyfin as a normal plugin 

![image](https://github.com/user-attachments/assets/f10b9d15-f8af-4604-979f-3f12e8e77eee)
![image](https://github.com/user-attachments/assets/a588a1d8-f6e7-43c1-8aa9-d852e77c1895)
![image](https://github.com/user-attachments/assets/00f48b7b-fe12-4ddd-8e66-4e48d92bedbc)
![image](https://github.com/user-attachments/assets/03245ba9-335f-4b12-84d0-c067dbf82090)


once the innitial config is done and SSH enable, we can connect to it.
`ssh root@192.168.1.16`
![image](https://github.com/user-attachments/assets/85247ef3-dc93-4990-bd97-a31c433ae646)

now we need to setup the VPN access on the raspberry pi so you can gift it to friend/family and they can use it out of the box with no tech skill required.

you need to make a VPN config as explained earlier and copy paste it into a text editor.
the config should look something like this :
```

[Interface]
PrivateKey = MYPRIVATEKEY
Address = MY_ADDRESS_ON_THE_VPN
DNS = 1.1.1.1, 9.9.9.9

[Peer]
PublicKey = MYPUBLICKEY
PresharedKey = MYPRESHAREDKEY
Endpoint = MYDOMAIN:51820
AllowedIPs = 0.0.0.0/0, ::0/0
PersistentKeepalive = 25
```

first create the VPN config
`nano .config/wireguard/wireguard.config.original`
then copy this content inside that file and modify the required field
```
[provider_wireguard]
Type = WireGuard
Name = WireGuard VPN Tunnel
Host = MYDOMAIN
WireGuard.Address = MY_ADDRESS_ON_THE_VPN
WireGuard.ListenPort = 51820
WireGuard.PrivateKey = MYPRIVATEKEY
WireGuard.PublicKey = MYPUBLICKEY
WireGuard.PresharedKey = MYPRESHAREDKEY
WireGuard.DNS = 9.9.9.9, 1.1.1.1
WireGuard.AllowedIPs = 0.0.0.0/0
WireGuard.EndpointPort = 51820
WireGuard.PersistentKeepalive = 25
```

then you can add this script : (do not forget to replace MYDOMAIN by your domain/STATIC IP address)
`nano .config/wireguard/connectVPN.sh`
```
#!/bin/bash

#wait 10s for network... any ideas on how to fix this? :/
sleep 10

#resolve hostname to ip and save it in config file
SERVER_HOST="MYDOMAIN"
SERVER_IP=$(getent hosts $SERVER_HOST | awk '{ print $1 }')
echo "Server host: $SERVER_HOST"
echo "Resolved server ip: $SERVER_IP"
sed "s/$SERVER_HOST/$SERVER_IP/g" /storage/.config/wireguard/wireguard.config.original > /storage/.config/wireguard/wireguard.config

#wait 1 sec for network name to be available
sleep 1

#list connections and select the one with Wireguard in name/description
VPN_CONNECTION_NAME=$(connmanctl services | grep 'WireGuard' | awk -F' ' '{print $NF}')
echo "VPN connection name: $VPN_CONNECTION_NAME"

#connect to VPN
connmanctl connect $VPN_CONNECTION_NAME
```
you can add this script :

`nano .config/wireguard/disconnectVPN.sh`

```
#!/bin/bash

#list connections and select the one with Wireguard in description
VPN_CONNECTION_NAME=$(connmanctl services | grep 'WireGuard' | awk -F' ' '{print $NF}')
echo "VPN connection name: $VPN_CONNECTION_NAME"

#connect to VPN
connmanctl disconnect $VPN_CONNECTION_NAME
```
run `chmod +x /storage/.config/wireguard/connectVPN.sh` and 
`chmod +x /storage/.config/wireguard/disconnectVPN.sh` to make the script executable.

now we create the service that will run the VPN whenever the PI boot :
`nano .config/system.d/wireguard.service`

again, just paste that inside and save it.
```
[Unit]
Description=WireGuard VPN Service
After=network-online.target nss-lookup.target wait-time-sync.service connman-vpn.service
Before=kodi.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/storage/.config/wireguard/connectVPN.sh
ExecStop=/storage/.config/wireguard/disconnectVPN.sh

[Install]
WantedBy=multi-user.target
```

run theses three command and you are set.
`systemctl enable /storage/.config/system.d/wireguard.service`

`systemctl start wireguard.service`

`systemctl status wireguard.service`

finally reboot `reboot now` and connect back. You can test the connection by pinging the container on ${SUBNET}.3 (replace what you set for SUBNET in the server section) `ping 192.168.7.3`
if you have issue pinging, ensure there was no mistake when copy pasting.

now you can add the jellyfin server address

once installed, jellycon will ask you for the server URL

![image](https://github.com/user-attachments/assets/ebbf9fcb-153a-44b6-9810-f21402ff93f0)
set it the container IP address + port. ( http://192.168.7.3:8096 ) (don't add a / at the end)
![image](https://github.com/user-attachments/assets/761dc952-955a-4670-b179-8a5af31366d5)
as a last optional step if the HDMI control fail, you can use your smartphone as a remote with app such as kore.
fo that : go to settings -> Services -> Control -> Allow remote control via HTTP (you might wanto to disable authentication)
then your smartphone should automatically detect it when kore is launched and allow you to control the pi.



