---
title: "Home server setup: Hardware and OS, WireGuard VPN with Pi-hole and Unbound"
date: 2024-08-03
tags:
  - programming
  - server
categories:
  - tech
keywords:
  - vpn
  - server
---

## Server Hardware and OS Configuration

I had an old HP laptop lying around (a HP Pavilion TS 14 Notebook PC) that was too slow to use as a daily driver, but it looked like it would be able to handle a new life as a simple home server. It had issues with lag even with the lightest Linux desktop environments installed, so I decided to install Ubuntu Server on it instead, which runs without a DE by default. I could use SSH to perform administrative tasks, which I found was much lighter and faster than using remote desktop software.

I installed Ubuntu Server 22.04.4 LTS on the laptop using the standard live USB installation method. I then needed to configure the laptop to stay on even when its lid was closed and the screen was turned off, so that I could run it perpetually without consuming too much power and risking screen burn. I opened up `/etc/systemd/logind.conf` and uncommented and altered the following lines like so:

```ini
HandleSuspendKey=ignore
HandleLidSwitch=ignore
HandleLidSwitchDocked=ignore
```

## Installing Pi-hole, WireGuard and Unbound

### Use case

I wanted to install Pi-hole as a network-wide ad-blocking DNS server, but I decided to put it behind a WireGuard VPN for a couple of reasons. First of all, I share Wi-Fi with my neighbours, so if Pi-hole broke it would affect their DNS resolution as well as mine. By putting Pi-hole behind WireGuard, if Pi-hole broke for any reason I could simply log out of WireGuard to switch back to my network's default DNS server, and anyone else not using the WireGuard VPN would be unaffected by the break. Secondly, I wanted to be able to access my Pi-hole from anywhere, even when not connected to my home network. A WireGuard VPN would make this possible. As a bonus, I would be able to hide any services I was running on my home network behind the VPN using firewall rules on my server. Anyone connected to the network wouldn't even know the services existed unless they had access to the VPN.

### Installing Pi-hole

I followed the installation instructions on the [Pi-hole website](https://docs.pi-hole.net/main/basic-install/). I was able to get it up and running fairly quickly using the One-Step Automated Install script. The installation required that I set a static IP for my server, and I was luckily able to set one through my router's web interface. I wouldn't need to disable DHCP this way; the router would just keep assigning the same reserved IP address to my server and assign other IPs to other devices as normal. Editing the router settings directly was also easier and more persistent than setting the static IP by editing the server settings.

### Installing WireGuard

I followed the WireGuard installation instructions on the Pi-hole website once again. According to the instructions, to make my VPN accessible outside of my home network I needed to set dynamic DNS for my router's IP address and allow port forwarding for the port WireGuard was running on. I used [FreeDNS](https://freedns.afraid.org/) to set dynamic DNS for my router. I was also able to allow port forwarding through my router's web interface.

Getting Pi-hole to work as a DNS server for my VPN was the trickiest part, but after doing some digging I found a simple solution. I had each VPN client configured to communicate with the WireGuard server, but according to [this post](https://www.reddit.com/r/WireGuard/comments/ejjppr/comment/fcz2uwp/) I also needed to allow my clients to communicate with my Pi-hole. I accomplished this by adding my Pi-hole's IP address (which is the same as my server's static IP) to `AllowedIPs` and setting `DNS` equal to this IP.

### Installing Unbound and connecting to Pi-hole

I installed Unbound as a recursive DNS server for my Pi-hole. It eliminates the need to rely on third-party DNS servers to access websites, which would leave my server more vulnerable to security issues such as DNS poisoning. I followed [this guide](https://docs.pi-hole.net/guides/dns/unbound/) and was able to set up Unbound without any problems.

## Setting up a firewall for WireGuard

P
The final step was to use Uncomplicated Firewall to set up some simple rules so that my server would only be accessible over WireGuard. I found [this article](https://www.procustodibus.com/blog/2021/05/wireguard-ufw/) pretty helpful when figuring out which commands to use.

I started with a couple of commands to deny all incoming traffic and allow all outgoing traffic by default. I then put in two more commands, one allowing connections over the port WireGuard was listening on, and another allowing connections through WireGuard's `wg0` interface. Finally, I reloaded the firewall and checked its status. These were the exact commands I used:

```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 47111/udp
sudo ufw allow in on wg0
sudo ufw reload
sudo ufw status
```

And this was the output:

```
Status: active

To                         Action      From
--                         ------      ----
47111/udp                  ALLOW       Anywhere
Anywhere on wg0            ALLOW       Anywhere
47111/udp (v6)             ALLOW       Anywhere (v6)
Anywhere (v6) on wg0       ALLOW       Anywhere (v6)
```

Now I could only SSH into my server or access its web services if I logged into my WireGuard VPN first, which is exactly what I wanted.
