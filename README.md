# vpn-gateway
sample file and script for crate gateway vpn

---------------------------------------------------
VPN Gateway w/ Kill Switch

Set Static IP address
/etc/netplan/50-cloud-init.yaml 

network:
    ethernets:
        eth0:
            addresses: [#.#.#.#/24]
            gateway4: #.#.#.#
            nameservers:
                addresses: [1.1.1.1,1.0.0.1]
            dhcp4: false
            optional: true
    version: 2

~~~~~~~~~~~~~~~~~~~~


Install Programs
sudo apt-get install openvpn openssh-server unzip

~~~~~~~~~~~~~~~~~~~~

Download OVPN Config files (NordVPN)
cd /etc/openvpn
sudo wget https://downloads.nordcdn.com/configs/archives/servers/ovpn.zip
sudo unzip ovpn.zip

~~~~~~~~~~~~~~~~~~~~

/etc/openvpn/connect.sh
sudo openvpn --config "/etc/openvpn/ovpn_udp/CC####.nordvpn.com.udp.ovpn" --auth-user-pass /etc/openvpn/auth.txt

~~~~~~~~~~~~~~~~~~~~

/etc/openvpn/auth.txt
Username@email.com
Password

~~~~~~~~~~~~~~~~~~~~

/etc/openvpn/iptables.sh
#!/bin/bash
# Flush
iptables -t nat -F
iptables -t mangle -F
iptables -F
iptables -X

# Block All
iptables -P OUTPUT DROP
iptables -P INPUT DROP
iptables -P FORWARD DROP

# allow Localhost
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Make sure you can communicate with any DHCP server
iptables -A OUTPUT -d 255.255.255.255 -j ACCEPT
iptables -A INPUT -s 255.255.255.255 -j ACCEPT

# Make sure that you can communicate within your own network
iptables -A INPUT -s #.#.#.#/24 -d #.#.#.#/24 -j ACCEPT
iptables -A OUTPUT -s #.#.#.#/24 -d #.#.#.#/24 -j ACCEPT

# Allow established sessions to receive traffic:
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow TUN
iptables -A INPUT -i tun+ -j ACCEPT
iptables -A FORWARD -i tun+ -j ACCEPT
iptables -A FORWARD -o tun+ -j ACCEPT
iptables -t nat -A POSTROUTING -o tun+ -j MASQUERADE
iptables -A OUTPUT -o tun+ -j ACCEPT

# allow VPN connection
iptables -I OUTPUT 1 -p udp --destination-port 1194 -m comment --comment "Allow VPN connection" -j ACCEPT

# Block All
iptables -A OUTPUT -j DROP
iptables -A INPUT -j DROP
iptables -A FORWARD -j DROP

# Log all dropped packages, debug only.

iptables -N logging
iptables -A INPUT -j logging
iptables -A OUTPUT -j logging
iptables -A logging -m limit --limit 2/min -j LOG --log-prefix "IPTables general: " --log-level 7
iptables -A logging -j DROP

echo "saving"
iptables-save > /etc/iptables.rules
echo "done"
#echo 'openVPN - Rules successfully applied, we start "watch" to verify IPtables in realtime (you can cancel it as usual CTRL + c)'
#sleep 3
#watch -n 0 "sudo iptables -nvL"

~~~~~~~~~~~~~~~~~~~~

Enable IP Forwarding
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"

~~~~~~~~~~~~~~~~~~~~

Create rc.local service
/etc/systemd/system/rc-local.service

[Unit]
Description=/etc/rc.local Compatibility
ConditionPathExists=/etc/rc.local


[Service]
Type=forking
ExecStart=/etc/rc.local start
TimeoutSec=0
StandardOutput=tty
RemainAfterExit=yes
SysVStartPriority=99


[Install]
WantedBy=multi-user.target

~~~~~~~~~~~~~~~~~~~~

Create rc.local script
/etc/rc.local
#!/bin/sh -e
sudo bash /etc/openvpn/iptables.sh &
sleep 10
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
sudo bash /etc/openvpn/connect.sh &

exit 0

~~~~~~~~~~~~~~~~~~~~

TO TEST

sudo bash /etc/openvpn/iptables.sh
sudo bash /etc/openvpn/connect.sh

Point another device’s default gateway at your VPN server. Ping google.com

CTRL-C to kill OpenVPN

Ping should stop on remote box.

If all is working, enable the service and reboot. IPTables are assigned each boot, and then the VPN connects. This allows troubleshooting or maintenance by commenting out rc.local calls and boot the server without IPTables restrictions.

~~~~~~~~~~~~~~~~~~~~

Enable/Start rc.local service
sudo systemctl enable rc-local
sudo systemctl start rc-local.service

~~~~~~~~~~~~~~~~~~~~
