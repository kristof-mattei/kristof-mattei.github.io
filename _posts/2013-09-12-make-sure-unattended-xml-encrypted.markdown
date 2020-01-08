---
author: Kristof
comments: true
date: 2013-09-12 16:16:20+00:00
layout: post
link: http://kristofmatteibe.azurewebsites.net/2013/09/12/make-sure-unattended-xml-encrypted/
slug: make-sure-unattended-xml-encrypted
title: Make sure unattended.xml is not encrypted!
wordpress_id: 2051
categories:
- '7'
- OS
- Windows
---

I was playing around with Sysprep using unattended.xml when I hit a weird issue with VirtualBox and Encrypted folders on the host.

Setup:



	
  * `unattended.xml` on the host, encrypted (with Windows EFS).

	
  * Virtual Machine, hosted in VirtualBox


I mounted the folder with `unattended.xml` (and other files) inside the VirtualBox and started sysprep (`sysprep+shutdown.cmd` just executes the sysprep with the `unattended.xml` from the location and copies a `SetupComplete.cmd` to `c:\Windows\Scripts`).

{% include image.html name="Windows-Setup-encountered.png" caption="Windows Setup encountered an internal error while loading or searching for an unattended answer file." %}

When booting the VM I got the following error:

{% include image.html name="sysprep-files.png" %}

To investigate the error I hit up Shift+F10 and checked `c:\Windows\Panther\setuperr.log`, which had the following error:


<blockquote>[setup.exe] UnattendSearchExplicitPath: Found unattend file at [C:\Windows\Panther\unattend.xml] but unable to deserialize it; status = 0x80070005, hrResult = 0x0.</blockquote>


Googling for the error string didn't help. Googling for the error code did help. It meant [Access Denied](http://support.microsoft.com/kb/816731). Now what could it be. I had a suspicion that it was the encryption. Let's find out:

By using the command

    
    cipher /s:c:\Windows\Panther


I saw this:

{% include image.html name="cipher.png" %}

Notice the `E`, which means, Encrypted.

Executing

    
    notepad c:\Windows\Panther\unattended.xml


confirmed my suspicion:

{% include image.html name="access-denied.png" %}


After removing the file with a Windows disk BEFORE the first boot (afterwards it doesn't work it seems) the boot went fine, I sysprepped it again (with a non-encrypted unattended.xml) and all went fine.

So make sure you don't copy `unattended.xml` to a machine that is encrypted. The keys are lost upon sysprepping!
