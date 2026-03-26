# Installation Nextcloud sur un linux
# Apache, MariaDB et ~~WireGuard~~ TailScale

### Mise a niveau du système 
```bash
apt update && apt upgrade -y
```

### Mise a niveau complète du système 
```bash
sudo apt full-upgrade -y
```

### Clean après maj
```bash
sudo apt autoremove --purge
sudo apt autoclean
```

## Installer un serveur SSH pour communiquer

Permet de communiquer avec le nas à distance en localhost.

Vérifier si déjà installer 
```bash
sudo systemctl status ssh
```

Si non installer 
```bash
sudo apt update && sudo apt install openssh-server -y
```

Trouver l'ip du NAS, via la box internet ou via ligne de commande
```bash
ip a
```

Se connecter avec 
```bash
ssh utilisateur@adresse_ip_du_nas
```

## RAID 1 (Attention l'opération est longue)
L'utilisation d'un raid1 classique suffit pour mon usage. **Selon la taille des disque l'opération peux durer plusieurs heure voir plusieurs jours.**
### Créer le mirroring 
Commencer par le RAID 1 permet d'éviter d'avoir à reconfigurer le fichier de config nextcloud avec un dataDirectory différent de celui de l'init (sinon c'est cassé et ça marche pas...)
/!\ Note remplacer les valeurs des disque par celle retourner par lsblk (sda, sdb, sdc etc)

Installer 
```bash
sudo apt update && sudo apt install mdadm -y
```
Nettoyer les disques (si vide)
```bash
sudo wipefs -a /dev/sda
sudo wipefs -a /dev/sdb
```
Créer le RAID
```bash
sudo mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sda /dev/sdb
```
Attendre la fin de la sync et vérifier l'état 
```bash
cat /proc/mdstat
sudo mdadm --detail /dev/md0
```

Formater le RAID en ext4
```bash
sudo mkfs.ext4 /dev/md0
```

Créer un point de montage
```bash
sudo mkdir -p /mnt/raid0
```

Monter le RAID 
```bash
sudo mount /dev/md0 /mnt/raid0
```

Monter automatiquement au démarrage
```bash
echo "/dev/md0 /mnt/raid0 ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
```

Sauvegarder 
```bash
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
```

Ajouter les permissions 
```bash
sudo chown -R www-data:www-data /mnt/raid0/data
sudo chmod -R 750 /mnt/raid0/data
```

En cas de panne de sda ou sdb remonter avec le nouveau disque (remplacer X par le nouveau nom)
```bash
sudo mdadm --add /dev/md0 /dev/sdX
```

Vérifier l'état du RAID
```bash
cat /proc/mdstat
sudo mdadm --detail /dev/md0
```

## Accès distant

### Wireguard

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
# Client 1
wg genkey | sudo tee /etc/wireguard/client_private.key | wg pubkey | sudo tee /etc/wireguard/client_public.key

# Client 2
wg genkey | sudo tee /etc/wireguard/client2_private.key | wg pubkey | sudo tee /etc/wireguard/client2_public.key
```

#### Configuration serveur — /etc/wireguard/wg0.conf (Penser à passer les valeurs dans le champs vides)
```ini
[Interface]
PrivateKey = 
Address = 10.0.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = 
AllowedIPs = 10.0.0.2/32

[Peer]
PublicKey = 
AllowedIPs = 10.0.0.3/32
```

#### Configuration client — /etc/wireguard/client.conf (Penser à passer les valeurs dans le champs vides)
```ini
[Interface]
PrivateKey = 
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = 
Endpoint = :51820
AllowedIPs = 10.0.0.1/32
PersistentKeepalive = 25
```

> Pour client2.conf : remplacer Address par `10.0.0.3/24` et utiliser client2_private.key

#### Activation du forwarding IP
```bash
sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sudo sysctl -p
```

#### Démarrage et activation au boot
```bash
sudo systemctl enable --now wg-quick@wg0
```

#### Génération QR code pour smartphone
```bash
sudo qrencode -t ansiutf8 < /etc/wireguard/client.conf
sudo qrencode -t ansiutf8 < /etc/wireguard/client2.conf
```

#### Ajout d'un nouveau client

1. Générer les clés client
2. Ajouter un bloc `[Peer]` dans `wg0.conf` avec la clé publique et une IP libre (10.0.0.x/32)
3. Créer le fichier `clientX.conf`
4. Redémarrer le service : `sudo systemctl restart wg-quick@wg0`
5. Générer le QR code et scanner depuis l'app WireGuard

#### Box internet — Redirection de port

- Protocole : UDP
- Port : 51820
- Machine destination : 192.168.1.11 (NAS)

#### Nextcloud — Domaines de confiance

Ajouter `10.0.0.1` dans `/var/www/html/nextcloud/config/config.php` :
```php
'trusted_domains' =>
array (
  0 => 'localhost',
  1 => '192.168.1.11',
  2 => '10.0.0.1',
),
```

#### Notes

- Split tunneling actif : seul le trafic vers `10.0.0.1` passe par le tunnel
- Chaque client a sa propre IP : 10.0.0.2, 10.0.0.3, etc.

## Apache2
#### Installation d'apache2
```bash
apt install apache2 -y
```

#### Installation des prérequis
```bash
apt install php php-common libapache2-mod-php php-bz2 php-gd php-mysql \
php-curl php-mbstring php-imagick php-zip php-common php-curl php-xml \
php-json php-bcmath php-xml php-intl php-gmp zip unzip wget -y
```
#### Activation des modules requis
```bash
a2enmod env rewrite dir mime headers setenvif ssl
```

#### Lancer Apach2
```bash
systemctl restart apache2
systemctl enable apache2
```

Vérifier que tout est OK 
```bash
systemctl status apache2
```

Vérifier les modules chargés 
```bash
apache2ctl -M
```

#### Installer et configurer MariaDB
```bash
apt install mariadb-server -y
```

Log dans mysql 
```bash
mysql
```

### Créer la base de données avec les bonnes permissions
Changer nclouduser par le user admin souhaité et le MDP par le mot de passer souhaité
```sql
CREATE DATABASE ncloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'nclouduser'@'localhost' IDENTIFIED BY 'MDP';
GRANT ALL PRIVILEGES ON ncloud.* TO 'nclouduser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
### Activation et démarrage des services MariaDB
```bash
systemctl restart mariadb
systemctl enable mariadb
```
Vérifier le bon fonctionnement 
```bash
systemctl status mariadb
```
## Installation Nextcloud 
```bash
mkdir /var/www/html
cd /var/www/html
wget https://download.nextcloud.com/server/releases/latest.zip
unzip latest.zip
```
Delete le zip qui ne sert plus a rien 
```bash
rm -rf /var/www/html/latest.zip
```

### Donner les droits à l'utilisateur 
```bash
chown -R www-data:www-data /var/www/html/nextcloud/
```

### Installation 
```bash
cd /var/www/html/nextcloud
```bash
sudo -u www-data php occ  maintenance:install --database \
"mysql" --database-name "nextcloud"  --database-user "USER" --database-pass \
'MDP' --admin-user "USER_ADMIN" --admin-pass "MDP"
```
Souvent on rencontre l'erreur ```Could not open input file: occ```
C'est parce qu'il manque de privilège sur la machine, un chmod devrais résoudre le soucis
```bash
sudo chmod +x /var/www/html/nextcloud/occ
```
 Contenu actuel ```var/www/html/nextcloud/config/config.php```
Exemple :
```bash
<?php
$CONFIG = array (
  'memcache.local' => '\\OC\\Memcache\\Redis',
  'redis' => 
  array (
    'host' => 'localhost',
    'port' => 6379,
  ),
  'instanceid' => 'ociq1hgiug5o',
  'passwordsalt' => 'IM6I9Z3dFCp3tPQBvePrLBAbeHva2X',
  'secret' => 'wLA++8RdkhlJKTvTPcAUaLRRaD+5njdjlip7/hO+LoZ+TF59',
  'trusted_domains' => 
  array (
    0 => '192.168.1.11',
  ),
  'datadirectory' => '/var/www/nextcloud/data',
  'dbtype' => 'mysql',
  'version' => '30.0.5.1',
  'overwrite.cli.url' => 'http://192.168.1.11',
  'dbname' => 'nextcloud',
  'dbhost' => 'localhost',
  'dbport' => 3306,
  'dbtableprefix' => 'oc_',
  'mysql.utf8mb4' => true,
  'dbuser' => 'nclouduser',
  'dbpassword' => 'MDP',
  'installed' => true,
);
```

### Configurer apache pour charger la ressource

Delete la conf apache
```bash
rm /etc/apache2/sites-enabled/000-default.conf
```
Et la recréer (j'ai eu des soucis à l'édition)
```bash
sudo nano /etc/apache2/sites-enabled/000-default.conf
```
Contenu à coller :
``` bash
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
Redémarrer 
``` bash
systemctl restart apache2
```
### Première connexion 
Se connecter sur localhost et configurer le nextcloud. 
Pour voir le port MariaDB (généralement 3306)
```bash
sudo systemctl status mariadb
```
**/!\ Faire pointer le stockage vers le volume monté** sinon il va falloir recommencer ^^

## Optimisations
### Créer une page de vérification (utile)
Dans ```/var/www/nextcloud``` ajouter un fichier ```info.php```
```bash
<?php phpinfo(); ?>
```

### Configuration PHP (Important)
Il est recommander configurer les valeurs du fichier ```/etc/php/8.3/fpm/php.ini``` pour qu'il puisse recevoir de gros fichiers (ou pas). Voici les valeurs que j'ai choisi. (Pour le memory limit ayant un raspbery 8go de ram, je pense augmenter de 1024mo à 6144mo)
```bash
memory_limit = 1024M
upload_max_filesize = 10G
post_max_size = 10G
max_execution_time = 600
max_input_vars = 3000
max_input_time = 1000
date.timezone = Europe/Paris
```

Pour tout mettre à jour d'un coup. Normalement, ça marche avec ça (si tout ce passe bien)
```bash
sudo sed -i 's/^memory_limit = .*/memory_limit = 1024M/' /etc/php/8.3/fpm/php.ini && \ sudo sed -i 's/^upload_max_filesize = .*/upload_max_filesize = 10G/' /etc/php/8.3/fpm/php.ini && \ sudo sed -i 's/^post_max_size = .*/post_max_size = 10G/' /etc/php/8.3/fpm/php.ini && \ sudo sed -i 's/^max_execution_time = .*/max_execution_time = 600/' /etc/php/8.3/fpm/php.ini && \ sudo sed -i 's/^max_input_vars = .*/max_input_vars = 3000/' /etc/php/8.3/fpm/php.ini && \ sudo sed -i 's/^max_input_time = .*/max_input_time = 1000/' /etc/php/8.3/fpm/php.ini && \ sudo sed -i 's|^date.timezone = .*|date.timezone = Europe/Paris|' /etc/php/8.3/fpm/php.ini && \ sudo systemctl restart php8.3-fpm && \ echo "Configuration mise à jour et PHP-FPM redémarré."
```

On peux vérifier que ça a bien mis à jour avec 
```bash
php-fpm8.3 -i | grep -E "memory_limit|upload_max_filesize|post_max_size|max_execution_time|max_input_vars|max_input_time|date.timezone"
```

## Outils
### Ubuntu
Désactiver l'interface graphique au démarrage. A terme, il est inutile que le NAS génère une interface graphique.
```bash
sudo systemctl set-default multi-user.target 
sudo reboot
```

La réactiver
```bash
sudo systemctl set-default graphical.target sudo reboot
```

Désactiver jusqu'au prochain démarrage 
```bash
sudo systemctl stop gdm
```

Relancer à chaud 
```bash
sudo systemctl start gdm
```

Tout restart
```bash
systemctl restart redis-server
systemctl restart php8.3-fpm
systemctl restart apache2
```

## Backup
### Carte SD

Avant de commencer à utiliser le NAS, s'il est stocker sur une carte SD comme dans mon cas (raspbery pi oblige) je recommande d'avoir une save de toute cette conf pour ne pas se la retaper. Pour ce faire il faut une autre SD de la même taille ou plus. Et la dupliquer soit via un utilitaire Windows ou mac, soit via ligne de commande avec un ordi : 

Lister les nom pour voir qui est qui : (⚠ **Ne pas utiliser une partition** comme `/dev/sdb1`, mais bien le disque entier `/dev/sdb`)
```bash
`sudo fdisk -l`
```

Créer une image de la carte SD : (X étant à remplacer par la bonne lettre)
```bash
sudo dd if=/dev/sdX of=~/carte_sd.img bs=4M status=progress
sync
```

Ici on a une image qu'on peux stocker où on veux.
Si on a une autre carte SD sous la main, on peux directement y installer notre image : (Y étant à remplacer par la bonne lettre)
```bash
sudo dd if=~/carte_sd.img of=/dev/sdY bs=4M status=progress
sync
```

### Données
Petit script de backup permettant de créer une archive avec le contenu du RAID. Si on souhaite faire un extract de toute les data du nas pour les stocker dans un endroit sécuriser. (A tester sur un disque réel)

```bash
#!/bin/bash

# Définition des variables par défaut
SOURCE_DIR="/mnt/raid0"  # Dossier à sauvegarder
BACKUP_DIR="$1"          # Le premier argument sera le disque externe
DATE=$(date +"%Y-%m-%d_%H-%M-%S")
ARCHIVE_NAME="backup_$DATE.tar.gz"
LOG_FILE="/var/log/backup_nas.log"
RETAIN_COUNT=5  # Nombre d'archives à conserver

# Vérifier si un argument a été fourni
if [ -z "$BACKUP_DIR" ]; then
    echo "Usage: $0 <chemin_du_disque_externe>"
    exit 1
fi

echo "$(date) - Début de la sauvegarde vers $BACKUP_DIR..." | tee -a "$LOG_FILE"

# Création de l'archive
tar -czf "$BACKUP_DIR/$ARCHIVE_NAME" "$SOURCE_DIR" 2>> "$LOG_FILE"

if [ $? -eq 0 ]; then
    echo "$(date) - Sauvegarde terminée avec succès : $ARCHIVE_NAME" | tee -a "$LOG_FILE"
else
    echo "$(date) - Erreur lors de la création de l'archive." | tee -a "$LOG_FILE"
    exit 1
fi

# Suppression des anciennes sauvegardes (conserver les X dernières)
echo "$(date) - Nettoyage des anciennes sauvegardes..." | tee -a "$LOG_FILE"
ls -t "$BACKUP_DIR"/backup_*.tar.gz | tail -n +$((RETAIN_COUNT+1)) | xargs rm -f

echo "$(date) - Sauvegarde complète ! ✅" | tee -a "$LOG_FILE"
exit 0
```
Penser à donner des droits suffisant 
```bash
chmod 770 backup_nas.sh
```

Usage 
```bash
./backup_nas.sh ${chemin}
```

Pour dézip le tar.gz
```bash
tar -xzf ${MON-BACKUP} -C ${Chemin vers dézip}
```


## Bonus 
### Réveil automatique via le réseau si compatible (Wake-on-LAN, Bios requis donc non compatibe raspbery)
Permet de mettre le NAS en veille au bout d'un certain temps et de le reveiller à la demande. (Non fonctionnel sur raspberry pi car nécessite un BIOS

Vérifier que c'est actif
```bash
sudo ethtool -s eth0 wol g
```

S'il retourne ceci, c'est mort :)
```bash
	Supports Wake-on: d
	Wake-on: d
```


### Installer PHP-FPM  (Optionel)

Permet d'accélérer l'exécution de fichier php sur apache2 par rapport à mpm-prefork
```bash
apt install php8.3-fpm
```

Vérifier que c'est bien créé 
```bash 
service php8.3-fpm status
php-fpm8.3 -v
ls -la /var/run/php/php8.3-fpm.sock
```

Désactiver mod_php et le module de prefork
```bash
a2dismod php8.3
a2dismod mpm_prefork
```

Activer PHP-FPM
```bash
a2enmod mpm_event proxy_fcgi setenvif
a2enconf php8.3-fpm
```

Redémarrer 
```bash 
systemctl restart apache2
```

### Configuration PHP-FPM (Optionel)

Idem qu'avec la config ici d'avant, il faut changer les values suivantes ici ```/etc/php/8.3/fpm/pool.d/www.conf```
```bash
pm.max_children = 64
pm.start_servers = 16
pm.min_spare_servers = 16
pm.max_spare_servers = 32
```

Normalement ça devrait le faire avec ça
```bash
sudo sed -i \
    -e 's/^pm.max_children = .*/pm.max_children = 64/' \
    -e 's/^pm.start_servers = .*/pm.start_servers = 16/' \
    -e 's/^pm.min_spare_servers = .*/pm.min_spare_servers = 16/' \
    -e 's/^pm.max_spare_servers = .*/pm.max_spare_servers = 32/' \
    /etc/php/8.3/fpm/pool.d/www.conf
```

Vérifier avec 
```bash
grep -E "pm.max_children|pm.start_servers|pm.min_spare_servers|pm.max_spare_servers" /etc/php/8.3/fpm/pool.d/www.conf
```

Restart
```bash
sudo systemctl restart php8.3-fpm
```

### Mettre à jour la conf default php (Optionel)
Ajouter ceci ici ```/etc/apache2/sites-enabled/000-default.conf```
```bash
<FilesMatch ".php$">
         SetHandler "proxy:unix:/var/run/php/php8.3-fpm.sock|fcgi://localhost/"
	</FilesMatch>
```

Le code doit avoir été copier ici ```
/etc/apache2/sites-enabled/000-default.conf ```après un restart 
```bash
systemctl restart apache2
```


### Activer OPCache en PHP (Optionel)
Dans ```/etc/php/8.3/fpm/conf.d/10-opcache.ini``` ajouter 
```bash
zend_extension=opcache.so
opcache.enable_cli=1
opcache.jit=on
opcache.jit = 1255
opcache.jit_buffer_size = 128M
```
**opcache.enable_cli** n'est utile que si on utilise nextcloud en mode command line

Petit restart des familles 
```bash
service php8.3-fpm restart
```
Attention, voir https://gist.github.com/rohankhudedev/1a9c0a3c7fb375f295f9fc11aeb116fe

### Activer APCu in PHP (Optionel)
Installer 
```bash
apt install php8.3-apcu
```

Ajouter ceci dans ```/etc/php/8.3/fpm/conf.d/20-apcu.ini```

```bash
extension=apcu.so
apc.enable_cli=1
```

Petit restart des familles 
```bash
systemctl restart php8.3-fpm
systemctl restart apache2
```

Ajouter la conf dans nextcloud ici ```/var/www/html/nextcloud/config/config.php```
```bash
'memcache.local' => '\OC\Memcache\APCu',
```

### Activer Redis Cache (Conseillé)
Installer
```bash
apt install redis-server php-redis -y
```

Activer le service 
```bash
systemctl start redis-server
systemctl enable redis-server
```

Configurer Redis pour utiliser le socket Unix
```bash
port 0
unixsocket /var/run/redis/redis.sock
unixsocketperm 770
```

Ajouter apache2 au groupe Redis
```bash
usermod -a -G redis www-data
```

Ajouter la conf à nextcloud ```/var/www/nextcloud/config/config.php``` 

Activer Redis dans ```/etc/php/8.3/fpm/php.ini```
```bash
redis.session.locking_enabled=1
redis.session.lock_retries=-1
redis.session.lock_wait_time=10000
```

Tout restart
```bash
systemctl restart redis-server
systemctl restart php8.3-fpm
systemctl restart apache2
```
## Gérer la température du boitier
### Matériel

- Module relais 2 canaux 5V (NPN, LOW trigger)
- Ventilateur 5V 2 fils
- Jumper wires femelle-femelle

### Branchement

### GPIO → Module relais
| Module relais | Pin GPIO |
|---|---|
| VCC | Pin 2 (5V) |
| GND | Pin 6 (GND) |
| IN1 | Pin 11 (GPIO 17) |

> Le jumper JD-VCC/VCC reste en place

### Ventilateur → Relais K1
| | |
|---|---|
| Pin 4 (5V) | COM de K1 |
| NO de K1 | Fil rouge ventilateur (+) |
| Pin 9 (GND) | Fil noir ventilateur (-) |

### Installation des dépendances
```bash
sudo apt install python3-gpiozero python3-lgpio -y
```

#### Script — /usr/local/bin/fan_control.py
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

#### Service systemd — /etc/systemd/system/fan-control.service
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

## Activation au boot
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now fan-control.service
```

## Commandes utiles
```bash
# Statut du service
sudo systemctl status fan-control.service

# Logs
tail -f /var/log/fan_control.log

# Température en temps réel
watch -n 1 "date '+%H:%M:%S' && cat /sys/class/thermal/thermal_zone0/temp | awk '{printf \"%.1f°C\n\", \$1/1000}'"

# Test de charge CPU
stress --cpu 4 --timeout 60
```

## Températures observées

| Scénario | Température |
|---|---|
| Repos | ~40°C |
| Usage réel intensif | ~45°C |
| Stress CPU total | ~61°C |

## Notes

- Hystérésis de 10°C entre ON et OFF pour éviter les cycles trop fréquents
- Le script doit être lancé en root (accès GPIO)
- Le ventilateur du dissipateur (5V, 4 fils) tourne en permanence via USB
