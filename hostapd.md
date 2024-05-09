##  ACCESS POINT:

### INSTALL:
sudo apt-get update  
sudo apt-get dist-upgrade  
sudo apt-get install hostapd  
sudo systemctl unmask hostapd  
sudo systemctl enable hostapd  
sudo systemctl stop hostapd

### BRIDGE:
sudo nano /etc/systemd/network/bridge-br0.netdev

    [NetDev]
    Name=br0
    Kind=bridge

sudo nano /etc/systemd/network/br0-member-usb0.network

    [Match]
    Name=eth0 usb0 lo

    [Network]
    Bridge=br0

sudo systemctl enable systemd-networkd

### DHCPCD:
sudo cp -Rpv /etc/dhcpcd.conf /etc/dhcpcd.conf.ORI  
sudo nano /etc/dhcpcd.conf

...

    denyinterface wlan0 eth0 usb0
    interface br0

sudo service dhcpcd restart

### HOSTAPD:
sudo rfkill unblock wlan  
sudo nano /etc/hostapd/hostapd.conf

    #   HOSTAPD CONF
    interface=wlan0
    bridge=br0
    country_code=US
    ssid=MySSID


    #   AUTH
    auth_algs=1
    wpa_passphrase=YOUPI
    wpa=2
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=TKIP
    rsn_pairwise=CCMP
    macaddr_acl=0


    #   2.4 GHz settings
    #hw_mode=g
    #channel=7
    ieee80211h=1
    ieee80211d=1
    ieee80211n=1
    require_ht=1
    ht_capab=[HT40+][SHORT-GI-20][SHORT-GI-40][MAX-AMSDU-3839][DSSS-CCK-40]


    #   5 GHz settings
    hw_mode=a
    channel=36
    ieee80211ac=1
    require_vht=1
    vht_capab=[MAX-MPDU-3895][SHORT-GI-80][SU-BEAMFORMEE]
    vht_oper_chwidth=1
    vht_oper_centr_freq_seg0_idx=42

    wme_enabled=1
    wmm_enabled=1
    max_num_sta=2
    ignore_broadcast_ssid=0

    logger_stdout=-1
    logger_stdout_level=4

sudo reboot -h 0