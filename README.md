# Portfolio_Two
Linux for embedded Objects 1 Portfolio 2

#--------------------------
# Group 25
# Jannes Helck
# Balazs Megyeri
#--------------------------

#--------------------------
#Installing lxc:
#--------------------------

sudo apt install lxc

#--------------------------------------------------------------------------------------------------------
# We followed the guide to enable the usage of unprivileged containers (https://help.ubuntu.com/lts/serverguide/lxc.html)
#--------------------------------------------------------------------------------------------------------

mkdir -p ~/.config/lxc
echo "lxc.id_map = u 0 100000 65536" > ~/.config/lxc/default.conf
echo "lxc.id_map = g 0 100000 65536" >> ~/.config/lxc/default.conf
echo "lxc.network.type = veth" >> ~/.config/lxc/default.conf
echo "lxc.network.link = lxcbr0" >> ~/.config/lxc/default.conf
echo "$USER veth lxcbr0 2" | sudo tee -a /etc/lxc/lxc-usernet

#--------------------------------------------------------------------------------------------------------
# Create containers (we created unprivileged containers on the RPi but when we tried to start them with the following command:
#--------------------------------------------------------------------------------------------------------

lxc-start -n C1 -F

#--------------------------------------------------------------------------------------------------------
# but we got the following error message:
#--------------------------------------------------------------------------------------------------------

Error attaching vethQ9IWDQ to lxcbr0
Quota reached
lxc-start: start.c: lxc_spawn: 1207 Failed to create the configured network.
lxc-start: start.c: __lxc_start: 1346 Failed to spawn container "C1".
lxc-start: tools/lxc_start.c: main: 366 The container failed to start.
lxc-start: tools/lxc_start.c: main: 370 Additional information can be obtained by setting the --logfile and --logpriority options.

#--------------------------------------------------------------------------------------------------------
# We were able to create privileged containers on the Pi, but when we attached to them we were not able to update the apk repo
#--------------------------------------------------------------------------------------------------------

sudo lxc-attach -n C1
/ # apk update

#--------------------------------------------------------------------------------------------------------
fetch http://dl-cdn.alpinelinux.org/alpine/v3.4/main/armhf/APKINDEX.tar.gz
ERROR: http://dl-cdn.alpinelinux.org/alpine/v3.4/main: temporary error (try again later)
WARNING: Ignoring APKINDEX.167438ca.tar.gz: No such file or directory
fetch http://dl-cdn.alpinelinux.org/alpine/v3.4/community/armhf/APKINDEX.tar.gz
ERROR: http://dl-cdn.alpinelinux.org/alpine/v3.4/community: temporary error (try again later)
WARNING: Ignoring APKINDEX.a2e6dac0.tar.gz: No such file or directory
2 errors; 16 distinct packages available
#--------------------------------------------------------------------------------------------------------
# Because we were not able to solve this issue we continued working with privileged containers on our main linux system
#--------------------------------------------------------------------------------------------------------

sudo lxc-create -n C1 -t download -- -d alpine -r 3.4 -a amd64
sudo lxc-create -n C2 -t download -- -d alpine -r 3.4 -a amd64

#----------------------------------------------------
#Install and start lighttpd webserver in C1:
#----------------------------------------------------

lxc-start -n C1
lxc-attach -n C1
lxc-attach -n C1 -- apk update
lxc-attach -n C1 -- apk add lighttpd php5 php5-cgi php5-curl php5-fpm

#------------------------------------------------------------------------------
#Uncomment "mod_fastcgi.conf" line in /etc/lighttpd/lighttpd.conf and start service:
#------------------------------------------------------------------------------
rc-update add lighttpd default
openrc

#------------------------------------------------------------------------------
#Creating /var/www/localhost/htdocs/index.php:
#------------------------------------------------------------------------------

cd /var/www/localhost/htdocs/
touch index.php
vi index.php
#----------------------------------------------------
#Copy the following into the file:
#----------------------------------------------------
<!DOCTYPE html>
<html><body>
Group 25: Jannes Helck, Balazs Megyeri
<pre>
<?php 
        // create curl resource 
        $ch = curl_init(); 
        // set url 
        curl_setopt($ch, CURLOPT_URL, "C2:8080"); 
        //return the transfer as a string 
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1); 
        // $output contains the output string 
        $output = curl_exec($ch); 
        // close curl resource to free up system resources
        curl_close($ch);
        print $output;
?>
</body></html>

#----------------------------------------------------
#Install randomness into C2: install socat
#----------------------------------------------------

sudo lxc-attach -n C2
/ # apk update
/ # apk add socat

#----------------------------------------------------
#Creating the random script in C2:
#----------------------------------------------------
/ # cd /bin
/ # touch rng.sh
/ # vi rng.sh

#--------------------------------------------------------------------------------------------------------
#Copying the content into rng.sh: (instead of bash we had to use ash as shebang)
#--------------------------------------------------------------------------------------------------------

!/bin/ash
dd if=/dev/random bs=4 count=16 status=none | od -A none -t u4

#--------------------------------------------------------------------------------------------------------
# Serving the script with socat
#--------------------------------------------------------------------------------------------------------

socat -v -v tcp-listen:8080,fork,reuseaddr exec:/bin/rng.sh

#--------------------------------------------------------------------------------------------------------
# Establishing the bridge between the two containers (following the guide: https://angristan.xyz/setup-network-bridge-lxc-net/)
#--------------------------------------------------------------------------------------------------------

apt install dnsmasq-base

sudo vim /etc/lxc/default.conf

#--------------------------------------------------------------------------------------------------------
# we replaced the content with the following lines:
#--------------------------------------------------------------------------------------------------------

lxc.network.type = veth
lxc.network.link = lxcbr0
lxc.network.flags = up
lxc.network.hwaddr = 00:16:3e:xx:xx:xx

#--------------------------------------------------------------------------------------------------------
# we created the file /etc/default/lxc-net and put this in it:
#--------------------------------------------------------------------------------------------------------

USE_LXC_BRIDGE="true"

#--------------------------------------------------------------------------------------------------------
# restarting:
#--------------------------------------------------------------------------------------------------------

systemctl restart lxc-net
systemctl status lxc-net


#--------------------------------------------------------------------------------------------------------
# As next step we made the ip-adresses of the containers static: we created /etc/lxc/dhcp.conf and put our Container configuration inside:
#--------------------------------------------------------------------------------------------------------

dhcp-host=C1,10.0.3.11
dhcp-host=C2,10.0.3.12

#--------------------------------------------------------------------------------------------------------
# We uncommented the followig line in /etc/default/lxc-net:
#--------------------------------------------------------------------------------------------------------

LXC_DHCP_CONFILE=/etc/lxc/dhcp.conf

#--------------------------------------------------------------------------------------------------------
# After that we restarted lxc-net and the containers:
#--------------------------------------------------------------------------------------------------------

systemctl restart lxc-net
lxc-stop -n C1 && lxc-start -n C1
lxc-stop -n C2 && lxc-start -n C2

#--------------------------------------------------------------------------------------------------------
# To make container C1 accessable from outside we used iptables with the following command:
#--------------------------------------------------------------------------------------------------------

sudo iptables -t nat -A PREROUTING -i <host_nic> -p tcp --dport <host_port> -j DNAT --to-destination <ct_ip>:<ct_port>

    host_nic: wlp61s0
    ct_ip: 10.0.3.11
    host_port: 80
    ct_port: 80
    
    
sudo iptables -t nat -A PREROUTING -i wlp61s0 -p tcp --dport 80 -j DNAT --to-destination 10.0.3.11:80

#--------------------------------------------------------------------------------------------------------
# Now we were able to index.php file on C1 (http://10.0.3.11) with our local web-browser and to see the created random numbers
#--------------------------------------------------------------------------------------------------------
