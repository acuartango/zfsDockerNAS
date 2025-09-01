# zfsDockerNAS

Instalación UBuntu server
- Español
- Ubuntu server minimized + third-party drivers

Instalaré después ZFS con todos los discos

## Habilitar WOL (Arranque automático desde la red de casa)

- Verificar soporte WOL en la interfaz de red
```sudo ethtool eth0 | grep Wake-on```

- Habilitar WOL
```sudo ethtool -s eth0 wol g```

- Para que persista después del reinicio
```echo 'ethtool -s eth0 wol g' | sudo tee -a /etc/rc.local```

- /etc/netplan/50-cloud-init.yaml
network:
  version: 2
  ethernets:
    enp3s0:
      dhcp4: true
      wakeonlan: true
      
## Software recomendado

```
sudo apt install bash-completion
sudo apt install zfsutils-linux

```
