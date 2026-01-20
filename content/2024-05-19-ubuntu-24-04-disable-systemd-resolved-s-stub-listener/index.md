+++
title = "Ubuntu 24.04: disable systemd-resolved's stub listener"
date = 2024-05-19 13:43:01-07:00

[taxonomies]
categories = ["network", "ubuntu", "IPv6"]

[extra]
author = "Kristof"
+++

I'm rebuilding my server and I need to make sure the server itself does not listen to port 53 as we'll be hosting a Docker container with AdGuard on port 53.

To do that we need to disable systemd-resolved's stub listener.

The trick is to set `DNSStubListener=no` in `/etc/systemd/resolved.conf`:

```
# ...

[Resolve]
# ...
#DNSStubListener=yes
# ...
```

However, this will cause APT to ask you to resolve configuration discrepancies should systemd-resolved be updated (e.g. when updating Ubuntu to vNext).

So we take a step back, read the comments, and ...:

```
# Entries in this file show the compile time defaults. Local configuration
# should be created by either modifying this file (or a copy of it placed in
# /etc/ if the original file is shipped in /usr/), or by creating "drop-ins" in
# the /etc/systemd/resolved.conf.d/ directory.
```

Cool, we create the directory:

```
# Create directory
sudo mkdir -p /etc/systemd/resolved.conf.d/
# Set permissions
sudo chmod 755 /etc/systemd/resolved.conf.d/
# Disable
printf "[Resolve]\nDNSStubListener=no\n" | sudo tee /etc/systemd/resolved.conf.d/noresolved.conf
# Restart systemd-resolved
sudo systemctl restart systemd-resolved.service
# Test
cat /etc/resolv.conf
```

And notice that it no longer contains something like this:

```
nameserver 127.0.0.53
options edns0 trust-ad
search .
```

But something like this:

```
nameserver 192.168.1.1
search home.domain.com
```

Until next time!

-K
