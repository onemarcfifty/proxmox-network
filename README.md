# proxmox-network
Scripts and Stuff around the Proxmox Network

Here is how to build the Virtual Network in Proxmox for MPTCP Test lab
as outlined in this video

![Network Diagram](/images/Diagram1.jpg)
Format: ![Alt Text](url)




Setup Details and Commands that I issued on the machines

1.) Setting up the network
==========================

PVE-Network-Add Linux Bridge

(vmbr0 and vmbr1 are my physical ones)

The bridge between the perimeter router and the shapers:

vmbr6 - 192.168.56.2/24 (set it to 192.168.1.2/24 if you don't want to change OpenWrt's LAN IP)

The bridges after the shapers: (slightly different in the video but it makes more sense like this) :

vmbr7 - 10.7.0.2/24
vmbr8 - 10.8.0.2/24
vmbr9 - 10.9.0.2/24

The outmost right bridge (between OMR and the client):

vmbr10 - 192.168.100.2/24

(this is because OMR by default will have 192.168.100.1)

2.) OpenWrt Perimeter Router
============================

1 CPU, 512MB RAM. Scrap the disk.
net0: vmbr6, VIRTIO
net1: vmbr0 (physical), VIRTIO

One-Liner to execute on the Proxmox PVE console to download the OpenWrt Image, unzip and import the disk to the OpenWrt VM:

wget -O - "OPENWRTURL" | gunzip -c >openwrt.img ; qm importdisk 102 openwrt.img local-lvm

(replace OPENWRTURL with the URL of the OpenWrt generix ext4 COMBINED image)
(replace 102 with the ID of your OpenWrt VM)
(replace local-vm with the storage where you want your disk to be located)

scrap the original disk: Select the OpenWrt VM in the Proxmox UI, select hardware, select the original disk, click "detach", then select it again, cick "Remove". Select the newly imported disk, select "Edit", chose IDE controler - that attaches it.

Inside OpenWrt - changing the IP

`#show the ip addresses:`

`ip addr`

`#change the lan address:`

`uci show network.lan`

`uci set network.lan.ipaddr=192.168.56.1`

`uci commit`

`reboot`

3.) Shaper Machines
===================

Create CT - Debian 10 template - 8 GB Disk, 1 Core, 512 MB RAM
eth0 = vmbr6, dhcp (address provided by perimeter router)

shaper1: eth1 = vmbr7, fix, 10.7.0.1
shaper2: eth1 = vmbr8, fix, 10.8.0.1
shaper3: eth1 = vmbr8, fix, 10.9.0.1

enabling IPV4 routing: 

`sysctl net.ipv4.ip_forward=1`
(this does not survive a reboot)

make it permanent in `/etc/sysctl.conf`

remove the trailing "\#" from this line:

`\#net.ipv4.ip_forward=1`

put the following in `/etc/rc.local` (might need to chmod +x on this file)

`\#!/bin/bash`

`sysctl net.ipv4.ip_forward=1`

`iptables -t nat -A POSTROUTING -o eth0 -j  MASQUERADE`

(the iptables command enables NAT/Masquerading to the outside)

installing software:

`apt update`

`apt install bmon isc-dhcp-server speedometer sudo`

DHCP options in /etc/dhcp/dhcpd.conf (for shaper1):

option domain-name "testnet.lan";
default-lease-time 3600; 
max-lease-time 7200;
authoritative;

subnet 10.7.0.0 netmask 255.255.255.0 {
        option routers                  10.7.0.1;
        option subnet-mask              255.255.255.0;
        option domain-search            "testnet.lan";
        range   10.7.0.100   10.7.0.120;
}

(replace all 10.7.0 with 10.8.0 for shaper2 and with 10.9.0 for shaper3)

in /etc/default/isc-dhcp-server set:

`INTERFACESv4="eth1"`

(I also set eth2 because that's a VLAN I pulled to my workstation for the sake of making the video)


`#restarting the server with`

`systemctl restart isc-dhcp-server`

`#checking the logs with`

`grep -i dhcp /var/log/syslog`

`# create the ssh keys with`

`ssh-keygen`

`# the files are in /root/.ssh/id_rsa*`

adapt `/etc/ssh/sshd_config`

(find the line AuthorizedKeysFile, uncomment it if trailing "#", append `.ssh/id_rsa .ssh/id_rsa.pub)`


4.) VPS for OpenMPTCPRouter
===========================

Details in this video: https://youtu.be/mYYoIDCWszo

Create VM - Debian 10 - Disk size 32 GB - 1 CPU, 2048 MB RAM
net0: vmbr6 (VIRTIO)

Debian Install : Install SSH Server

Install the OpenMPTCPRouter config by issuing this command

`wget -O - https://www.openmptcprouter.com/server/debian10-x86_64.sh | sh`

5. VM with MPTCP enabled Kernel
===============================

Details in this video: https://youtu.be/mYYoIDCWszo

Create VM - Debian 10 - Disk size 32 GB - 1 CPU, 2048 MB RAM
Debian Install : Install SSH Server

net0:vmbr7, VIRTIO
net1:vmbr8, VIRTIO
net2:vmbr9, VIRTIO

Install the MPTCP enabled Kernel by issuing this command 
(remove the spaces or underscores in the URLs)

wget https:/_/raw.githubusercontent.com/onemarcfifty/mptcp-tools/main/makescript.sh
chmod 755 makescript.sh
./makescript.sh
# this creates the fetch_ycarus_kernel.sh
chmod 755 fetch_ycarus_kernel.sh
./fetch_ycarus_kernel.sh
# this now creates two debian packages in the /tmp directory
apt install /tmp/llinux-image....deb
apt install /tmp/llinux-headers....deb

6. VM with OpenMPTCPRouter
==========================

(pretty much the same like installing the Perimeter Router)

1 CPU, 512MB RAM. Scrap the disk.
net0: vmbr10, VIRTIO
net1: vmbr7, VIRTIO
net2: vmbr8, VIRTIO
net3: vmbr9, VIRTIO

One-Liner to execute on the Proxmox PVE console to download the OMR Image, unzip and import the disk to the OMR VM:

wget -O - "OMRURL" | gunzip -c >openwrt.img ; qm importdisk 102 omr.img local-lvm

(replace OMRURL with the URL of the OMR generix ext4 COMBINED image)
(replace 102 with the ID of your OMR VM)
(replace local-vm with the storage where you want your disk to be located)

Sample OMR URL (remove spaces):

https:   //download.openmptcprouter.com/release/v0.57.3/x86_64/targets/x86/64/openmptcprouter-v0.57.3-r0+15225-bfc433efd4-x86-64-generic-ext4-combined.img.gz

Inside OMR - setting the wan interfaces (towards the shapers) to dhcp:

uci set network.wan1.proto=dhcp
uci set network.wan2.proto=dhcp
uci set network.wan3.proto=dhcp
uci commit
/etc/init.d/network restart
#(or reboot of course)

7. MATE client (utmost right)
=============================

Create CT, Debian 10, 8 GB Disk, 1 CPU, 2048 MB RAM
net0 = vmbr10 (VIRTIO), dhcp
net1 = vmbr0 (either static or remove default gateway in the client machine to make sure the client is not routing through this inteerface but rather through the shapers)

# create non-root-user
adduser marc
usermod -a -G sudo marc

# install software
apt update
apt install mate x2go sudo firefox-esr nodejs git

get the installer from https://wiki.x2go.org/doku.php/download:start

download the Qdisc scheduler Web Interface from my github https://github.com/onemarcfifty/mptcp-tools

If you don't have OpenMPTCPRouter installed, then do this on the MPTCP client machine (5. VM with MPTCP enabled Kernel)

(remove spaces and/or underscores)

git clone https:/_/github.com/onemarcfifty/mptcp-tools.git
cd mptcp-tools
cd nodejs
# adapt the qdiscs.js file
# download the ssh keys (id_rsa) from the shapers and name them id_shaper1, id_shaper2, id_shaper3
chmod 600 id_shaper1
chmod 600 id_shaper2
chmod 600 id_shaper3
node qdiscs.js
# browse to localhost:8080


Marc's channel on youtube: https://www.youtube.com/channel/UCG5Ph9Mm6UEQLJJ-kGIC2AQ
Marc on Twitter:  https://twitter.com/onemarcfifty
Marc on Facebook: https://www.facebook.com/onemarcfifty/
Marc on Reddit: https://www.reddit.com/user/onemarcfifty
Chat with me on Discord: https://discord.com/invite/DXnfBUG

Licence-free music on /  Lizenzfreie Musik von https://www.terrasound.de/lizenzfreie-musik-fuer-youtube-videos/Licence-free music on /  Lizenzfreie Musik von https://www.terrasound.de/lizenzfreie-musik-fuer-youtube-videos/