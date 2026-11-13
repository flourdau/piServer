#   PiServer

```
	created : 09/07/2025  
	updated : 12/11/2026

	@flourdau
```
##	🌍 Hello World!  
*Si vous souhaitez installer **Raspberry Pi OS Lite** (Debian 13 *Trixie*), en faire un **serveur multimédia** avec un disque dur (extern USB3 2To dans mon cas) 
Je centralise & partage [mes notes ici](https://flourdau.github.io/piServer/)!  
Bon doubles & triples clics!* ^^

##	SOMMAIRE:
- 📋 **Préparation**
- 🔧 **Configurations**
- 🛡️ **Sécurisation OS & réseau** *(nftables, fail2ban, ssh)*
- 💾 **HDD & Partage** *(ntfs-3g, samba, minidlna)*
- 🤖 **Coeur IoT** *(nodejs, nodered)*
- 🌍 **Stack Web** *(apache2, php, mariadb, symfony)*

###	🔗 Liens: 
- [Raspberry Pi](https://www.raspberrypi.com)  
- [Raspberry Pi OS - ISO](https://www.raspberrypi.com/software/operating-systems/)  
- [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
- [Raspberry Pi Documentation](https://www.raspberrypi.com/documentation/)

---

###	⚠️ Variables à remplacer dans le tutoriel ⚠️

| Variable | Signification |
|----------|---------------|
| `USER` | Utilisateur créé avec l'Imager |
| `HOSTNAME` | Nom de la machine créée avec l'Imager |
| `PORT` | Port |
| `LAN_CIDR` | Groupe réseau |
| `UUID_NTFS` | Identifiant HDD |

---

##	📋 PRÉPARATION :

0. Générer la clé sur le poste d'administration
```bash
ssh-keygen -t ed25519 -C "USER-admin"
cat ~/.ssh/id_ed25519.pub
```
1. Télécharger [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
2. Choisir : *Raspberry Pi 4* → *Raspberry Pi OS (other)* → **Raspberry Pi OS Lite (64-bit)** → le support microSD ⚠️
3. Dans la personnalisation de l'Imager :
   - nom d'hôte : `HOSTNAME` ;
   - utilisateur `USER` + mot de passe ;
   - langue/clavier : France (`Europe/Paris`, `fr`) ;
   - SSH activé, **authentification par clé publique** (coller la clé ed25519 ci-dessous) ;
   - Wi-Fi : non (câble Ethernet).


##	🔧 CONFIGURATIONS :
###	⚠️ Remplacer `USER`, `HOSTNAME`,`SSH_PORT`...
`ssh $USER@HOSTNAME`  
*Ensuite une fois connecté sur la machine localement ou en SSH :*  
```bash
sudo raspi-config
sudo reboot
sudo apt-get update
sudo apt-get full-upgrade
sudo apt-get autoclean
sudo reboot
sudo apt-get install -y nftables fail2ban ntfs-3g samba minidlna mosquitto mosquitto-clients git python3-systemd python3-pip apache2 mariadb-server mariadb-client librabbitmq-dev composer php8.4 php8.4-{apcu,amqp,bcmath,bz2,common,cli,curl,dev,fpm,gd,intl,mbstring,mysql,pgsql,opcache,readline,redis,xdebug,xml,yaml,zip}
```
---

##	🛡️ SÉCURISATION OS & RÉSEAU :
###		1. NFTABLES
*Sauvegarde de la version originale*
```bash
sudo cp -a /etc/nftables.conf /etc/nftables.conf.ORI
```
---

*Copie & modification de l'exemple workstation.nft*
```bash
sudo cp -Rpv /usr/share/doc/nftables/examples/workstation.nft /etc/nftables.conf
sudo nano /etc/nftables.conf
```
```bash
#!/usr/sbin/nft -f
define LAN4 = 192.168.1.0/24
define LAN6 = { fe80::/10, fc00::/7 }

table inet filter
delete table inet filter

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        iif lo accept
        ct state invalid drop
        ct state established,related accept
        ip  saddr $LAN4 tcp dport 44044 ct state new accept
        ip6 saddr $LAN6 tcp dport 44044 ct state new accept

        # Regle Temporaire
        ip  saddr $LAN4 tcp dport 22 ct state new accept
        # ---

        ip  saddr $LAN4 tcp dport { 80, 445, 1880, 1883, 8001, 8200 } ct state new accept
        ip6 saddr $LAN6 tcp dport { 80, 445, 1880, 1883, 8001, 8200 } ct state new accept
        ip  saddr $LAN4 udp dport { 1900, 5353 } accept
        ip6 saddr $LAN6 udp dport { 1900, 5353 } accept

        ip protocol icmp icmp type echo-request accept

        meta nfproto ipv6 icmpv6 type { destination-unreachable, packet-too-big, time-exceeded, parameter-problem, echo-reply, echo-request, nd-router-solicit, nd-router-advert, nd-neighbor-solicit, nd-neighbor-advert, 148, 149 } accept
        ip6 saddr fe80::/10 icmpv6 type { 130, 131, 132, 143, 151, 152, 153 } accept

        counter drop
    }
}
```

**Vérifier la syntaxe sans rien appliquer :**  
`sudo nft -c -f /etc/nftables.conf`  
*La commande ne doit rien afficher.*

**Sécurité contre l'auto-blocage** : on programme la suppression de la table dans 5 minutes. Si la connexion SSH est coupée, le pare-feu se retire tout seul.

```bash
sudo systemd-run --on-active=5min --unit=nft-rollback nft delete table inet filter
sudo systemctl enable --now nftables
sudo nft list ruleset
```
Ouvrir un **second terminal** SSH pour vérifier que l'accès fonctionne toujours :

---

```bash
ssh USER@HOSTNAME
```
Si la connexion fonctionne, annuler la sécurité contre l'auto-blocage:

```bash
sudo systemctl stop nft-rollback.timer
systemctl list-timers | grep nft-rollback
```
*La seconde commande ne doit plus rien afficher.*

---
###		2. SSH 

#### Créer les clefs:  
*Depuis la machine hôte **Bash** :*
```bash
ssh-keygen
```
```bash
ssh-copy-id -pPORT USER@HOSTNAME
```
---
*Ou avec **PowerShell** :*
```cmd 
ssh-keygen.exe  -t ed25519
```
```cmd
scp.exe -PPORT .\.ssh\id_ed25519.pub USER@HOSTNAME:~/.ssh/authorized_keys
```
```cmd
ssh -pPORT USER@HOSTNAME
```
####	Configurations :  
`sudo nano /etc/ssh/sshd_config.d/00-HOSTNAME.conf`  
*le préfixe `00-` fait gagner ce fichier sur les autres : sshd retient la première valeur lue*.

```bash
Port PORT
PermitRootLogin no
AllowUsers USER
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
X11Forwarding no
MaxAuthTries 3
```
**⚠️ATTENTION Remplacer USER & PORT!!!**  

`sudo systemctl restart ssh`  
**⚠️Garder la session SSH ouverte**, puis, depuis un **second terminal** :

```bash
ssh -p PORT USER@HOSTNAME
```

Seulement si cette connexion fonctionne, retirer la règle temporaire du port 22 : supprimer dans `/etc/nftables.conf` les deux lignes (commentaire + règle) « SSH temporaire », puis valider et recharger :


```bash
sudo nano /etc/nftables.conf
sudo nft -c -f /etc/nftables.conf
sudo systemctl reload nftables
sudo nft list ruleset | grep -c 'dport 22 '
```
*La dernière commande doit afficher `0`.*  

---
###		3. Fail2Ban

```bash
sudo nano /etc/fail2ban/jail.d/HOSTNAME.local
```
```bash
[DEFAULT]
backend = systemd
banaction = nftables-multiport
banaction_allports = nftables-allports
ignoreip = 127.0.0.1/8 ::1 LAN_CIDR

[sshd]
enabled = true
port = PORT
maxretry = 4
bantime = 1h
findtime = 10m
```
```bash
sudo systemctl enable --now fail2ban
```
```bash
sudo systemctl restart fail2ban
```
```bash
sudo fail2ban-client status sshd
```
---
###		4. Contrôle

| Vérification | Commande | Attendu |
|--------------|----------|---------|
| Pare-feu actif | `sudo nft list ruleset` | table `inet filter`, chaîne `input` avec `policy drop`, règle `tcp dport PORT` |
| Pare-feu actif au boot | `systemctl is-enabled nftables` | `enabled` |
| Syntaxe valide | `sudo nft -c -f /etc/nftables.conf` | aucune sortie |
| Filet de sécurité retiré | `systemctl list-timers \| grep nft-rollback` | aucune sortie |
| Règle SSH temporaire supprimée | `sudo nft list ruleset \| grep -c 'dport 22 '` | `0` |
| SSH écoute sur le bon port | `sudo ss -tlnp \| grep sshd` | `:PORT` seulement |
| Config effective | `sudo sshd -T \| grep -Ei 'port\|passwordauth\|permitroot\|allowusers'` | valeurs du drop-in |
| Mot de passe refusé | `ssh -p PORT -o PubkeyAuthentication=no USER@HOSTNAME` | `Permission denied (publickey)` |
| Fail2ban | `sudo fail2ban-client status sshd` | jail `sshd` active |
| Table Fail2ban conservée après rechargement | `sudo systemctl reload nftables && sudo nft list tables` | `inet filter` **et** `inet f2b-table` |
| Persistance | `sudo reboot`, puis reconnexion | accès par clé OK, règles nftables présentes |

**Retour arrière si SSH est inaccessible** :  
attendre 5 minutes (le filet de sécurité retire la table) ou brancher écran + clavier, se connecter localement, puis `sudo nft delete table inet filter`.  
Pour SSH : supprimer `/etc/ssh/sshd_config.d/00-HOSTNAME.conf` puis `sudo systemctl restart ssh`.

---

##	💾 HDD & PARTAGE :
###		1. NTFS
*Identifier la partition*
```bash
clear && lsblk -f && sudo blkid
```
*Noter l'`UUID` de la partition concernée*

---
*Montage en lecture/écriture*
```bash
sudo mkdir -p /mnt/data2to
sudo cp -a /etc/fstab /etc/fstab.ORI
sudo nano /etc/fstab
```
⚠️ **Remplacez** `UUID_NTFS` par **votre UUID**

```text
UUID=UUID_NTFS  /mnt/data2to  ntfs-3g  rw,uid=1000,gid=1000,umask=022,big_writes,nofail,x-systemd.device-timeout=10  0  0
```

- `rw` : lecture et écriture activées.
- `uid=1000,gid=1000` : l'utilisateur `USER` (uid 1000) est propriétaire de tout le système de fichiers.
- `umask=022` : `USER` a les droits lecture/écriture, et les autres utilisateurs du Pi (dont `minidlna`) ont les droits de lecture.
- `big_writes` : active les écritures par blocs de 128 Ko au lieu de 4 Ko, réduisant drastiquement la charge CPU sur le Pi 4 et améliorant le débit.
- `nofail` + `device-timeout=10` : le Pi démarre sans bloquer même si le disque USB est débranché.

```bash
sudo systemctl daemon-reload
sudo mount /mnt/data2to
findmnt /mnt/data2to
ls /mnt/data2to
# Test d'ecriture et suppression
touch /mnt/data2to/test_ecriture && rm -v /mnt/data2to/test_ecriture
```

---
###		2. Samba
```bash
sudo cp -a /etc/samba/smb.conf /etc/samba/smb.conf.ORI
sudo nano /etc/samba/smb.conf
```
```samba
[global]
   workgroup = WORKGROUP
   server string = HOSTNAME
   server role = standalone server
   log file = /var/log/samba/log.%m
   max log size = 1000
   map to guest = never
   server min protocol = SMB2
   disable netbios = yes
   hosts allow = 127.0.0.1 LAN_CIDR
   hosts deny = 0.0.0.0/0
   load printers = no
   printing = bsd
   printcap name = /dev/null
   disable spoolss = yes

[Data2To]
   path = /mnt/data2to
   comment = Disque 2 To (lecture/ecriture)
   browseable = yes
   read only = no
   writable = yes
   guest ok = no
   valid users = USER
   force create mode = 0664
   force directory mode = 0775
```
```bash
testparm -s
sudo smbpasswd -a $USER
sudo systemctl enable --now smbd
sudo systemctl restart smbd
```
###		3. MiniDLNA
```bash
sudo cp -a /etc/minidlna.conf /etc/minidlna.conf.ORI
sudo nano /etc/minidlna.conf
```
```bash
user=minidlna
media_dir=A,/mnt/data2to/Musique
media_dir=V,/mnt/data2to/Vidéos
media_dir=P,/mnt/data2to/Pictures
db_dir=/var/cache/minidlna
log_dir=/var/log/minidlna
port=8200
friendly_name=HOSTNAME
inotify=yes
album_art_names=Cover.jpg/cover.jpg/AlbumArtSmall.jpg/albumartsmall.jpg
album_art_names=AlbumArt.jpg/albumart.jpg/Album.jpg/album.jpg
album_art_names=Folder.jpg/folder.jpg/Thumb.jpg/thumb.jpg
```
```bash
sudo systemctl enable --now minidlna
sudo systemctl restart minidlna
```
###		4. Contrôle

| Vérification | Commande | Attendu |
|--------------|----------|---------|
| Montage `rw` | `findmnt -o TARGET,FSTYPE,OPTIONS /mnt/data2to` | `ntfs-3g`, options `rw`, `big_writes` |
| Écriture locale | `clear && touch /mnt/data2to/t && rm -v /mnt/data2to/t` | succès (aucun message d'erreur) |
| Samba valide | `testparm -s` | aucune erreur, `read only = No` sur `[Data2To]` |
| Samba accessible | depuis Windows : `\\HOSTNAME\Data2To` | demande d'identifiants, lecture ET écriture OK |
| Invité refusé | accès sans identifiants | refus |
| MiniDLNA | `systemctl status minidlna` ; page `http://IP:8200` | service actif, nombre de fichiers > 0 |
| Démarrage | `sudo reboot`, puis `findmnt /mnt/data2to` | disque remonté |
| Démarrage sans disque | débrancher le disque, redémarrer | le Pi démarre quand même |

---

##	🤖 COEUR IoT :
###		1. Mosquitto
```bash
sudo nano /etc/mosquitto/conf.d/HOSTNAME.conf
```
```bash
listener 1883
allow_anonymous false
password_file /etc/mosquitto/passwd
```
```bash
sudo mosquitto_passwd -c /etc/mosquitto/passwd USER
sudo chown mosquitto:mosquitto /etc/mosquitto/passwd
sudo chmod 640 /etc/mosquitto/passwd
sudo systemctl enable --now mosquitto
sudo systemctl restart mosquitto

```

**Contrôle** :

```bash
systemctl status mosquitto
mosquitto_pub -h localhost -t test/HOSTNAME -m "ok"
```
La commande ci-dessus doit être **refusée** (pas d'authentification). Avec identifiants :

```bash
mosquitto_sub -h localhost -t test/HOSTNAME -u USER -W 5 &
mosquitto_pub -h localhost -t test/HOSTNAME -m "ok" -u USER
```
Chaque commande avec `-u` demande le mot de passe au clavier (option `-P` à éviter).
###		2. Node-RED
####		1. Installation
```bash
bash <(curl -sL https://github.com/node-red/linux-installers/releases/latest/download/install-update-nodered-deb)
sudo systemctl enable nodered.service
```

---
####		2. configuration
```bash
cp /home/USER/.node-red/settings.js /home/USER/.node-red/settings.js.ORI
echo "NODE_RED_CREDENTIAL_SECRET=$(openssl rand -hex 32)" | sudo tee /etc/default/nodered-HOSTNAME
sudo nano ~/.node-red/settings.js
```

```
uiHost: "::",
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nodered.service
```

---
###		3. Contrôle

| Vérification | Commande | Attendu |
|--------------|----------|---------|
| Mosquitto actif | `systemctl is-active mosquitto` | `active` |
| Anonyme refusé | `mosquitto_pub -h localhost -t t -m x` | `Connection error` / refus |
| MQTT authentifié | cf. ci-dessus | message reçu |
| Node-RED actif | `systemctl is-active nodered` | `active` |
| Éditeur protégé | navigateur : `http://HOSTNAME_IP:1880` | écran de connexion |
| Ports | `sudo ss -tlnp \| grep -E '1883\|1880'` | en écoute |

---


##	🌍 STACK DEV & WEB :

###		1. Git

```bash
sudo apt-get install -y unzip
git config --global user.name "NAME"
git config --global user.email "EMAIL"
git config --global init.defaultBranch main
git config --global color.ui auto
ssh-keygen -t ed25519 -C "USER-github"
cat ~/.ssh/id_ed25519.pub
```
*Coller la clé publique sur GitHub, puis*  
`ssh -T git@github.com`.

---
### 		2. Apache2 + PHP 8.4

```bash
sudo a2enmod proxy_fcgi setenvif rewrite
sudo a2enconf php8.4-fpm
sudo systemctl enable --now apache2 php8.4-fpm
sudo systemctl reload apache2
```
```bash
sudo tee /etc/php/8.4/fpm/conf.d/90-HOSTNAME.ini > /dev/null <<'EOF'
memory_limit = 256M
post_max_size = 32M
upload_max_filesize = 64M
display_errors = Off
log_errors = On
EOF
```
```bash
sudo systemctl restart php8.4-fpm
```
###		3. MariaDB

```bash
sudo systemctl enable --now mariadb
sudo mariadb
```

```sql
CREATE DATABASE projet_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER '<USER>'@'localhost' IDENTIFIED BY '<MOT_DE_PASSE_FORT>';
GRANT ALL PRIVILEGES ON projet_db.* TO '<USER>'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
###		4. Adminer

```bash
sudo mkdir -p /var/www/adminer
sudo wget -q -O /var/www/adminer/index.php https://www.adminer.org/latest.php
sudo chown -R root:root /var/www/adminer
sudo chmod 644 /var/www/adminer/index.php
```
```bash
sudo tee /etc/apache2/sites-available/adminer.conf > /dev/null <<'EOF'
Listen 8001
<VirtualHost *:8001>
    ServerName HOSTNAME-adminer
    DocumentRoot /var/www/adminer
    <Directory /var/www/adminer>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require ip 127.0.0.1 LAN_CIDR
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/adminer_error.log
    CustomLog ${APACHE_LOG_DIR}/adminer_access.log combined
</VirtualHost>
EOF
```
```bash
sudo apache2ctl configtest
sudo a2ensite adminer.conf
sudo systemctl reload apache2
```
###		5. Symfony CLI,

```bash
curl -fsSL 'https://dl.cloudsmith.io/public/symfony/stable/setup.deb.sh' -o symfony-setup.sh
less symfony-setup.sh
sudo bash symfony-setup.sh
sudo apt-get install -y symfony-cli
rm -v symfony-setup.sh
symfony check:requirements
```
###		6. Contrôle

| Vérification | Commande | Attendu |
|--------------|----------|---------|
| Apache | `curl -sI http://localhost` | `HTTP/1.1 200` |
| PHP-FPM relié à Apache | créer `/var/www/html/info.php` avec `<?php phpinfo();`, `curl -s http://localhost/info.php \| grep -i 'Server API'` | `FPM/FastCGI` — **supprimer ensuite le fichier** |
| PHP | `php -v` | `8.4.x` |
| MariaDB | `sudo mariadb -e "SELECT VERSION();"` | version affichée |
| MariaDB non exposée | `sudo ss -tlnp \| grep 3306` | `127.0.0.1` seulement |
| Adminer | navigateur : `http://HOSTNAME_IP:8001` | page de connexion ; accès refusé hors LAN |
| Connexion applicative | `mariadb -u USER -p projet_db -e "SELECT 1;"` | `1` |
| Composer | `composer --version` | version affichée |
| Symfony | `symfony check:requirements` | « all requirements are met » |
| Python | `python3 -m venv /tmp/t && /tmp/t/bin/python -V` | `3.x` |
| Node | `node -v` | version affichée |

---

##	🔎 CONTRÔLE FINAL :

```bash

sudo nft list ruleset
systemctl is-active ssh fail2ban nftables smbd minidlna mosquitto nodered apache2 php8.4-fpm mariadb
sudo ss -tlnp
free -h
```

##	💗 BONUS :
###		1. Sauvegarde
```bash
clear;df -h / && df -h /mnt/data2to  
sudo apt clean
clear;df -h / && df -h /mnt/data2to  
mkdir -p ~/backup
sudo tar czf ~/backup/etc-HOSTNAME-$(date +%F).tgz /etc/ssh/sshd_config.d /etc/nftables.conf /etc/fail2ban/jail.d /etc/samba/smb.conf /etc/minidlna.conf /etc/mosquitto/conf.d /etc/apache2/sites-available /etc/php/8.4/fpm/conf.d/90-HOSTNAME.ini
sudo mariadb-dump --all-databases > ~/backup/mariadb-$(date +%F).sql
```
###		2. Aliases  
`nano .bash_aliases`  
```
alias PERFS="clear;df -h / && df -h /mnt/data2to && free -h"
alias Rfresh="source ~/.bashrc"
alias cls="clear;ls"
alias clsa="clear;ls -lshaG"
alias doc="cd ~/Documents/;clsa"
alias BACKUP="sudo tar czf ~/backup/etc-HOSTNAME-\$(date +%F).tgz /etc/ssh/sshd_config.d /etc/nftables.conf /etc/fail2ban/jail.d /etc/samba/smb.conf /etc/minidlna.conf /etc/mosquitto/conf.d /etc/apache2/sites-available /etc/php/8.4/fpm/conf.d/90-HOSTNAME.ini && sudo mariadb-dump --all-databases > ~/backup/mariadb-\$(date +%F).sql"
```
`source ~/.bashrc`  
###		3. Ajouter un utilisateur
```bash
sudo adduser USER
sudo usermod -a -G adm,dialout,cdrom,sudo,audio,video,plugdev,games,users,input,netdev,gpio,i2c,spi,minidlna USER
sudo cp -Rpv /etc/sudoers.d/010_pi-nopasswd /etc/sudoers.d/010_USER-nopasswd
sudo nano /etc/sudoers.d/010_USER-nopasswd
```
```ini
USER ALL=(ALL:ALL) ALL
```
`sudo reboot`  
###		4. Supprimer un utilisateur
```bash
sudo pkill -u USER
sudo deluser --remove-home USER
sudo rm -Rfv /etc/sudoers.d/010_USER-nopasswd
```
