---
title: "Slackware Post Install"
date: 2021-08-11T17:31:48Z
draft: true
---

## Purpose

Here are the tasks I performed when I have a fresh install of Slackware

## Adding a regular user

    # adduser username
    # usermod -s /bin/zsh username

## Apply the latest patches

Get my getpaches.sh script available on https://github.com/richardpct/slackware/tree/main/scripts
and launch it

## Configuring Lilo

Decrease the timeout to 20 in /etc/lilo.conf then perform the following
command:

    # lilo -v

## Configuring the locale

Edit the /etc/profile.d/lang.sh file and set the following variable:

    export LANG=en_US

to

    export LANG=en_US.utf8

## Compiling the Kernel

Set the permission of /usr/src in order that a regular user can compile in it:

    # chgroup users /usr/src
    # chmod 775 /usr/src

Get the latest Kernel:

    $ cd /usr/src
    $ wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.13.9.tar.xz

Decompress the kernel archive:

    $ tar xJvf linux-5.13.9.tar.xz

Create a symbolic link to linux:

    $ ln -s linux-5.13.9 linux

Copy your kernel configuration in /usr/src/linux/.config

Configure the kernel:

    $ make menuconfig

When it's done, just exit

Build the kernel:

    $ make -j8

Install the kernel:

    root# make install

Install the kernel modules:

    root# make modules_install

Reboot your computer:

    root# shutdown -r now

## Conguring git

Edit your ~/.gitconfig and add the following:

```
[color]
        ui = always
        diff = always
        branch = always
        interactive = always
        status = always
        grep = always
        pager = true
        decorate = always
        showbranch = always
[core]
        pager = less -R
```

