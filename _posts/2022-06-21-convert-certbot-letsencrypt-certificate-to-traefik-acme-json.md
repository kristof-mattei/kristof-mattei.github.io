---
author: Kristof
comments: true
date: 2022-06-21 10:03:01-08:00
layout: post
slug: convert-certbot-letsencrypt-certificate-to-traefik-acme-json.
title: Convert Certbot Let's Encrypt certificate to Traefik's acme.json
categories:
- self-hosted
---

I run a couple of services at home for which I historically used [LinuxServer's SWAG](). It served both as a proxy and as an application that automatically fetches and updates a Let's Encrypt certifiate for that sweet HTTPS padlock. Last weekend I stumbled upon Traefik. A more modern proxy that looked easier to set up and integrate. I decided to give it a shot.

Updating the Docker configuration was easy. It is much more concise versus SWAG, and it can all be stored in the `docker-compose.yml`, as opposed to SWAG's `proxy-confs`'s dir. Let's Encrypt and Cloudflare integration was also super easy. The keys go into the CLI arguments of Traefik.

Of course some iterations were needed to get things just right, and 5 iterations further I noticed that things started to fail... What? 

Well, Let's Encrypt only issues 5 certificates / domain / 168 hours (7 days)... and Traefik stores the data in `acme.json` which by default is not stored in a `VOLUME`, so you need to be explicit that you want to have this file survive container recreation...

But now I'm stuck, I want to keep on hacking at this but SSL just isn't working.

My first solution was to switch to Let's Encrypt's staging servers. I iterated a couple more times and now I had a setup where the `acme.json` file actually survived container recreations. 

But the staging servers aren't trusted. I went looking into the old SWAG folder and found a bunch of certificates and I started to wonder if I could port those over to `acme.json` and make Traefik believe that I had valid certificates. That way I had HTTPS at least for 2 weeks, and if I THEN have issues I'll solve them when they pop up...

The trick was to come up with a way to take certbot's configuration for a certain domain (private key, private key for domain, public key for domain) and transform it in a way that Traefik accepts in `acme.json`.

And I'd like to share the way that I 'discovered'.

We'll need a couple of things.

* `base64`
* `openssl`
* `python3`
* A SWAG setup with a valid Let's Encrypt certificate

If we look at `acme.json` we see 4 blanks that we have to fill in.

```
{
  "letsencrypt": {
    "Account": {
      "Email": "<email>",
      "Registration": {
        "body": {
          "status": "valid",
          "contact": [
            "mailto:<email>"
          ]
        },
        "uri": "<uri>"
      },
      "PrivateKey": "<private key 1>",
      "KeyType": "4096"
    },
    "Certificates": [
      {
        "domain": {
          "main": "<domain>",
          "sans": [
            "*.<domain>"
          ]
        },
        "certificate": "<certificate>",
        "key": "<private key 2>",
        "Store": "default"
      }
    ]
  }
}
```

5 iterations later I learned that I missed setting up Traefik's `acme.json` to my volumes, so Let's Encrypt
