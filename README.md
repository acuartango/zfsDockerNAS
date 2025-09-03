# zfsDockerNAS

Instalación UBuntu server
- Español
- Ubuntu server minimized + third-party drivers

DISCOS:
- SSD SATA 1tb
- HDD SATA 1tb
- HDD SATA 1tb
- SSD SATA 1tb
- SSD SATA 2tb


Instalaré después ZFS con todos los discos

## Habilitar WOL (Arranque automático desde la red de casa)

- Verificar soporte WOL en la interfaz de red
```sudo ethtool eth0 | grep Wake-on```

- Habilitar WOL
```sudo ethtool -s eth0 wol g```

- Para que persista después del reinicio
```echo 'ethtool -s eth0 wol g' | sudo tee -a /etc/rc.local```

- /etc/netplan/50-cloud-init.yaml
```
network:
  version: 2
  ethernets:
    enp3s0:
      dhcp4: true
      wakeonlan: true
```

## Software recomendado

```
sudo apt install zfsutils-linux lshw htop bash-completion hdparm sudo smartmontools 



```

## ZFS

- Creo un simil a RAID1 con ZFS:
```
wipefs -a /dev/sda
wipefs -a /dev/sdd
zpool create -f datos mirror /dev/sda /dev/sdd
zfs set compression=lz4 datos

# Para que al reiniciar se automonten las particiones ZFS:
systemctl enable zfs-import-cache
systemctl enable zfs-mount
```

- Creo un RAID 0 con mdadm y luego lo meto en ZFS:
```


wipefs -a /dev/sdb
wipefs -a /dev/sdc
wipefs -a /dev/sde
mdadm --create --verbose /dev/md0 --level=0 --raid-devices=3 /dev/sdb /dev/sdc /dev/sde
cat /proc/mdstat
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
sudo update-initramfs -u
# Creamos sobre el un pool ZFS:
zpool create multimedia /dev/md0
zfs set compression=lz4 multimedia

```
## Migrar datos desde otro ZFS

Este método es el más veloz para copiar por red una unidad ZFS a otro servidor. Se ejecuta todo desde la máquina origen:

1. Crear snapshot
```
sudo zfs snapshot datos@migracion
```
2. Enviar los datos al destino
```
sudo zfs send datos@migracion | pv | ssh nugbe@192.168.1.139 sudo zfs receive datos
```

## Instalar Docker
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker $USER
newgrp docker

```

## Contenedores

La idea es montar una homepage bajo nginx para poner una simple página de acceso a todos los servicios del servidor para simplificar su uso.

Contenedores a usar
 amule (descargas de internet)
 HomeAssistant (Para monitorizar temperaturas, controlar focos y dispositivos... domotica de mi casa)
 immich (Portal avanzado de fotos, detección de caras, geolocalización, etc.)
 minidlna (Para compartir imágenes y vídeos por mi red local)
 nginx (Servidor web con la página de inicio que hemos comentado)
 Portainer (Portal con todos estos conteneodres para parar/arrancar/monitorizar estos servicios o poner más)
 qbittorrent (descargas de internet)
 samba (Compartición de archivos de la NAS por la red local)
 wetty (Acceso ssh desde el propio servidor web)


### amule docker-compose.yml

```
services:
  amule:
    image: ngosang/amule:latest
    container_name: amule
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
      - WEBUI_PWD=nugbe
    volumes:
      - /datos/docker/config/amule:/home/amule/.aMule
      - /multimedia/INCOMING:/incoming
      - /datos/docker/logs/amule:/temp
    ports:
      - 4711:4711       # Web UI
      - 4662:4662/tcp   # ED2K TCP
      - 4665:4665/udp   # ED2K UDP (optional)
      - 4672:4672/udp   # ED2K UDP (optional)
    restart: unless-stopped
```

### HomeAssistant
```
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:stable
    volumes:
      - ./config:/config
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped
    network_mode: host
```




