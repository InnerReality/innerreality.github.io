---
title: "Setting up Wireguard (with DuckDNS)"
weight: 2
draft: false
description: "Setting up Wireguard with optional DuckDNS in homeserver"
slug: "setting-up-wireguard"
tags: ["Homelab", "SSH", "Wireguard", "EndeavourOS", "DuckDNS"]
series: ["Homelab Setup"]
series_order: 2
showDate: true
showTableOfContents: true
date: 2026-07-12
---

## Introduction

By the time of writing, I have switched my server to **EndeavourOS**, an Arch-based Linux distribution. Therefore, some commands differ from Debian/Ubuntu-based systems.

{{< alert >}}
**Warning!** It is recommended to perform the initial WireGuard and firewall setup **directly on the server**. Configuring these remotely over SSH can lock you out if firewall rules are applied over SSH before the tunnel is working.
{{< /alert >}}

The goal of this setup is:

- Create a WireGuard VPN tunnel between a client and server.
- Restrict SSH access through the VPN tunnel.
- (Optional) Use DuckDNS for dynamic public IP resolution.
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

### Server Configuration

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

ListenPort = 51830 # make note of this listening port
PrivateKey = !!!put the contents of ./wg0.priv here!!!


[Peer]
PublicKey = !!!put the contents of ./wg1.pub here!!!
AllowedIPs = 10.0.0.1/32

# you can add multiple clients as follows:
[Peer]
PublicKey = !!!put the contents of ./wg2.pub here!!!
AllowedIPs = 10.0.0.2/32
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

### Client Configuration

Create the WireGuard configuration in client device.

Example `wg1.conf`:

```ini
[Interface]
PrivateKey = !!!put the contents of ./wg1.priv here!!!
ListenPort = 51831
Address = 10.0.0.1/24
DNS = 1.1.1.1


[Peer]
PublicKey = !!!put the contents of ./wg0.pub here!!!
AllowedIPs = 10.0.0.10/32
Endpoint = server.duckdns.org:51830 # or the server ip address, with listen port

# no need to add other clients here, since this is not point to point configuration
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

DuckDNS provides a free dynamic DNS service that maps a domain name to your current public IP. Setup instructions can be found on their website.[^duckdns]. After signing up and obtaining **domain name** and **token**, setup the `cron` job for the domain `server.duckdns.org`

### Installing `cron`

Since EndeavourOS doesn't come with `cron`, install it by

```bash
sudo pacman -S cronie
```

Enable (on startup) and start by

```bash
sudo systemctl enable --now cronie
```

#### Checking `cron` status

Verify that it is `Active(running)`

```bash
sudo systemctl status cronie
```

### Creating `duckdns.sh`

This is a script that automatically allows `duckdns.org` to update public IP address of the server (here, the router)

```bash
mkdir duckdns
cd duckdns
vim duckdns.sh
```

Inside `duckdns.sh`, paste the script, change the server **domain name** under `domains` (mutiple domains are supported) and update the **token** under `token`

```bash
echo url="https://www.duckdns.org/update?domains=server&token=<your-server-token-from-duckdns>&ip=" | \
curl -k  -o ~/duckdns/duckdns.log -K -

echo "[$(/bin/date '+%Y-%m-%d %H:%M:%S')] Updated DuckDNS" >> ~/duckdns/duckdns.log
```

The last line is a custom modification that appends the date and time of the update. Used to check if `cron` job updates it correctly.

### Testing `duckdns.sh`

Make it executable by

```bash
chmod 700 duckdns.sh
```

Test the script by

```bash
./duckdns.sh
```

The output should be

```bash
  % Total    % Received % Xferd  Average Speed  Time    Time    Time   Current
                                 Dload  Upload  Total   Spent   Left   Speed
100      2   0      2   0      0      2      0                              0
```

#### Checking `duckdns.log`

Check the `duckdns.log` file. It now contains the last updataed timestamp. `OK` indicates working, whereas `KO` indicates not working.

```bash
cat duckdns.log
```

```bash
OK[2026-07-12 16:18:48] Updated DuckDNS
```

### Setting up `cron`

Run

```bash
crontab -e
```

This will open up the file in vim editor (probably an EndeavourOS thing). Paste the following at the bottom and save it.

```ini
*/5 * * * * ~/duckdns/duckdns.sh >/dev/null 2>&1
```

Upon saving, the output shows

```bash
crontab: installing new crontab
```

The `cron` job will run the `duckdns.sh` every 5 minutes. You can verify by [checking `duckdns.log`](#checking-duckdnslog) after 5 minutes. Alternatively, you could also check for timestamp near `CMDEND (~/duckdns/duckdns.sh >/dev/null 2>&1)` in the logs when [checking `cron` status](#checking-cron-status)

### Notes

DuckDNS supports up to five domains for free. Note that the required UDP and SSH ports need to be forwarded via router.

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

### UFW Rules

#### Allow WireGuard Port

WireGuard listens on UDP port `51830`:

```bash
sudo ufw allow 51830/udp
```

This port must be accessible from the internet if connecting remotely.

#### Allow Local Network Access

To allow devices on the local network to access the server directly:

```bash
sudo ufw allow from 192.168.1.0/24 to 192.168.1.xxx
```

Replace:

```bash
192.168.1.xxx
```

with the server's LAN IP.

#### Allow SSH Through WireGuard

Allow SSH access only from the WireGuard subnet:

```bash
sudo ufw allow from 10.0.0.0/24 to 10.0.0.10 port 22 proto tcp
```

This allows:

{{< mermaid >}}
flowchart LR
    A[Client] -->|WireGuard tunnel| B["Server\n10.0.0.10"]
    B --> C["SSH\nPort 22"]
{{< /mermaid >}}

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

### Check if wireguard is running

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

## Troubleshooting

### Removing Old SSH Host Signature

After reinstalling the operating system, SSH may show:

```bash
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

This happens because reinstalling the OS generates a new SSH host key.

#### Verify the New Server Key

Run this directly on the server (not through SSH):

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Confirm that the displayed fingerprint matches the server's expected key.

#### Remove the Old Client Entry

On the client:

```bash
ssh-keygen -R [10.0.0.10]:22 # Port number matters if you're using non-standard port
```

Then reconnect:

```bash
ssh user@10.0.0.10
```

SSH will ask to accept the new host key.

### SSH Connection Refused

If SSH connection refused as follows on your client device

```bash
ssh: connect to host 10.0.0.10 port 22: Connection refused
```

#### 1. Check if SSH is running on host

   ```bash
    sudo systemctl status sshd
  ```

   Look for `Active: active (running)` in the output. If it's running, you can connect. Otherwise, use the following command to start the service.

  ```bash
    sudo systemctl start sshd
  ```

  Additionally if it doesn't display `enabled` here:
  
  ```bash
   Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; preset: disabled)
  ```

  Then it is recommended to automatically restart on boot by

  ```bash
    sudo systemctl enable sshd
  ```

#### 2. Wireguard Status

Check if [wireguard is running](#check-if-wireguard-is-running)

#### 3. UFW configuration

1. Check UFW configuration (only shows up if it is enabled) and verify the [rules added](#ufw-rules)

    ```bash
    sudo ufw status numbered
    ```

2. Try connecting via client within the local network (with `192.168.1.xxx`)

3. Disable firewall and check if SSH connection works with or without wireguard.

    ```bash
    sudo ufw disable
    ```

## References

Disclaimer: Minor markdown edits and filling some gaps utilized ChatGPT [^chatgpt] which was then crosschecked by me. Subsequent edits were typed entirely by me.

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
