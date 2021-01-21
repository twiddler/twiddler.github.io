---
layout: post
title: Site-to-site, split-tunneling VPN with WireGuard
categories: [wireguard, vpn, split, tunneling, site-to-site]
---

This is a quick intro to setting up your first site-to-site virtual private network (VPN) with split-tunneling enabled. After completing this guide

- your client and server will be on the same virtual private network `10.0.0.0/24`
- your client will only send request with a target of that virtual network to your server (split tunneling)

# What is a site-to-site VPN?

VPN is short for virtual private network. They emulate a private network over the internet. VPNs are popular for obfuscating client identities and origins (e.g. for circumventing geoblocking) or accessing internal networks remotely (e.g. when working from home). The latter are called site-to-site VPNs.

# Where do I get one?

With [WireGuard](https://www.wireguard.com/), you can set up your VPN on your server directly. There is no need to purchase any other service.

Note that for this guide we assume your server has a static IP.

So let's get our hands dirty!

# Setting up the server

Follow these steps to install and configure WireGuard on your server. Execute (almost) all commands with superuser privileges (`#`).

## Install WireGuard

[Refer to the WireGuard docs.](https://www.wireguard.com/install/)

## Create an identity

The configuration and private key file we will create must belong to and only be accessible by `root`. We can restrict permissions of new files with

```console
# umask 077
```

WireGuard authenticates connections with public and private keys. To generate a new key pair, run

```console
# cd /etc/wireguard
# wg genkey | tee private | wg pubkey > public
```

## Configure WireGuard

We can configure WireGuard with `.conf` files or by passing parameters on the command line. We will use configuration files for reproducibility.

Create a new file `/etc/wireguard/wg0.conf`:

```
[Interface]
PrivateKey = <your private key>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer] # client 1
PublicKey = <public key of client 1, see sections below>
AllowedIPs = 10.0.0.2/32 # fixed IP assigned to client 1
```

Note that WireGuard listens on UDP port 51820 for incoming connections. You will have to open that port in your firewall to accept incoming connection requests.

Some guides will tell you to turn on IP forwarding (`net.ipv4.ip_forward = 1` in `/etc/sysctl.conf`). Since we only want to tunnel requests to 10.0.0.0/24 subnet, we do not need IP forwarding (`net.ipv4.ip_forward = 0`).

We will launch WireGuard after we configured the client, too.

# Setting up the client

Install WireGuard and generate a private and public key like you did on the server.

Create a new file `/etc/wireguard/wg0.conf` and plug in your server's public key and public IP:

```console
[Interface]
Address = 10.0.0.2/32 # same fixed IP as above
PrivateKey = <client's private key>

[Peer]
PublicKey = <server's public key>
Endpoint = <server's public IP>:51820
AllowedIPs = 10.0.0.0/24 # which IPs to look up in the VPN
```

Go back to the server config and insert your client's public key.

# Starting the VPN

On both client and server, run

```console
# wg-quick up wg0
# watch --color WG_COLOR_MODE=always wg
```

to start and watch WireGuard connections.

To confirm that client and server are connected, run

```
$ ping 10.0.0.1
```

in another terminal on your client, and

```
$ ping 10.0.0.2
```

in another terminal on your server. Take a look at the `watch` output and notice that both client and server receive data over WireGuard.

# Starting WireGuard on system startup

Enable and run WireGuard services on those machines that should run in on system startup:

```
# systemctl enable --now wg-quick@wg0.service
```

# Adding more clients

To add more clients, repeat the client steps on all clients you want to connect. Then add their public keys and assign fixed IPs in your server's `wg0.conf`.

# Tunneling all traffic through the VPN

If you'd like to tunnel all traffic through your server, set `AllowedIPs = 0.0.0.0/0` in your client's `wg0.conf` and turn on IP forwarding on the server.

# Credits

Thank you very much to everyone who contributed to the man pages and online documentations of the tools used in this article! You rock!

Also, thank you very much to the authors of the following articles! Take a look at those if you found some of my instructions incomplete or hard to follow, maybe they fit your way of learning things more.

- M. Mota, [_Getting Started with WireGuard_](https://dev.to/miguelmota/getting-started-with-wireguard-n9e#installing-wireguard-on-client)
- A. Kuusela, [_How to get started withWireGuardVPN_](https://upcloud.com/community/tutorials/get-started-wireguard-vpn/)

Have fun! :tada:
