---
author: Kristof
comments: true
date: 2014-10-27 18:43:08+00:00
layout: post
link: http://kristofmatteibe.azurewebsites.net/2014/10/27/enabling-dynamic-compression-gzip-webapi-iis/
slug: enabling-dynamic-compression-gzip-webapi-iis
title: Enabling dynamic compression (gzip) for WebAPI and IIS
wordpress_id: 2244
categories:
- Programming
---

A lot of code on the internet refers to writing custom ActionFilters, or even HttpHandlers that will compress your return payload for you.

For example, see this [package](https://github.com/azzlack/Microsoft.AspNet.WebApi.MessageHandlers.Compression/) (which with its name implies that it is Microsoft, but then says it's not Microsoft).

At the moment of writing the above-linked package even throws an error when you return a 200 OK without a body...

But in the end, it's very simple to enable compression on your IIS server without writing a single line of code:

You first need to install the IIS Dynamic Content Compression module:

{% include image.html name="dynamic_content_compression.png" %}

Or, if you're a command line guy, execute the following command in an elevated CMD:

    
```
dism /online /Enable-Feature /FeatureName:IIS-HttpCompressionDynamic
```


Next up you need to enable the Dynamic Content Compression to compress

    
```
application/json
```


and

```
application/json; charset=utf-8
```


To do this, execute the following commands in an elevated CMD:

```
cd c:\Windows\System32\inetsrv

appcmd.exe set config -section:system.webServer/httpCompression /+"dynamicTypes.[mimeType='application/json',enabled='True']" /commit:apphost
appcmd.exe set config -section:system.webServer/httpCompression /+"dynamicTypes.[mimeType='application/json; charset=utf-8',enabled='True']" /commit:apphost
    


This adds the 2 mimetypes to the list of types the module is allowed to compress. Validate that they are added with this command:

    
```
appcmd.exe list config -section:system.webServer/httpCompression
```

Validate that the 2 mimetypes are there and enabled:

{% include image.html name="cmd.png" %}

And lastly, you'll probably need to restart the Windows Process Activation Service.

Best is to do this through the UI because I have yet to find a way in CMD to restart a service (can't seem to start services that are dependent on the one we just started).

In services.msc you'll need to search for Windows Process Activation Service. Restart it.

{% include image.html name="wpas.png" %}

Obviously there are more settings available, take a look at the [httpCompression Element](http://msdn.microsoft.com/en-us/library/ms690689.aspx) settings page.

I recommend reading about 2 at least:
* dynamicCompressionDisableCpuUsage
* noCompressionForProxies


Good luck,

-Kristof
