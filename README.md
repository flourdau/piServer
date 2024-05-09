#   piServer.md
    flourdau
    09 Mai 2024

## DOWNLOAD
ISO: https://www.raspberrypi.com/software/operating-systems/  
Raspberry Pi Imager: https://www.raspberrypi.com/software/  

## SETUP:
    ⚠ ATTENTION à bien remplacer USER, IP, YOUR_DOMAIN_NAME... & modifier le PORT!  

## UPDATE:
sudo raspi-config  

    # activer le i2c si horloge
    # activer le ssh, vnc, gpio...
    # hostname, ssid, pass et/ou vos paramètres perso...

sudo reboot -h 0  
sudo apt-get update  
sudo apt-get dist-upgrade  
sudo apt-get autoclean  
sudo reboot -h 0  
sudo rpi-update  
sudo reboot -h 0  
sudo apt-get install -y htop git fail2ban ntfs-3g samba samba-common-bin minidlna mosquitto hostapd raspberrypi-ui-mods rpi-imager rpi-chromium-mods rpd-wallpaper vlc

## SSH:
mkdir .ssh  
sudo nano /etc/ssh/sshd_config  

    Include /etc/ssh/sshd_config.d/*.conf
    KbdInteractiveAuthentication no
    UsePAM yes
    X11Forwarding yes
    PrintMotd no
    AcceptEnv LANG LC_*
    Subsystem       sftp    /usr/lib/openssh/sftp-server
    Port 42042
    ChallengeResponseAuthentication no
    PermitRootLogin no
    AllowUsers USER

### Depuis la machine hote
ssh-keygen  
ssh-copy-id -p44044 USER@IP  

### OU SI POWERSHELL
scp.exe -P44044 .\.ssh\id_rsa.pub USER@IP:~/.ssh/authorized_keys  
ssh -p44044 USER@IP  

### Change & Add
sudo nano /etc/ssh/sshd_config

...


    PasswordAuthentication no
    UsePAM no

## PICLOCK:
sudo i2cdetect -y 1  
echo ds3231 0x68 | sudo tee /sys/class/i2c-adapter/i2c-1/new_device  
sudo hwclock  
sudo dpkg-reconfigure tzdata  
sudo hwclock -w  
sudo nano /etc/rc.local  

### coller avant le exit 0

    sudo echo ds3231 0x68 > /sys/class/i2c-adapter/i2c-1/new_device
    sudo hwclock -s

sudo update-rc.d fake-hwclock disable  
sudo reboot -h 0  

## WIRINGPI:
git clone https://github.com/WiringPi/WiringPi  
cd WiringPi  
./build  

## MOSQUITTO:
systemctl status mosquitto  
sudo mosquitto_passwd -c /etc/mosquitto/passwd USER  
sudo nano /etc/mosquitto/mosquitto.conf  

    ...
    allow_anonymous false
    password_file /etc/mosquitto/passwd

sudo systemctl restart mosquitto

## FSTAB:  
lsblk (reccuperation du chemin)  
sudo blkid (reccuperation du UUID)  
sudo nano /etc/fstab  

...

    UUID=70FCF8C36B663AE4   /media/USER/Data2To/ ntfs-3g permissions,defaults,nofail    0   0

sudo mount /media/USER/Data2To/  
sudo reboot  

## MINIDLNA:
sudo nano /etc/default/minidlna

    START_DAEMON="yes"
    USER="USER"
    GROUP="USER"
    DAEMON_OPTS="-r"

sudo mkdir /etc/systemd/system/minidlna.service.d/  
sudo nano /etc/systemd/system/minidlna.service.d/run-as-user.conf

    [Service]
    User=USER
    Group=USER

sudo cp -Rpv /etc/minidlna.conf /etc/minidlna.conf.ORI  
sudo nano /etc/minidlna.conf

    user=USER
    media_dir=A,/var/lib/minidlna/music
    media_dir=V,/var/lib/minidlna/videos
    media_dir=P,/var/lib/minidlna/pictures
    db_dir=/media/USER/Data2To/
    port=8200
    friendly_name=NAME
    serial=123456789
    model_number=1
    inotify=yes
    album_art_names=Cover.jpg/cover.jpg/AlbumArtSmall.jpg/albumartsmall.jpg
    album_art_names=AlbumArt.jpg/albumart.jpg/Album.jpg/album.jpg
    album_art_names=Folder.jpg/folder.jpg/Thumb.jpg/thumb.jpg

sudo ln -s /media/USER/Data2To/Musique /var/lib/minidlna/music  
sudo ln -s /media/USER/Data2To/Vidéos /var/lib/minidlna/videos  
sudo ln -s /media/USER/Data2To/Pictures /var/lib/minidlna/pictures  

## SAMBA:
sudo cp -Rpv /etc/samba/smb.conf /etc/samba/smb.conf.ORI  
sudo nano /etc/samba/smb.conf

    [global]
        workgroup = YOUR_DOMAIN_NAME
        realm = MACHINENAME
        netbios name = MACHINENAME
        dns proxy = no
        log file = /var/log/samba/log.%m
        max log size = 1000
        panic action = /usr/share/samba/panic-action %d
        server role = standalone server
        passdb backend = tdbsam
        obey pam restrictions = yes
        unix password sync = yes
        passwd program = /usr/bin/passwd %u
        passwd chat = *Enter\snew\s*\spassword:* %n\n *Retype\snew\s*\spassword:* %n\n *password\supdated\ssuccessfully* .
        pam password change = yes
        map to guest = bad user
        usershare allow guests = yes
        winbind separator = /
        winbind uid = 10000-20000
        winbind gid = 10000-20000
        winbind use default domain = yes
        follow symlinks = yes
        wide links = yes
        unix extensions = no

    [Home]
        path = /home/USER/
        Comment = USER HOME Folder
        browseable = yes
        writable = yes
        create mode = 0600
        directory mask = 0700


    [Data2To]
        path = /media/USER/Data2To/
        Comment = USER Data2To Folder
        browseable = yes
        writable = yes
        create mode = 0600
        directory mask = 0700

sudo smbpasswd -a USER  
sudo systemctl restart smbd.service  

### GUI:
[Forum RaspberryPi.com](https://forums.raspberrypi.com/viewtopic.php?t=133691 "Topic détaillé")

sudo apt-get install raspberrypi-ui-mods rpi-imager rpi-chromium-mods rpd-wallpaper vlc



##  SETTINGS:
### SAV:
sudo dd bs=4M if=/dev/sdb | gzip > raspbian.img.gz

### RESTOR:
gunzip --stdout raspbian.img.gz | sudo dd bs=4M of=/dev/sdb

### WIFI:
touch ssh

nano wpa_supplicant.conf

    country=fr
    update_config=1
    ctrl_interface=/var/run/wpa_supplicant

    network={
        scan_ssid=1
        ssid="MySSID"
        psk="MyPASS"
    }

### ADD USER:
sudo adduser USER  
sudo usermod -a -G adm,dialout,cdrom,sudo,audio,video,plugdev,games,users,input,netdev,gpio,i2c,spi,minidlna USER  
sudo cp -Rpv /etc/sudoers.d/010_pi-nopasswd /etc/sudoers.d/010_USER-nopasswd  
sudo nano /etc/sudoers.d/010_USER-nopasswd

    USER ALL=(ALL) PASSWD: ALL

sudo reboot -h 0
### DEL USER:
sudo pkill -u USER  
sudo deluser -remove-home USER  
sudo rm -Rfv /etc/sudoers.d/010_USER-nopasswd