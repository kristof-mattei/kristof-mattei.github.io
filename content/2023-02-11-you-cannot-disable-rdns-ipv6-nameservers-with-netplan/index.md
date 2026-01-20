+++
title = "You cannot disable rDNS (IPv6) nameservers with netplan"
date = 2023-02-11 16:23:01-08:00

[taxonomies]
categories = ["network", "IPv6"]

[extra]
author = "Kristof"
+++

I am playing with IPv6 and wanted to statically configure `/etc/resolv.conf`'s `nameserver`s. I wanted to avoid getting any from the Router, even though I wanted an IPv6 address through SLAAC.

### First try

Since the server runs Ubuntu, netplan is the default network configuration tool:

```
network:
  ethernets:
    eno1:
      addresses:
        # static IPv4 address
        - 10.0.10.10/24
      routes:
        # static IPv4 gateway
        - to: default
          via: 10.0.10.1
          metric: 100
          on-link: true
        # IPv6 gateway comes via RA
      nameservers:
        addresses:
          - 10.0.10.20
        # always use FQDN as short names clash with SSL
        search: [""]
      # disabled by default
      # dhcp4: false
      # dhcp4-overrides:
        # use-dns: false
        # use-domains: false
      # IPv6
      # We want IPv6 to be configured by RA only
      accept-ra: true
      # dhcp6: false
      # dhcp6-overrides:
        # use-dns: false
        # use-domains: false
      ipv6-privacy: false
  renderer: networkd
  version: 2
```

One `sudo netplan apply` later and we have

- a static IPv4 address
- a static DNS server
- a SLAAC IPv6 address
- ping to an IPv4 address work
- ping6 to an IPv6 address work

However, we have one issue. We're getting a IPv6 DNS address pushed...:

```
> cat /etc/resolv.conf
# ...
nameserver 10.0.10.20
nameserver 2600:xxxx:xxxx:xxxx::1
search .
```

Parsing netplan's reference yielded some reference to stateless configuration [here](<https://netplan.io/reference#:~:text=Off%20by%20default.-,dhcp6,-(bool)>). Time to try it out.

### Second try

Let's switch it on.

```
network:
  ethernets:
    eno1:
      # ...
      # IPv6
      # We want IPv6 to be configured by RA only
      accept-ra: true
      dhcp6: true
      dhcp6-overrides:
        use-dns: false
        use-domains: false
      ipv6-privacy: false
  # ...
```

Running `> sudo netplan apply && sudo systemctl restart systemd-resolved.service` and ...

```
> cat /etc/resolv.conf
# ...
nameserver 10.0.10.20
nameserver 2600:xxxx:xxxx:xxxx::1
search .
```

Still there... What's going on?

Digging deeper reveals that netplan is an abstraction tool over systemd.network. Reading through its configuration reference something caught my attention:
[`IPv6AcceptRA=`](https://manpages.ubuntu.com/manpages/bionic/man5/systemd.network.5.html#[network]%20section%20options:~:text=Defaults%20to%20%22no%22.-,IPv6AcceptRA%3D,-Enable%20or%20disable)

Notice the reference to an [`[IPv6AcceptRA]` section](https://manpages.ubuntu.com/manpages/bionic/man5/systemd.network.5.html#[ipv6acceptra]%20section%20options)

And this rang a bell... The DNS address that I got pushed comes from my router's rDNS.

So how do we configure this with netplan? We cannot. If we look in netplan's code for `IPv6AcceptRA` which only yields the following result:

{{ image(caption="Netplan doesn't support rendering the IPv6AcceptRA section.", name="ipv6acceptra-github.png") }}

### Last try

Eventually I ended up setting it manually in the networkd configuration:

```
> cat /run/systemd/network/10-netplan-eno1.network
[Match]
Name=eno1

[Network]
DHCP=ipv6
LinkLocalAddressing=ipv6
Address=10.0.10.10/24
IPv6AcceptRA=yes
DNS=10.0.10.20

[Route]
Destination=0.0.0.0/0
Gateway=10.0.10.1
GatewayOnLink=true
Metric=100

[DHCP]
RouteMetric=100
UseMTU=true
UseDNS=false
UseDomains=false

# vvvvvvvv
[IPv6AcceptRA]
UseDNS=false
UseDomains=false
# ^^^^^^^^
```

No more `sudo netplan apply`. This time we do `sudo systemctl restart systemd-networkd.service && sudo systemctl restart systemd-resolved.service`.

Checking `/etc/resolv.conf` one more time:

```
> cat /etc/resolv.conf
# ...
nameserver 10.0.10.20
search .
```

Bingo!

After some more searching I did stumble upon [an existing bug report](https://bugs.launchpad.net/netplan/+bug/1982010) on launchpad. At least I'm not the only one.

### Wrapping up

I removed the config for netplan. That way it cannot overwrite the network configuration that I edited.

-Kristof
