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
* `conv.py` from [https://gist.github.com/ivanov17/67a8acb1bfd0e7b5e3929aa9fbc379a5](here)

To start we'll have Traefik generate the `acme.json` file for us with the staging server. This way we ensure that our `acme.json` survives container recreation.

PROTIP! Destroy your container and re-deploy there. Is `acme.json` still the same? Great! 

If we look at `acme.json` we see 4 blanks that we have to fill in.

```jsonc
{
  "letsencrypt": {
    "Account": {
      "Email": "<email>", // 1
      "Registration": {
        "body": {
          "status": "valid",
          "contact": [
            "mailto:<email>" // 2
          ]
        },
        "uri": "<account uri>" // 3
      },
      "PrivateKey": "<private key>", // 4
      "KeyType": "4096"
    },
    "Certificates": [
      {
        "domain": {
          "main": "<domain>", // 5
          "sans": [
            "*.<domain>" // 5
          ]
        },
        "certificate": "<certificate>", // 6
        "key": "<key>", // 7
        "Store": "default"
      }
    ]
  }
}
```

### `Email` (1), (2)
Your email address. I'm not sure if it needs to match the original one or not. I didn't do it, and it still works!

For (2) make sure to retain the `mailto:` prefix.

### `Registration.uri` (3)
This field is the `uri` found in `<swag>/etc/letsencrypt/accounts/acme-v02.api.letsencrypt.org/directory/******/meta.json`

### `PrivateKey` (4)
For this you need the `private_key.json` found in `<swag>/etc/letsencrypt/accounts/acme-v02.api.letsencrypt.org/directory/<some identifier>/private_key.json`.

You also need `openssl`, `python3` and `conv.py`.

The command you'll want to execute is:

```
# convert the key to a standard format
$ openssl asn1parse -noout -out private_key.der -genconf <(python3 conv.py private_key.json)
# dump to base64
$ cat private_key.der | base64 --wrap=0
```

Now on screen you have the private key in base64 format in a way that can be pasted inside the `PrivateKey` field.

### `domain` (5)

The domain for which you had the certificate

### `certificate` (6)

This is the public key certificate for your keypair.

You can find the old one here: `<swag>/etc/letsencrypt/live/<domain>/fullchain.pem`.

You can dump it to the console and base64 it:

```
# dump to base64
$ cat fullchain.pem | base64 --wrap=0
```

Note: you _MAY_ need to put in linebreaks between each certificate before converting it to `base64`.

And c/p that in `certificate`.

### `key` (7)

For this one you'll want to take the private key in your domain folder and convert it to the format Traefik wants:

```
$ openssl pkcs8 -topk8 -nocrypt -in privkey.pem | base64 --wrap=0
```

And paste that in in `key`!


