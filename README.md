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

