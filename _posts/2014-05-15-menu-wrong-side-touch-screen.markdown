---
author: Kristof
date: 2014-05-15 19:21:52+00:00
layout: post
title: Menu on the wrong side with a touch screen?
wordpress_id: 2146
categories:
- '7'
- '8'
- '8.1'
- OS
- Various
- Windows
---

When you're reading this you probably have a touch screen.

So, I never use my touch screen. Almost never. But I did notice that by default my menus in Windows (from a menu bar, not a ribbon) appear (when possible) on the right side of the clicked menu item.

Like this:

{% include image.html name="menu_left.png" %}

Goosebumps. Something is off. It took me a while to realize this,

**The menu expanded to the left!**

So, what is this. I can't remember what exactly I searched for, but the change you need to make is in **Tablet PC Settings**.

When your menus are expanded to the left you'll see something like this:

{% include image.html name="right_handed.png" %}

This is different from the default that I've been used to since I've been using Windows 95.

Change it to 'Left-handed':

{% include image.html name="left_handed.png" %}

Hit apply, and restart any offending programs, open a menu and enjoy:

{% include image.html name="menu_right.png" %}

I can rest easy again...
