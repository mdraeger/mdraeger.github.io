---
layout: post
title: "Fixing a damaged Linux installation"
description: "Fixing a damaged Linux installation"
category: Linux
tags: [Linux]
---
{% include JB/setup %}

Motivation
----------

This post is more a reminder for me rather than providing insight that you don't have already.
Recently, I tried to install the original NVIDA drivers for my Geforce GT440 to play a bit 
with CUDA. The combination of the NVIDA driver and the 3.16 kernel rendered my system pretty 
much useless due to random freezes that occurred even before I could access a shell.

For diagnostics and to fix it, I used a rescue disk from which I accessed my installation 
safely via `chroot`. The details I outline in this post.

{% highlight bash %}
# first, mount the culprit to /mnt
$ sudo mount /dev/sda2 /mnt # or whatever it happens to be in your case
# mount the system fs 
$ sudo mount -o bind /proc /mnt/proc
$ sudo mount -o bind /dev /mnt/dev
$ sudo mount -o bind /dev/pts /mnt/dev/pts
$ sudo mount -o bind /sys /mnt/sys
# for internet access, copy resolv.conf
$ sudo cp /etc/resolv.conf /mnt/etc/resolv.conf
# finally, chroot into your broken installation
$ sudo chroot /mnt /bin/bash
{% endhighlight %}

Now, you can start working on your installation, install/ uninstall/ downgrade packages, etc.

You can also put these commands in a script file and save it on your rescue system to have a
single command solution to switch into your system.

That's it for today, I hope someone finds it useful.
