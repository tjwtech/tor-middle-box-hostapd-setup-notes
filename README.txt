Read: https://trac.torproject.org/projects/tor/wiki/doc/TransparentProxy#AnonymizingMiddlebox


Install: hostapd dnsmasq tor

Install latest stable tor from repo: https://2019.www.torproject.org/docs/debian.html.en

Set IP address for wlan0: (This change is only temporary, consider using somthing like netplan to apply on startup)
ip addr add 192.168.1.1/24 dev wlan0

dnsmasq:
port=0                                    # Disable dns server (All dns lookups are handeled by tor)
interface=wlan0                           # hostapd AP interface
dhcp-range=192.168.1.50,192.168.1.150,24h # DHCP client pool (100 available addresses with 24h leases)
dhcp-option=option:dns-server,192.168.1.1 # DNS server for all DHCP clients (make sure this is the IP address of wlan0)

hostapd: (Remove the comments, as hostapd sometimes struggles with them)(You may need to set DAEMON_CONF= in /etc/init.d/hostapd or /etc/default/hostapd)
ssid=Tor Free WiFi              # WiFi name
interface=wlan0                 # AP interface
hw_mode=g                       # g simply means 2.4GHz band
channel=6                       # WiFi channel
ieee80211d=1                    # limit the frequencies used to those allowed in the country
country_code=US                 # the country code
ieee80211n=1                    # 802.11n support
driver=nl80211                  # Driver recomended for your WiFi card
logger_syslog=0
logger_syslog_level=0
wmm_enabled=1                   # QoS support (https://wiki.gentoo.org/wiki/Hostapd#Sample_configurations)
preamble=1                      # Enable short preamble for frames sent at 2 Mbps, 5.5 Mbps, and 11 Mbps to improve network performance (https://w1.fi/cgit/hostap/plain/hostapd/hostapd.conf)
noscan=1                        # No idea, was recomended for my WiFi card by armbian
ctrl_interface=/var/run/hostapd
ctrl_interface_group=0

tor:
Log notice file /var/log/tor/notices.log # Good for troubleshooting tor
VirtualAddrNetworkIPv4 10.192.0.0/10
AutomapHostsOnResolve 1
TransPort 192.168.1.1:9040
DNSPort 192.168.1.1:5353
# You may also wish to setup bandwidth limits to stop users from saturating your network

Setup iptables rules for transperant proxying: (These changes are only temporary, consider using iptables-persistent to apply these on startup)
######################################
_trans_port="9040" # Tor's TransPort
_inc_if="wlan0"    # hostapd AP interface

iptables -F
iptables -t nat -F

iptables -t nat -A PREROUTING -i $_inc_if -p udp --dport 53 -j REDIRECT --to-ports 5353
iptables -t nat -A PREROUTING -i $_inc_if -p udp --dport 5353 -j REDIRECT --to-ports 5353
iptables -t nat -A PREROUTING -i $_inc_if -p tcp --syn -j REDIRECT --to-ports $_trans_port
######################################

Stop NetworkManager from managing wlan0:
nano /etc/NetworkManager/NetworkManager.conf
--------------------------------------
[keyfile]
unmanaged-devices=interface-name:wlan0
######################################
