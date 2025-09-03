# zfsDockerNAS

La idea es montar una NAS para mi casa con servicios sobre la red local. Como no estoy 24x7 horas en casa mi idea es poder arrancar el servidor desde un icono de mi móvil y apagarlo del mismo modo. Todo funcionará con la idea de poder apagarse sin problema a demanda.

## Arranque automático WOL (Wake On LAN)

Para habilitar el Wake on LAN debes realizar dos pasos

1. Consultar el manual de tu BIOS ya que cada una habilita WOL de manera distinta y seguir esas instrucciones.
2. En Ubuntu server como se usa netplan:
```
cat /etc/netplan/50-cloud-init.yaml
network:
  version: 2
  ethernets:
    enp3s0:
      dhcp4: true
      **wakeonlan: true**
```

## Instalación UBuntu server
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

amule (descargas de internet)

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

### HomeAssistant docker-compose.yml

HomeAssistant (Para monitorizar temperaturas, controlar focos y dispositivos... domotica de mi casa)

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

### immich docker-compose.yml

immich (Portal avanzado de fotos, detección de caras, geolocalización, etc.)

```
services:
  immich-server:
    container_name: immich-server
    image: ghcr.io/immich-app/immich-server:release
    volumes:
#      - immich-data:/usr/src/app/upload
      - /datos/fotos:/usr/src/app/photos:ro
      - /datos/fotos/immich-upload:/usr/src/app/upload
    ports:
      - "2283:2283"
    env_file:
      - .env
    environment:
      # Limita workers y optimiza rendimiento
#      - IMMICH_WORKERS_INCLUDE=api
      - IMMICH_LOG_LEVEL=error
#      - IMMICH_SERVER_WORKERS=4          # Más workers API
      - IMMICH_MACHINE_LEARNING_URL=http://immich-machine-learning:3003
    depends_on:
      database:
        condition: service_healthy
      immich-redis:
        condition: service_started
    restart: always
#    deploy:
#      resources:
#        limits:
#          cpus: '6.0'      # Máximo 5 núcleos
#        reservations:
#          cpus: '3.0'      # Reserva mínima 2 núcleos
  immich-machine-learning:
    container_name: immich-machine-learning
    image: ghcr.io/immich-app/immich-machine-learning:release
    cpuset: "0-1"   # Usar solo núcleos 0 a 1
    volumes:
      - ./immich-data:/usr/src/app/upload
    env_file:
      - .env
    environment:
      # Optimizaciones específicas para ML
#      - MACHINE_LEARNING_MODEL_TTL=300
      - MACHINE_LEARNING_WORKERS=2
#      - MACHINE_LEARNING_WORKER_TIMEOUT=300
    restart: always
    deploy:
      resources:
        limits:
          cpus: '3.0'      # Máximo 3 núcleos para ML
#          memory: 6G       # Límite explícito de memoria
        reservations:
          cpus: '1.0'      # Reserva mínima 1 núcleo
#          memory: 1G
  database:
    container_name: immich-postgres
    image: registry.hub.docker.com/tensorchord/pgvecto-rs:pg14-v0.2.0
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      # Optimizaciones PostgreSQL
      POSTGRES_INITDB_ARGS: "--data-checksums"
    command: >
      postgres
      -c shared_preload_libraries=vectors.so
      -c shared_buffers=8GB
      -c effective_cache_size=20GB
      -c maintenance_work_mem=2GB
      -c work_mem=256MB
      -c checkpoint_completion_target=0.9
      -c wal_buffers=64MB
      -c default_statistics_target=200
      -c random_page_cost=1.1
      -c effective_io_concurrency=200
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USERNAME} -d ${DB_DATABASE_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: always
    deploy:
      resources:
        limits:
          cpus: '3.0'      # Máximo 2 núcleos
        reservations:
          cpus: '1.0'      # Reserva mínima 0.5 núcleos
  immich-redis:
    container_name: immich-redis
    image: redis:7
    volumes:
      - redisdata:/data
    # Optimizaciones Redis para reducir CPU
    command: >
      redis-server
      --save 900 1
      --save 300 10
      --save 60 10000
    restart: always
volumes:
  immich-data:
  pgdata:
  redisdata:
  ml-cache:  # Nuevo volumen para cache de modelos ML
```

### minidlna docker-compose.yml

minidlna (Para compartir imágenes y vídeos por mi red local)

```
services:
  minidlna:
    image: vladgh/minidlna:latest
    container_name: minidlna
    restart: unless-stopped
    network_mode: host
    environment:
      - MINIDLNA_MEDIA_DIR_1=V,/media/pelis
      - MINIDLNA_MEDIA_DIR_2=V,/media/series
      - MINIDLNA_MEDIA_DIR_3=A,/media/musica
      - MINIDLNA_MEDIA_DIR_4=P,/media/fotos
      - MINIDLNA_FRIENDLY_NAME=Servidor DLNA nugbe
      - MINIDLNA_INOTIFY=yes
    volumes:
      - /multimedia/pelis:/media/pelis:ro
      - /multimedia/series:/media/series:ro
      - /multimedia/musica:/media/musica:ro
      - /datos/fotos:/media/fotos:ro
      - minidlna_cache:/var/cache/minidlna
    entrypoint: >
      sh -c "rm -f /minidlna/cache/files.db && exec /entrypoint.sh"

volumes:
  minidlna_cache:
```

### nginx docker-compose.yml

nginx (Servidor web con la página de inicio que hemos comentado)

```
services:
  steampunk-web:
    image: nginx:alpine
    container_name: steampunk_web
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
      - ./certs:/etc/nginx/certs:ro
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    restart: unless-stopped

```

Como detalle, la home page con su index.html está en el directorio html/index.html

### Portainer docker-compose.yml

Portainer (Portal con todos estos conteneodres para parar/arrancar/monitorizar estos servicios o poner más)

```
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - "9000:9000"   # Puerto web
      - "9443:9443"   # Puerto web seguro HTTPS
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock  # Acceso a Docker del host
      - portainer_data:/data                        # Persistencia de configuración
    tty: true

volumes:
  portainer_data:
```

### qbittorrent docker-compose.yml

qbittorrent (descargas de internet)

```
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
      - WEBUI_PORT=8080
      - WEBUI_PASSWORD=${WEBUI_PASSWORD}
    volumes:
      - ./config:/config
      - /multimedia/INCOMING:/downloads
    ports:
      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"
    restart: unless-stopped
    networks:
      - torrent-network
    depends_on:
      - prowlarr

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
    volumes:
      - ./prowlarr-config:/config
    ports:
      - "9696:9696"
    restart: unless-stopped
    networks:
      - torrent-network

networks:
  torrent-network:
    driver: bridge
```

### samba docker-compose.yml

samba (Compartición de archivos de la NAS por la red local)

```
services:
  samba:
    image: dperson/samba:latest
    container_name: samba
    restart: always
    ports:
      - "139:139"
      - "445:445"
      - "137:137/udp"
      - "138:138/udp"
    environment:
      # Definición de los shares (nombre;ruta;browsable;readonly;guest)
      SHARE:  "Pelis;/multimedia/pelis;yes;no;yes"
      SHARE2: "Series;/multimedia/series;yes;no;yes"
      SHARE3: "Música;/multimedia/musica;yes;no;yes"
      SHARE4: "INCOMING;/multimedia/INCOMING;yes;no;yes"
      SHARE5: "Fotos;/datos/fotos;yes;no;yes"

      # Ajustar permisos para permitir escritura al invitado
      PERMISSIONS: "yes"
    volumes:
      - /multimedia/pelis:/multimedia/pelis
      - /multimedia/series:/multimedia/series
      - /multimedia/musica:/multimedia/musica
      - /multimedia/INCOMING:/multimedia/INCOMING
      - /datos/fotos:/datos/fotos
```

### wetty docker-compose.yml

wetty (Acceso ssh desde el propio servidor web)

```
services:
  wetty:
    image: wettyoss/wetty:latest
    container_name: wetty
    restart: unless-stopped
    ports:
      - "3000:3000"
    command: >
      --ssh-host 192.168.1.222
      --ssh-user nugbe
      --base /
    environment:
      - TERM=xterm-256color
```
