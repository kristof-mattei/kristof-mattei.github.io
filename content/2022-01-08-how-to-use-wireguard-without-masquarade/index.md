+++
title = "How to use Wireguard without Masquarade"
date = 2022-01-08 11:27:01-08:00

[taxonomies]
categories = ["VPN"]

[extra]
author = "Kristof"
+++

For a while now I've been running [linuxserver.io's Docker container of Wireguard](https://docs.linuxserver.io/images/docker-wireguard) to be able to get access to my home network when I'm on the go.

Initially, we had a network for the computers at home, `192.168.25.0/24`. (Of course IOT sits in another network).

The Wireguard VPN device sat on `192.168.25.20`. And since it uses NAT, and that doesn't allow me to track individual peers at the network level. All traffic coming over the Wireguard VPN would look like it originates from THAT IP.

Time to split that up.

So what are the steps that we need to do?

### Define the VPN network and set up the route

Note: I contemplated keeping my Wireguard clients in the same subnet, but that would mean that EVERY device in that subnet would need to get static routes to the Wireguard clients. Whereas if I move them to a separate subnet, only the router needs to get the static route.

I chose to use `192.168.30.0/24` for all my Wireguard clients.

Our Wireguard server sits at `192.168.25.20`, so on the router we add the static route:

{{ image(caption="Static route screenshot.", name="static-route-screenshot.png") }}

### Update Wireguard's config to not use Masquerade

The default `wg0.conf` needs to be updated to look like this:

```ini
[Interface]
Address = 192.168.30.1
ListenPort = 51820
PrivateKey = ##############################################

[Peer]
# peer_iPhoneKristof
PublicKey = ##############################################
AllowedIPs = 192.168.30.10/32
```

So what did we change?

1. We removed `PostUp` and `PostDown`, as these are only needed when we do NAT (i.e. Masquarade)
1. We set up the `Interface`'s address to be `.1` in our new range.
1. We set our client to be `.10` in our new range.

### Ensure the server forwards IPv4 packets AND enable proxy ARP

Lastly, the underlying server needs 2 changes:

In `/etc/sysctl.conf` (or wherever it is on your flavor of Linux):

```ini
# Enable IPv4 packet forwarding
net.ipv4.ip_forward=1

# Enable Proxy ARP (https://en.wikipedia.org/wiki/Proxy_ARP)
net.ipv4.conf.all.proxy_arp=1
```

### Update the client's config

We need to make sure that the client now connects with the right IP, and that the client's `AllowedIPs` are set up to target our ORIGINAL range:

```ini
[Interface]
PrivateKey = ##############################################
ListenPort = 51820
Address = 192.168.30.10
DNS = 192.168.25.5 # adguard sits here

[Peer]
PublicKey = ##############################################
AllowedIPs = 192.168.25.0/24
Endpoint = my.endpoint.com:51820
```

### Conclusion

We now can track individual Wireguard clients on our network.
