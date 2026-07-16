---
title: "Setting up Wireguard"
weight: 2
draft: false
description: "Setting up Wireguard in homeserver"
slug: "setting-up-wireguard"
tags: ["Homelab", "SSH", "Wireguard", "EndeavourOS"]
series: ["Homelab Setup"]
series_order: 2
showDate: true
showTableOfContents: true
date: 2026-07-12
---

## Introduction

By the time of writing, I have switched my server to **EndeavourOS**, an Arch-based Linux distribution. Therefore, some commands differ from Debian/Ubuntu-based systems.

It is recommended to perform the initial WireGuard and firewall setup **directly on the server**. Configuring these remotely over SSH can lock you out if firewall rules are applied before the tunnel is working.

The goal of this setup is:

- Create a WireGuard VPN tunnel between a client and server.
- Restrict SSH access through the VPN tunnel.
- Allow optional LAN access.
- Use DuckDNS for dynamic public IP resolution.
- Configure UFW firewall rules.

## Setup WireGuard

### Install WireGuard

On EndeavourOS:

```bash
sudo pacman -S wireguard-tools
```

### Generate WireGuard Keys

Generate a private/public key pair on both the server and client:

For server

```bash
# generate the private key
$ wg genkey >./wg0.priv

# derive the public key
$ wg pubkey <./wg0.priv >./wg0.pub
```

For client

```bash
# generate the private key
$ wg genkey >./wg1.priv

# derive the public key
$ wg pubkey <./wg1.priv >./wg1.pub
```

Additionally, pre-shared keys can also be generated. Refer [^wireguard-setup]

## Server Configuration

Create the WireGuard configuration:

```bash
sudo nano /etc/wireguard/wg0.conf
```

Example:

```ini
[Interface]
Address = 10.0.0.10/24
SaveConfig = true

PostUp = ufw route allow in on wg0 out on wlp2s0
PostUp = iptables -t nat -I POSTROUTING -o wlp2s0 -j MASQUERADE

PreDown = ufw route delete allow in on wg0 out on wlp2s0
PreDown = iptables -t nat -D POSTROUTING -o wlp2s0 -j MASQUERADE

ListenPort = 51830
PrivateKey = !!!put the contents of ./wg0.priv here!!!


[Peer]
PublicKey = !!!put the contents of ./wg1.pub here!!!
AllowedIPs = 10.0.0.1/32
```

`wlp2s0` is the wifi network interface which can be replaced by your preferred network interface like `eth0`.

Find your network interface:

```bash
ip addr
```

Example interfaces:

```bash
wlp2s0   ## WiFi
eth0   ## Ethernet
```

`PostUp` and `PreDown` masquerading rules are taken from [^masquerade]. You can add your own rules to configure different functionality like killswitch [^killswitch].

## Client Configuration

Create the client WireGuard configuration:

```bash
wg0.conf
```

Example:

```ini
[Interface]
PrivateKey = !!!put the contents of ./wg1.priv here!!!
ListenPort = 51831
Address = 10.0.0.1/24
DNS = 1.1.1.1


[Peer]
PublicKey = !!!put the contents of ./wg0.pub here!!!
AllowedIPs = 10.0.0.10/32
Endpoint = server.duckdns.org:51830 #or the server ip address
```

### Understanding AllowedIPs

For SSH-only access:

```ini
AllowedIPs = 10.0.0.10/32
```

means only traffic destined for the server WireGuard IP goes through the tunnel.

For routing all traffic through the VPN:

```ini
AllowedIPs = 0.0.0.0/0
```

would route all IPv4 traffic through the server.

## Setup DuckDNS Endpoint (Optional)

If the server is accessed outside the local network, the public IP address may change periodically.

DuckDNS provides a free dynamic DNS service that maps a domain name to your current public IP. Setup instructions can be found on their website.[^duckdns]

Example:

```bash
server.duckdns.org
```

DuckDNS supports up to five free domains. Note that the required UDP and SSH ports need to be forwarded via router.

## Enable WireGuard at Startup

Enable WireGuard to start automatically:

```bash
sudo systemctl enable wg-quick@wg0.service
```

## Setting Up UFW Firewall

### Install UFW

```bash
sudo pacman -S ufw
```

Enable and start:

```bash
sudo systemctl enable --now ufw
```

### Allow WireGuard Port

WireGuard listens on UDP port `51830`:

```bash
sudo ufw allow 51830/udp
```

This port must be accessible from the internet if connecting remotely.

### Allow Local Network Access

To allow devices on the local network to access the server directly:

```bash
sudo ufw allow from 192.168.1.0/24 to 192.168.1.xxx
```

Replace:

```bash
192.168.1.xxx
```

with the server's LAN IP.

### Allow SSH Through WireGuard

Allow SSH access only from the WireGuard subnet:

```bash
sudo ufw allow from 10.0.0.0/24 to 10.0.0.10 port 22 proto tcp
```

This allows:

```bash
Client
  |
  | WireGuard tunnel
  |
10.0.0.10 Server
  |
SSH port 22
```

### Run the firewall

Run the firewall via  

```bash
sudo ufw enable 
```

or to reload new rules,

```bash
sudo ufw reload 
```

## Start WireGuard

Start manually:

```bash
sudo wg-quick up wg0
```

or:

```bash
sudo systemctl start wg-quick@wg0.service
```

Check the interface:

```bash
ip addr
```

You should see:

```bash
wg0
```

Check WireGuard status:

```bash
sudo wg show
```

## Connect Through SSH

From the client:

```bash
ssh user@10.0.0.10
```

or specify the port:

```bash
ssh -p 22 user@10.0.0.10
```

## Removing Old SSH Host Signature

After reinstalling the operating system, SSH may show:

```bash
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

This happens because reinstalling the OS generates a new SSH host key.

### Verify the New Server Key

Run this directly on the server (not through SSH):

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Confirm that the displayed fingerprint matches the server's expected key.

### Remove the Old Client Entry

On the client:

```bash
ssh-keygen -R [10.0.0.10]:22 # Port number matters if you're using non-standard port
```

Then reconnect:

```bash
ssh user@10.0.0.10
```

SSH will ask to accept the new host key.

## References

Disclaimer: Minor markdown edits and filling some gaps utilized ChatGPT [^chatgpt] which was then crosschecked by me.

[^chatgpt]: [ChatGPT](chatgpt.com)

[^wireguard-setup]: "Simplest Wireguard setup". Available at: [https://weizenspr.eu/protect-ssh-with-wireguard/](https://weizenspr.eu/protect-ssh-with-wireguard/)

[^duckdns]: "DuckDNS install" [for linux](https://www.duckdns.org/install.jsp)

[^masquerade]: PreUp and Post down rules by [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-22-04#step-5-configuring-the-wireguard-server-s-firewall)

[^killswitch]:For [kill-switch](https://rexbytes.com/2026/02/10/setting-up-a-kill-switch-wireguard-vpn/)

### Interesting Resources

From Ubuntu

[Ubuntu Server – WireGuard VPN Common Tasks](https://ubuntu.com/server/docs/how-to/wireguard-vpn/common-tasks/)

The resources below explain the fundamentals of tunneling and advanced configurations

- [SSH over WireGuard](https://frank.sauerburger.io/2023/09/13/ssh-over-wireguard.html)
- [WireGuard Keys for SSH](https://www.procustodibus.com/blog/2024/02/wireguard-keys-for-ssh/)
- [WireGuard UFW – Point-to-Point](https://www.procustodibus.com/blog/2021/05/wireguard-ufw/#point-to-point)
- [WireGuard Point-to-Point Config](https://www.procustodibus.com/blog/2020/11/wireguard-point-to-point-config/)
