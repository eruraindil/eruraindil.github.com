---
layout: post
title:  "The State of my Network"
date:   2015-04-21 21:50:30
menu: blog
---

Quite a bit has changed in the past year - I finished school, moved, bought a condo, and am working full time. Working full time in an office is such a different lifestyle than being on campus and I am slowly adjusting.

One of my major projects has always been my home network. [My last post](/2014/04/06/building-a-router-out-of-an-old-computer.html) documented my adventure of learning how to build a router on an old computer. Even though I have since scrapped that implementation (because there is no need to reinvent the wheel), learning about bind9, fail2ban, and ISC DHCP has been useful in both my personal and professional projects.

As fun as my homemade router was, I was constantly worried about its security and stability. I also wanted more features like QoS which I know could be done with iptables. After reading man pages and tutorials for a couple hours I went looking and I found [pfSense](https://www.pfsense.org/): an open source enterprise level router/firewall system. The features match our pricy hardware firewall at work, but this one is free? Perfect, I'll take it.

In the 8 or so months I have been using it, I have been really happy with it. There have been a few issues on updates with the traffic shaper (QoS) (which could be my fault), but other than that, it has worked great and done everything I want it to. It has many features, and even optional packages like bind9 to extend the base functionality of the system (which uses dnsmasq for DNS OOTB).

My home network is growing and getting better. I ditched my consumer grade wifi router - and with the recent addition of some new [Raspberry Pi 2s](http://raspberrypi.org/) I am going to need a bigger switch soon.

![My home network diagram](/assets/home-network-public.png)

All trademarks and logos belong to their respective owners
 * TP Link Technologies Co, Ltd
 * Red Hat Inc
 * Canonical Ltd
 * Electric Sheep Fencing LLC
 * OpenELEC
 * Raspberry Pi Foundation
 * OnePlus
 * Google