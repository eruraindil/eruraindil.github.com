---
layout: post
title:  "Building a Router (Out of an Old Computer)"
date:   2014-04-06 11:43:00
menu: blog
---

It was time for a new project this weekend. Looking around my desk and box of tech goodies two things caught my attention:

 * A DLink PCI Ethernet card, that I acquired sometime during my life but never had a purpose for.
 * An old WinXP era desktop computer that I picked up from my Aunt when she upgraded to a laptop about 8 years ago.

I decided to do what any sane person would do, and build my own router because let's be honest, that little box that sits on my desk is m&eacute;diocre at best. It needs frequent restarting, it's DHCP is only adequate, and doesn't let me configure DNS.

I am going to go through the steps I followed to complete this project, you can jump to them [here](#installation). Sources will be listed at the bottom of the page. But first, a little prep.

## Prep

### What is this old computer running on?
 1. I started up the computer and was welcomed by: ![Ubuntu 8.10 boot splash](/assets/boot-screen.jpg "source: http://ubuntuforums.org/showthread.php?t=225458")
 1. To which I thought: ![Now that's something I haven't seen in a long time.](/assets/meme-ubuntu-8-10-boot-screen.jpg)
 1. Fsck started up, because yeah it *has* been over 120 days since last boot, churned for a bit and then errors filled the screen and the hard drive crashed. Oh well, there probably wasn't anything important on it anyway.
 1. Into my box of tech goodies again to get another IDE hard drive.
 1. While I was installing the new hard drive, I also popped the PCI Ethernet card in, and started torrenting Ubuntu 14.04 Server Edition.
 	* I know there's a lot of controversy over the direction that Ubuntu is going recently, and on the desktop edition I would agree, but as far as I'm concerned, the server edition is solid, I like the community resources, and APT package management.

### Base Installation
 1. Before I installed the server OS, I wanted to make sure that the hardware supported the Ethernet card, so I installed a copy of Xubuntu 13.10 onto a USB stick and started up the box with it's new HDD and Ethernet card. The BIOS has no option to boot from USB. Ah yes, it's 10 years old.
 1. Instead of wasting 2 CDs on disc images which would be obsolete in 6 months, I burned a copy of [Plop Boot Manager](http://www.plop.at/en/bootmanager/index.html) onto CD so I could always have the option of booting from USB in the future.
 1. From the Live CD I confirmed that the PCI Ethernet card worked, and then installed the new 14.04 edition from USB.

<a id="installation"></a>
## Router Installation

At this point the fun begins. If you're following this as a guide, you should have a modern installation of Ubuntu Server Edition (Debian would probably work too, but no guarantees) with 2 Ethernet cards.

> NOTE: This document is provided "as is" without warranty of any kind. I take no responsibility for any loss or damage arising from the use of this document.

### Network Design

 * My network domain is `lan.example.com`.
 * My router's host name is `pegasus`.
 * `eth0` is my external facing network card. My ISP expects the following settings:
 	* IP: `192.168.1.50`
 	* Subnet: `255.255.255.0`
 	* Gateway: `192.168.1.1`
 	* DNS: `192.168.1.1`
 * `eth1` is my internal facing network card. My router will use the following settings:
 	* IP: `192.168.0.1`
 	* Subnet: `255.255.255.0`
 	* Broadcast: `192.168.0.255`

### Install the stuff

	$ sudo apt-get install bind9 isc-dhcp-server ufw fail2ban

 * `bind9` is the DNS server.
 * `isc-dhcp-server` is the DHCP server.
 * `ufw` is a firewall (actually just a frontend of iptables).
 * `fail2ban` is a monitoring program which will ban IPs if they try to bruteforce common programs on your server.

At this point, you should unplug from the internet, or you are going to confuse your existing network.

### Configure Hostname
Edit `/etc/hosts`

{% highlight bash %}

127.0.0.1       localhost
127.0.1.1       pegasus
192.168.0.1     pegasus.lan.example.com pegasus

{% endhighlight %}

### Configure Network Interfaces

Edit `/etc/network/interfaces`

{% highlight bash %}

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet static
	address 192.168.1.50
	netmask 255.255.255.0
	gateway 192.168.1.1

# The internal network interface
auto eth1
iface eth1 inet static
	address 192.168.0.1
	network 192.168.0.0
	netmask 255.255.255.0
	broadcast 192.168.0.255

{% endhighlight %}

### Configure DNS
Edit `/etc/bind/named.conf.options`

{% highlight bash %}

options {
	directory "/var/cache/bind";

	forwarders {
		192.168.1.1;
	};

	dnssec-validation auto;

	auth-nxdomain no;    # conform to RFC1035

	listen-on-v6 { any; };

	allow-query {
		192.168.0/24;
		127.0.0.1;
	};

	allow-transfer {
		192.168.0/24;
		127.0.0.1;
	};
};

{% endhighlight %}

#### Dynamic Updating DNS

To get DHCP to update DNS automatically, you have to set up a cryptographic hash for the DNS and DHCP services to share.

	$ cd /etc/bind/
	$ sudo /usr/sbin/rndc-confgen -a

This will create a file `/etc/bind/rndc.key`.

#### Create DNS Zones

Edit `/etc/bind/named.conf.local`

{% highlight bash %}

include "/etc/bind/rndc.key";

zone "lan.example.com" {
	type master;
	file "/var/lib/bind/db.lan.example.com";
	allow-update { key rndc-key; };
};

zone "0.168.192.in-addr.arpa" {
	type master;
	notify no;
	file "/var/lib/bind/db.192";
	allow-update { key rndc-key; };
};

{% endhighlight %}

Copy sample zone files into `/var/lib/bind/` and customise.

	$ cd /etc/bind/
	$ sudo cp db.local db.127 /var/lib/bind/
	$ cd /var/lib/bind/
	$ sudo chown bind:bind *
	$ mv db.local db.lan.example.com
	$ mv db.127 db.192

Edit `/var/lib/bind/db.lan.example.com` to provide forward domain name to IP address translation.

{% highlight bash %}

$ORIGIN .
$TTL 604800     ; 1 week
lan.example.com	IN	SOA		pegasus.lan.example.com. root.lan.example.com. (
								2014040501 ; serial
								604800     ; refresh (1 week)
								86400      ; retry (1 day)
								2419200    ; expire (4 weeks)
								604800     ; minimum (1 week)
)
						NS      pegasus.lan.example.com.
						A       192.168.0.1
$ORIGIN lan.example.com.
pegasus					A		192.168.0.1 	; Authority
galactica				A		192.168.0.10 	; General
xbmc					A		192.168.0.12 	; XBMC client
raptor					CNAME	galactica		; SQL server
viper					CNAME	galactica		; HTTP server
cic 					CNAME	galactica		; Foreman puppet server

{% endhighlight %}

 * `$ORIGIN .`
	* `lan.example.com` is my domain name.
	* `pegasus.lan.example.com` is the FQDN of my router.
	* `root.lan.example.com` is the email address `root@lan.example.com` which will contact the admin user of the router.
	* `2014040501` is the date April 5th, 2014 + 01 representing revision 1. Every time an update is made to this file, the serial must be changed so a common way is to use the date + a revision number (in case you update the zone more than once a day).
	* `NS` the FQDN of your DNS server (this server).
	* `A` the DNS server's IP.
 * `$ORIGIN lan.example.com.`
 	* Here is where I specify the addresses, and aliases of my other servers.
 	* Using CNAMES is contested, it requires an additional DNS lookup, so there is some overhead. I use them however, because it is an effective way of managing growth in your environment. At a later time when I get some more servers, I can replace the CNAME records with A records and pre-existing applications will not break.

Edit `/var/lib/bind/db.192` to provide reverse IP address to domain name translation.

{% highlight bash %}

$ORIGIN .
$TTL 604800     ; 1 week
0.168.192.in-addr.arpa	IN	SOA pegasus.lan.example.com. root.lan.example.com. (
								2014040501 ; serial
								604800     ; refresh (1 week)
								86400      ; retry (1 day)
								2419200    ; expire (4 weeks)
								604800     ; minimum (1 week)
)
							NS 	pegasus.lan.example.com.
$ORIGIN 0.168.192.in-addr.arpa.
1 							PTR pegasus.lan.example.com.
10 							PTR galactica.lan.example.com.
12 							PTR xbmc.lan.example.com.

{% endhighlight %}

 * Note: only A records need an associated PTR record in this file.

Edit `/etc/default/bind9` and set:

	RESOLVCONF=yes

### Configure DHCP

Edit `/etc/default/isc-dhcp-server`

{% highlight bash %}

INTERFACES="eth1"

{% endhighlight %}

Edit `/etc/dhcp/dhcpd.conf`

{% highlight bash %}

ddns-updates on;
ddns-update-style interim;
authoritative;
update-static-leases on;

# Use the contents of /etc/bind/rndc.key
key rndc-key { algorithm hmac-md5; secret "XXXXXXXXXXXXXXXXXXXXX";}

allow unknown-clients;
use-host-decl-names on;

# Zones
zone 	lan.example.com. {
		primary localhost;
		key rndc-key;
}
zone 	0.168.192.in-addr.arpa. {
        primary localhost;
        key rndc-key;
}

# option definitions common to all supported networks...
default-lease-time 1814400; #21 days
max-lease-time 1814400; #21 days

subnet 192.168.0.0 netmask 255.255.255.0 {
	range 192.168.0.100 192.168.0.200;
	option subnet-mask 255.255.255.0;
	option broadcast-address 192.168.0.255;
	option routers 192.168.0.1;
	option domain-name-servers 192.168.0.1;
	option domain-name "lan.example.com";
	option netbios-name-servers 192.168.0.1;
	ddns-domainname "lan.example.com";
	ddns-rev-domainname "in-addr.arpa.";
}

log-facility local7;

{% endhighlight %}

### Configure UFW

Set defaults:

	$ sudo ufw default deny incoming
	$ sudo ufw default allow outgoing

And allow traffic from the local network:

	$ sudo ufw allow from 192.168.0.255


#### Allow Forwarding

Edit `/etc/ufw/sysctl.conf` and set the following:

	net/ipv4/ip_forward=1
	net/ipv6/conf/default/forwarding=1
	net/ipv6/conf/all/forwarding=1

Edit `/etc/default/ufw` and set the following:

	DEFAULT_FORWARD_POLICY="ACCEPT"

Edit `/etc/ufw/before.rules` and add this block to the top of the file, before the `*filter` statement.

{% highlight bash %}

# Configure NAT settings
*nat
:POSTROUTING ACCEPT [0:0]
#forward from eth1 through eth0
-A POSTROUTING -s 192.168.0.0/24 -o eth0 -j MASQUERADE

#forward internet HTTP traffic to internal address
-A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to-destination 192.168.0.10
-A PREROUTING -i eth0 -p udp --dport 80 -j DNAT --to-destination 192.168.0.10
COMMIT


{% endhighlight %}

### Configure Fail2Ban

	$ sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
	$ sudo cp /etc/fail2ban/fail2ban.conf /etc/fail2ban/fail2ban.local

Edit `/etc/fail2ban/jail.local` and enable or disable the jails based on the services you are running on the server. Some log file locations need to be updated.

#### Severly Ban Persistent Attackers

Taken verbatim from: <http://whyscream.net/wiki/index.php/Fail2ban_monitoring_Fail2ban>

When Fail2Ban notices an IP being blocked multiple times in it's own log file, ban it for an extra long time.

Add a new file `/etc/fail2ban/filter.d/fail2ban.conf`

{% highlight bash %}

# Fail2Ban configuration file
#
# Author: Tom Hendrikx
#
# $Revision$
#

[Definition]

# Option:  failregex
# Notes.:  regex to match the password failures messages in the logfile. The
#          host must be matched by a group named "host". The tag "<HOST>" can
#          be used for standard IP/hostname matching and is only an alias for
#          (?:::f{4,6}:)?(?P<host>\S+)
# Values:  TEXT
#

# Count all bans in the logfile
failregex = fail2ban.actions: WARNING \[(.*)\] Ban <HOST>

# Option:  ignoreregex
# Notes.:  regex to ignore. If this regex matches, the line is ignored.
# Values:  TEXT
#

# Ignore our own bans, to keep our counts exact.
# In your config, name your jail 'fail2ban', or change this line!
ignoreregex = fail2ban.actions: WARNING \[fail2ban\] Ban <HOST>

{% endhighlight %}

Edit `/etc/fail2ban/jails.local` and append the following:

{% highlight bash %}

[fail2ban]
enabled = true
filter = fail2ban
action = iptables-allports[name=fail2ban]
        sendmail-whois[name=fail2ban]
logpath = /var/log/fail2ban.log
# findtime: 1 week
findtime = 604800
# bantime: 1 week
bantime = 604800

{% endhighlight %}

### Restart the services

Unplug your old router, plug in your new super router and restart your services.

	$ sudo service networking restart
	$ sudo service bind9 restart
	$ sudo service isc-dhcp-server restart
	$ sudo service fail2ban restart
	$ sudo ufw disable && sudo ufw enable

## Sources

 * <http://blog.bigdinosaur.org/running-bind9-and-isc-dhcp/>
 * <https://help.ubuntu.com/community/Router>
 * <http://lani78.com/2008/08/12/dhcp-server-update-dns-records/>
 * <http://askubuntu.com/questions/162265/how-to-setup-dhcp-server-and-dynamic-dns-with-bind>
 * <https://www.digitalocean.com/community/articles/how-to-setup-a-firewall-with-ufw-on-an-ubuntu-and-debian-cloud-server>
 * <http://ubuntuforums.org/showthread.php?t=833844>
 * <http://stephen.rees-carter.net/thought/how-to-enable-ip-forwarding-with-ufw>
 * <http://linusramos.blogspot.ca/2012/01/ubuntu-enable-ip-forwarding-using-ufw.html>
 * <https://help.ubuntu.com/community/Fail2ban>
 * <http://whyscream.net/wiki/index.php/Fail2ban_monitoring_Fail2ban>
