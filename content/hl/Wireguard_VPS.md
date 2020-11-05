---
title: "Using Wireguard and a VPS to Bypass ISP Port Blocking and Hide Your Public IP"
date: 2020-11-05T09:46:29-08:00
draft: false
---
# Intro
In the past I've had to deal with ISPs blocking ports and in some cases most usable incoming ports. I want to show you how to bypass this using Wireguard and a VPS. That way you can start selfhosting services even if your ISP doesn't want you to.

For this tutorial I'm going to be using a DigitalOcean VPS (their smallest one) but you can use any provider you want. I'm going to provide referral links for some hosting services below. I don't make money off of them but I can get free server time.

* [DigitalOcean](https://m.do.co/c/d4aa430d72d9)

# Wireguard setup

Once you've got your VPS setup you'll want to ssh into and get started setting up wireguard. Keep a terminal open on your internal server that you want to forward to as we'll be running a lot of the same commands on both.

## Updates

The first thing we're going to do on both servers is update all of our software and reboot to make sure we've got the latest kernel.

```bash
#both
sudo apt update -y && sudo apt upgrade -y && sudo shutdown -r now
```

## Wireguard install
We're going to go ahead and add the wireguard repository and install it now. First you're going to want to add required packages in order to add their repository and then add their repository

```bash
#both
sudo apt install software-properties-common
sudo add-apt-repository ppa:wireguard/wireguard
```

Then we're going to actually install wireguard.

```bash
#both
sudo apt update # to make sure we've indexed the packages on their repo
sudo apt install wireguard -y
```

## Wireguard Configuration
First thing we're going to do is use the `wg` utility to generate some keys so the servers can authenticate with each other.

```bash
#both
(umask 077 && printf "[Interface]\nPrivateKey= " | sudo tee /etc/wireguard/wg0.conf > /dev/null)
wg genkey | sudo tee -a /etc/wireguard/wg0.conf | wg pubkey | sudo tee /etc/wireguard/publickey
```

Save both of the public keys that show for later. We'll use them later. Go ahead and open `/etc/wireguard/wg0.conf` with your prefered editor and we'll finish configuring these.

The following is an example of the wg0.conf on the VPS.
```ini
[Interface]
PrivateKey = <private key should be here>
ListenPort = 55107
Address = 192.168.4.1
[Peer]
PublicKey = <paste the public key from your home server here>
AllowedIPs = 192.168.4.2/32
```
The following is an example of the wg0.conf on your home server.
```ini
[Interface]
PrivateKey = <private key should be here>
Address = 192.168.4.2
[Peer]
PublicKey = <paste the public key from your VPS here>
AllowedIPPs = 192.168.4.1/32
Endpoint = <paste the public ipv4 address of your VPS here>:55107
PersistentKeepalive = 25
```

## Sysctl Setup

Now we'll need to make some changes to our sysctl.conf in order to allow our VPS to forward using IPtables. Open `/etc/sysctl.conf` in your favorite editor.

Find the following line and remove the `#` that is commenting it out.
```bash
#VPS
#net.ipv4.ip_forward=1
```

Then we'll apply that change with the following commands.
```bash
#VPS
sudo sysctl -p
sudo sysctl --system
```

## Testing
Now that everything is configured we're going to go ahead and startup our tunnel between the servers using the following command.

```bash
#both
sudo systenctl start wg-quick@wg0
sudo systemctl enable wg-quick@wg0
```

Then we're going to ping the servers from eachother. On your VPS try to ping your home server. On your homeserver try to ping your VPS. (using the IPs in your wireguard configuration)
```bash
#VPS
ping 192.168.4.2
```
```bash
#Home server
ping 192.168.4.1
```

If successful then the tunnel is working.

# IPTables Setup
On our VPS we're going to setup some IPtables rules to forward to a reverse proxy running on our home server.

*replace eth0 with the public interface of your VPS (found using `ip a`)*
```bash
# VPS

# By default drop traffic
sudo iptables -P FORWARD DROP

# Allow traffic on specified ports
sudo iptables -A FORWARD -i eth0 -o wg0 -p tcp --syn --dport 80 -m conntrack --ctstate NEW -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o wg0 -p tcp --syn --dport 443 -m conntrack --ctstate NEW -j ACCEPT

# Allow traffic between wg0 and eth0
sudo iptables -A FORWARD -i wg0 -o eth0 -m conntrack --cstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A FORWARD -i wg0 -o eth0 -m conntrack --cstate ESTABLISHED,RELATED -j ACCEPT

# Forward traffic from eth0 to wg0 on specified ports
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to-destination 192.168.4.2
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 443 -j DNAT --to-destination 192.168.4.2

# Forward traffic back to eth0 from wg0 on specified ports
sudo iptables -t nat -A POSTROUTING -o wg0 -p tcp --dport 80 -d 192.168.4.2 -j SNAT --to-source 192.168.4.1
sudo iptables -t nat -A POSTROUTING -o wg0 -p tcp --dport 443 -d 192.168.4.2 -j SNAT --to-source 192.168.4.1
```

## Peristing IPTables
In order to have these rules perist through reboots we'll need to install netfilter-peristent, use it to save the current configuration, and then enable it.

```bash
# VPS
sudo apt install netfilter-persistent
sudo netfilter-persistent save
sudo systemctl enable netfilter-persistent
```

Then we'll need to use iptables persistent and configure that.

```bash
# VPS
sudo apt install iptables-persistent
# hit yes to save the current rules.
```

# Summary
Now everything should be setup and port 80 and 443 should be forwarded to your homeserver.