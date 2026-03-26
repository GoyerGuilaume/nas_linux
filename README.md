# NAS Raspberry Pi 5 — Documentation

> Raspberry Pi 5 8Go RAM · Ubuntu Server · Nextcloud · Apache2 · MariaDB · PHP 8.2 · Redis

---

## Sommaire

1. [Préparation du système](#1-préparation-du-système)
2. [SSH](#2-ssh)
3. [RAID 1](#3-raid-1)
4. [Apache2 & PHP](#4-apache2--php)
5. [MariaDB](#5-mariadb)
6. [Nextcloud](#6-nextcloud)
7. [Redis](#7-redis)
8. [Optimisations PHP (Optionnel)](#8-optimisations-php-optionnel)
9. [Gestion thermique](#9-gestion-thermique)
10. [Accès distant](#10-accès-distant)
11. [Backup](#11-backup)
12. [Outils & commandes utiles](#12-outils--commandes-utiles)
13. [Bonus](#13-bonus)

---

## 1. Préparation du système

```bash
# Mise à niveau standard
sudo apt update && sudo apt upgrade -y

# Mise à niveau complète
sudo apt full-upgrade -y

# Nettoyage
sudo apt autoremove --purge
sudo apt autoclean
```

---

## 2. SSH

Permet de communiquer avec le NAS à distance.

```bash
# Vérifier si SSH est installé
sudo systemctl status ssh

# Si non installé
sudo apt update && sudo apt install openssh-server -y

# Trouver l'IP du NAS
ip a

# Se connecter
ssh utilisateur@adresse_ip_du_nas
```

---

## 3. RAID 1

> ⚠️ **Commencer par le RAID avant l'installation de Nextcloud** pour éviter de reconfigurer le `datadirectory`.  
> ⚠️ **Selon la taille des disques, la synchronisation peut durer plusieurs heures voire plusieurs jours.**

```bash
# Installation
sudo apt install mdadm -y

# Nettoyer les disques (si vides) — remplacer sda/sdb par les valeurs de lsblk
sudo wipefs -a /dev/sda
sudo wipefs -a /dev/sdb

# Créer le RAID 1
sudo mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sda /dev/sdb

# Vérifier l'état (attendre la fin de la sync)
cat /proc/mdstat
sudo mdadm --detail /dev/md0

# Formater en ext4
sudo mkfs.ext4 /dev/md0

# Créer le point de montage
sudo mkdir -p /mnt/raid0

# Monter le RAID
sudo mount /dev/md0 /mnt/raid0

# Montage automatique au démarrage
echo "/dev/md0 /mnt/raid0 ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab

# Sauvegarder la configuration
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u

# Permissions
sudo chown -R www-data:www-data /mnt/raid0/data
sudo chmod -R 750 /mnt/raid0/data
```

### En cas de panne d'un disque

```bash
# Remplacer X par le nom du nouveau disque
sudo mdadm --add /dev/md0 /dev/sdX

# Vérifier la reconstruction
cat /proc/mdstat
sudo mdadm --detail /dev/md0
```

---

## 4. Apache2 & PHP

```bash
# Installation Apache2
apt install apache2 -y

# Prérequis PHP 8.2
apt install php php-common libapache2-mod-php php-bz2 php-gd php-mysql \
php-curl php-mbstring php-imagick php-zip php-common php-curl php-xml \
php-json php-bcmath php-xml php-intl php-gmp zip unzip wget -y

# Activation des modules requis
a2enmod env rewrite dir mime headers setenvif ssl

# Démarrage et activation au boot
systemctl restart apache2
systemctl enable apache2

# Vérifications
systemctl status apache2
apache2ctl -M
```

### Configuration VirtualHost — `/etc/apache2/sites-enabled/000-default.conf`

```bash
# Supprimer la conf par défaut et la recréer
rm /etc/apache2/sites-enabled/000-default.conf
sudo nano /etc/apache2/sites-enabled/000-default.conf
```

Contenu :

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/nextcloud

    <Directory /var/www/html/nextcloud>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

```bash
systemctl restart apache2
```

### Configuration PHP — `/etc/php/8.2/fpm/php.ini`

```bash
memory_limit = 1024M
upload_max_filesize = 10G
post_max_size = 10G
max_execution_time = 600
max_input_vars = 3000
max_input_time = 1000
date.timezone = Europe/Paris
```

Mise à jour automatique :

```bash
sudo sed -i 's/^memory_limit = .*/memory_limit = 1024M/' /etc/php/8.2/fpm/php.ini && \
sudo sed -i 's/^upload_max_filesize = .*/upload_max_filesize = 10G/' /etc/php/8.2/fpm/php.ini && \
sudo sed -i 's/^post_max_size = .*/post_max_size = 10G/' /etc/php/8.2/fpm/php.ini && \
sudo sed -i 's/^max_execution_time = .*/max_execution_time = 600/' /etc/php/8.2/fpm/php.ini && \
sudo sed -i 's/^max_input_vars = .*/max_input_vars = 3000/' /etc/php/8.2/fpm/php.ini && \
sudo sed -i 's/^max_input_time = .*/max_input_time = 1000/' /etc/php/8.2/fpm/php.ini && \
sudo sed -i 's|^date.timezone = .*|date.timezone = Europe/Paris|' /etc/php/8.2/fpm/php.ini && \
sudo systemctl restart php8.2-fpm && \
echo "Configuration mise à jour."
```

Vérification :

```bash
php-fpm8.2 -i | grep -E "memory_limit|upload_max_filesize|post_max_size|max_execution_time|max_input_vars|max_input_time|date.timezone"
```

---

## 5. MariaDB

```bash
# Installation
apt install mariadb-server -y

# Connexion
mysql
```

```sql
-- Créer la base et l'utilisateur (remplacer USER et MDP)
CREATE DATABASE ncloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'nclouduser'@'localhost' IDENTIFIED BY 'MDP';
GRANT ALL PRIVILEGES ON ncloud.* TO 'nclouduser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

```bash
# Activation et démarrage
systemctl restart mariadb
systemctl enable mariadb
systemctl status mariadb
```

---

## 6. Nextcloud

```bash
# Téléchargement
mkdir /var/www/html
cd /var/www/html
wget https://download.nextcloud.com/server/releases/latest.zip
unzip latest.zip
rm -rf /var/www/html/latest.zip

# Permissions
chown -R www-data:www-data /var/www/html/nextcloud/

# Installation (remplacer USER, USER_ADMIN et MDP)
cd /var/www/html/nextcloud
sudo -u www-data php occ maintenance:install --database \
"mysql" --database-name "nextcloud" --database-user "USER" --database-pass \
'MDP' --admin-user "USER_ADMIN" --admin-pass "MDP"
```

> Si l'erreur `Could not open input file: occ` apparaît :
> ```bash
> sudo chmod +x /var/www/html/nextcloud/occ
> ```

> ⚠️ **Faire pointer le stockage vers `/mnt/raid0`** sinon tout est à recommencer.

### config.php — `/var/www/html/nextcloud/config/config.php`

```php
<?php
$CONFIG = array (
  'memcache.local' => '\\OC\\Memcache\\APCu',
  'memcache.locking' => '\\OC\\Memcache\\Redis',
  'redis' =>
  array (
    'host' => '127.0.0.1',
    'port' => 6379,
  ),
  'trusted_domains' =>
  array (
    0 => '192.168.1.11',
    1 => '10.0.0.1',        // IP WireGuard
  ),
  'datadirectory' => '/mnt/raid0/data',
  'dbtype' => 'mysql',
  'dbname' => 'nextcloud',
  'dbhost' => 'localhost',
  'dbport' => 3306,
  'dbtableprefix' => 'oc_',
  'mysql.utf8mb4' => true,
  'dbuser' => 'nclouduser',
  'dbpassword' => 'MDP',
  'overwrite.cli.url' => 'http://192.168.1.11',
  'installed' => true,
);
```

### Page de vérification PHP (utile)

Créer `/var/www/html/nextcloud/info.php` :

```php
<?php phpinfo(); ?>
```

---

## 7. Redis

```bash
# Installation
sudo apt install redis-server php-redis -y

# Activation au boot
sudo systemctl enable --now redis-server
```

### Configuration — `/etc/redis/redis.conf`

```
bind 127.0.0.1 -::1
port 6379
```

> Redis écoute uniquement en local. Pas d'authentification nécessaire.  
> La configuration via socket Unix a été testée mais non retenue.

### Commandes utiles

```bash
# Statut
sudo systemctl status redis-server

# Tester la connexion
redis-cli ping
# Réponse attendue : PONG

# Surveiller les requêtes en temps réel
redis-cli monitor

# Statistiques
redis-cli info stats
```

---

## 8. Optimisations PHP (Optionnel)

### PHP-FPM

Permet d'accélérer l'exécution PHP sur Apache2 par rapport à mpm_prefork.

```bash
apt install php8.2-fpm

# Vérification
service php8.2-fpm status
php-fpm8.2 -v
ls -la /var/run/php/php8.2-fpm.sock

# Désactiver mod_php et prefork
a2dismod php8.2
a2dismod mpm_prefork

# Activer PHP-FPM
a2enmod mpm_event proxy_fcgi setenvif
a2enconf php8.2-fpm

systemctl restart apache2
```

### Configuration PHP-FPM — `/etc/php/8.2/fpm/pool.d/www.conf`

```bash
pm.max_children = 64
pm.start_servers = 16
pm.min_spare_servers = 16
pm.max_spare_servers = 32
```

Mise à jour automatique :

```bash
sudo sed -i \
    -e 's/^pm.max_children = .*/pm.max_children = 64/' \
    -e 's/^pm.start_servers = .*/pm.start_servers = 16/' \
    -e 's/^pm.min_spare_servers = .*/pm.min_spare_servers = 16/' \
    -e 's/^pm.max_spare_servers = .*/pm.max_spare_servers = 32/' \
    /etc/php/8.2/fpm/pool.d/www.conf

sudo systemctl restart php8.2-fpm
```

Ajout dans `/etc/apache2/sites-enabled/000-default.conf` (dans le bloc VirtualHost) :

```apache
<FilesMatch ".php$">
    SetHandler "proxy:unix:/var/run/php/php8.2-fpm.sock|fcgi://localhost/"
</FilesMatch>
```

```bash
systemctl restart apache2
```

### OPCache — `/etc/php/8.2/fpm/conf.d/10-opcache.ini`

```ini
zend_extension=opcache.so
opcache.enable_cli=1
opcache.jit=on
opcache.jit=1255
opcache.jit_buffer_size=128M
```

```bash
service php8.2-fpm restart
```

> Voir https://gist.github.com/rohankhudedev/1a9c0a3c7fb375f295f9fc11aeb116fe

### APCu — `/etc/php/8.2/fpm/conf.d/20-apcu.ini`

```bash
apt install php8.2-apcu
```

```ini
extension=apcu.so
apc.enable_cli=1
```

Ajouter dans `config.php` :

```php
'memcache.local' => '\OC\Memcache\APCu',
```

```bash
systemctl restart php8.2-fpm
systemctl restart apache2
```

---

## 9. Gestion thermique

### Matériel

- Module relais 2 canaux 5V (NPN, LOW trigger)
- Ventilateur boîtier 5V 2 fils
- Ventilateur dissipateur 5V 4 fils (tourne en permanence via USB)
- Jumper wires femelle-femelle

### Branchement GPIO → Module relais

| Module relais | Pin GPIO Raspberry Pi 5 |
|---|---|
| VCC | Pin 2 (5V) |
| GND | Pin 6 (GND) |
| IN1 | Pin 11 (GPIO 17) |

> Le jumper JD-VCC/VCC reste en place.

### Branchement ventilateur boîtier → Relais K1

| Relais K1 | Connexion |
|---|---|
| COM | Pin 4 (5V) du Pi |
| NO | Fil rouge ventilateur (+) |
| — | Fil noir ventilateur (-) → Pin 9 (GND) du Pi |

### Ventilateur dissipateur

Branché directement en USB sur le Pi (fil rouge + fil noir uniquement). Tourne en permanence dès que le Pi est alimenté.

### Installation des dépendances

```bash
sudo apt install python3-gpiozero python3-lgpio stress -y
```

### Script — `/usr/local/bin/fan_control.py`

```python
#!/usr/bin/env python3
from gpiozero import OutputDevice
from time import sleep
import logging

logging.basicConfig(
    filename='/var/log/fan_control.log',
    level=logging.INFO,
    format='%(asctime)s - %(message)s'
)

TEMP_ON = 52    # Température déclenchement ventilateur (°C)
TEMP_OFF = 42   # Température extinction ventilateur (°C)

fan = OutputDevice(17, active_high=False)

def get_temp():
    return int(open("/sys/class/thermal/thermal_zone0/temp").read()) / 1000

while True:
    temp = get_temp()

    if temp > TEMP_ON and not fan.value:
        fan.on()
        logging.info(f"Ventilateur ON - Temp: {temp}°C")
    elif temp < TEMP_OFF and fan.value:
        fan.off()
        logging.info(f"Ventilateur OFF - Temp: {temp}°C")

    sleep(5)
```

### Service systemd — `/etc/systemd/system/fan-control.service`

```ini
[Unit]
Description=Fan control based on CPU temperature
After=multi-user.target

[Service]
ExecStart=/usr/bin/python3 /usr/local/bin/fan_control.py
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now fan-control.service
```

### Commandes utiles

```bash
# Statut du service
sudo systemctl status fan-control.service

# Logs ventilateur
tail -f /var/log/fan_control.log

# Température en temps réel dans le terminal
watch -n 1 "date '+%H:%M:%S' && cat /sys/class/thermal/thermal_zone0/temp | awk '{printf \"%.1f°C\n\", \$1/1000}'"

# Test de charge CPU
stress --cpu 4 --timeout 60
```

### Températures observées

| Scénario | Température |
|---|---|
| Repos | ~40°C |
| Usage réel intensif (upload/lecture) | ~45°C |
| Stress CPU total (4 cœurs à 100%) | ~61°C |

> Hystérésis de 10°C entre ON et OFF pour éviter les cycles trop fréquents.  
> Le script doit être lancé en root (accès GPIO).

---

## 10. Accès distant

### WireGuard

#### Installation

```bash
sudo apt install wireguard qrencode -y
```

#### Génération des clés serveur

```bash
wg genkey | sudo tee /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key
```

#### Génération des clés client

```bash
# Client 1 (ex: smartphone)
wg genkey | sudo tee /etc/wireguard/client_private.key | wg pubkey | sudo tee /etc/wireguard/client_public.key

# Client 2
wg genkey | sudo tee /etc/wireguard/client2_private.key | wg pubkey | sudo tee /etc/wireguard/client2_public.key
```

#### Configuration serveur — `/etc/wireguard/wg0.conf`

```ini
[Interface]
PrivateKey = <server_private.key>
Address = 10.0.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client_public.key>
AllowedIPs = 10.0.0.2/32

[Peer]
PublicKey = <client2_public.key>
AllowedIPs = 10.0.0.3/32
```

#### Configuration client — `/etc/wireguard/client.conf`

```ini
[Interface]
PrivateKey = <client_private.key>
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = <server_public.key>
Endpoint = <IP_PUBLIQUE>:51820
AllowedIPs = 10.0.0.1/32
PersistentKeepalive = 25
```

> Pour `client2.conf` : remplacer `Address` par `10.0.0.3/24` et utiliser `client2_private.key`.

#### Activation du forwarding IP

```bash
sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sudo sysctl -p
```

#### Démarrage et activation au boot

```bash
sudo systemctl enable --now wg-quick@wg0
sudo systemctl status wg-quick@wg0
```

#### Génération des QR codes pour smartphone

```bash
sudo qrencode -t ansiutf8 < /etc/wireguard/client.conf
sudo qrencode -t ansiutf8 < /etc/wireguard/client2.conf
```

#### Ajout d'un nouveau client

1. Générer les clés client
2. Ajouter un bloc `[Peer]` dans `wg0.conf` avec la clé publique et une IP libre (`10.0.0.x/32`)
3. Créer le fichier `clientX.conf`
4. Redémarrer : `sudo systemctl restart wg-quick@wg0`
5. Générer le QR code et scanner depuis l'app WireGuard

#### Box internet — Redirection de port (NAT/PAT)

| Paramètre | Valeur |
|---|---|
| Protocole | UDP |
| Port | 51820 |
| Machine destination | 192.168.1.11 |

#### Notes

- Split tunneling actif : seul le trafic vers `10.0.0.1` passe par le tunnel (`AllowedIPs = 10.0.0.1/32`)
- Chaque client a sa propre IP : `10.0.0.2`, `10.0.0.3`, etc.
- WireGuard et Tailscale coexistent sans conflit (interfaces `wg0` et `tailscale0` séparées)
- Désactiver l'expiration de clé sur le NAS via la console Tailscale (`tailscale.com/admin` → Machines → Disable key expiry)

---

## 11. Backup

### Carte SD — Cloner l'image système

> Recommandé avant de commencer à utiliser le NAS en production.

```bash
# Lister les disques (utiliser le disque entier, pas une partition)
sudo fdisk -l

# Créer une image (remplacer X par la bonne lettre)
sudo dd if=/dev/sdX of=~/carte_sd.img bs=4M status=progress
sync

# Restaurer sur une autre carte SD (remplacer Y par la bonne lettre)
sudo dd if=~/carte_sd.img of=/dev/sdY bs=4M status=progress
sync
```

> Note : Un adaptateur SSD NVMe est disponible pour remplacer la carte SD à terme.

### Données RAID — Script de backup

```bash
#!/bin/bash

SOURCE_DIR="/mnt/raid0"
BACKUP_DIR="$1"
DATE=$(date +"%Y-%m-%d_%H-%M-%S")
ARCHIVE_NAME="backup_$DATE.tar.gz"
LOG_FILE="/var/log/backup_nas.log"
RETAIN_COUNT=5

if [ -z "$BACKUP_DIR" ]; then
    echo "Usage: $0 <chemin_du_disque_externe>"
    exit 1
fi

echo "$(date) - Début de la sauvegarde vers $BACKUP_DIR..." | tee -a "$LOG_FILE"

tar -czf "$BACKUP_DIR/$ARCHIVE_NAME" "$SOURCE_DIR" 2>> "$LOG_FILE"

if [ $? -eq 0 ]; then
    echo "$(date) - Sauvegarde terminée : $ARCHIVE_NAME" | tee -a "$LOG_FILE"
else
    echo "$(date) - Erreur lors de la création de l'archive." | tee -a "$LOG_FILE"
    exit 1
fi

# Supprimer les anciennes sauvegardes (garder les X dernières)
ls -t "$BACKUP_DIR"/backup_*.tar.gz | tail -n +$((RETAIN_COUNT+1)) | xargs rm -f

echo "$(date) - Sauvegarde complète ✅" | tee -a "$LOG_FILE"
exit 0
```

```bash
# Donner les droits
chmod 770 backup_nas.sh

# Usage
./backup_nas.sh /chemin/disque/externe

# Décompresser une archive
tar -xzf MON-BACKUP.tar.gz -C /chemin/destination
```

---

## 12. Outils & commandes utiles

### Restart des services principaux

```bash
systemctl restart redis-server
systemctl restart php8.2-fpm
systemctl restart apache2
```

### Interface graphique Ubuntu

```bash
# Désactiver au démarrage (recommandé pour un NAS)
sudo systemctl set-default multi-user.target
sudo reboot

# Réactiver
sudo systemctl set-default graphical.target
sudo reboot

# Désactiver jusqu'au prochain démarrage
sudo systemctl stop gdm

# Relancer à chaud
sudo systemctl start gdm
```

---

## 13. Bonus

### Page de vérification PHP

Créer `/var/www/html/nextcloud/info.php` :

```php
<?php phpinfo(); ?>
```

### Wake-on-LAN (non compatible Raspberry Pi)

> ⚠️ Non fonctionnel sur Raspberry Pi — nécessite un BIOS.

```bash
sudo ethtool -s eth0 wol g
# Si retourne "Supports Wake-on: d" → non compatible
```

### Vérifier l'IP publique

```bash
curl -4 ifconfig.me
```
