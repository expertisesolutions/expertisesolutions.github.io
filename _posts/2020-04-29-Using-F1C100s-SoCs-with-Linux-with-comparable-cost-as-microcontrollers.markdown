---
layout: post
title:  "Using F1C100s SoCs with Linux with comparable cost as microcontrollers"
date:   2021-04-29 10:35:47 -0300
categories: buildroot yocto allwinner f1c100s f1c600s f1c200s linux microcontrollers arm
image: "/assets/img/2020-04-29-Using-F1C100s-SoCs-with-Linux-with-comparable-cost-as-microcontrollers.jpg"
share-img: "/assets/img/2020-04-29-Using-F1C100s-SoCs-with-Linux-with-comparable-cost-as-microcontrollers.jpg"
author: Felipe Magno de Almeida
---

What if you could use Linux in all your embedded projects? A good news
is that cost is the most important factor to consider a
microcontroller, then you should consider using a F1C100s SoC instead,
with a price of less than $2 it is really competitive even in the
microcontroller space when you factor the added RAM.

# About F1C100s

The F1C100s is a SoC from Allwinner which has a very interesting
compromise on features, price and a LQFP128 package. It has a
ARM926-EJS core which runs up to 900MHz with USB 2.0, LCD controller,
32MB RAM DDR1, a video decoder engine with 720p H264 encoding and
decoding, PWM, UART, ADC and audio support.

# Use cases

This SoC can be used in various applications: 

* Small screen devices such as smart light switches
  with touchscreen, digital picture frames, home automation touch
  screen switches
* Audio applications
* IP Camera's motion sensor or intelligent intrusion detection
* Specialized USB-devices
* Automation with relay-switching, or RS232/PWM/I2C
  device automation with a wireless adapter

# LicheePi Nano

So you're hooked and want to start prototyping. How do you do?
Fortunately, Seeed Studio manufactures the LicheePi Nano SBC and sells
for less than $8 with the size of a SD card with a F1C100s with
several I/Os, a microSD card reader, a 16MB SPI flash and a 40-pin RGB
LCD connector

They also have the LicheePi Zero with a bigger footprint, a V3S SoC
with 64MB RAM.

You can attach a 40-pin RGB LCD on the LicheePi Nano or a serial
adapter and run the linux already embedded in flash.

Or, you can build your own Linux distribution with a newer kernel.

# Buildroot

There are multiple buildroot projects available in GitHub for the
F1C100s CPU. Some of them are:

* https://github.com/florpor/licheepi-nano
* https://github.com/unframework/licheepi-nano-buildroot

And even a Yocto meta layer, which I haven't tested yet. But which I
want to try, since I do like Yocto better than Buildroot.

* https://github.com/voloviq/meta-licheepinano

You can also use our buildroot fork from florpor at
https://github.com/expertisesolutions/licheepi-nano

# Building

To build I do recommend you use a Docker container so you don't have
problems with dependencies. I use the following Dockerfile:

{% highlight Docker %}
FROM ubuntu:bionic

# Upgrade system and Yocto Proyect basic dependencies
RUN apt update
RUN apt -y install build-essential debconf locales
RUN apt -y install swig python-dev python3-dev fakeroot devscripts chrpath gawk texinfo libsdl1.2-dev whiptail diffstat cpio libssl-dev file wget rsync bc ncurses-dev

ENV TZ=America/Sao_Paulo
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# Set up locales
RUN localedef -c -i en_US -f UTF-8 en_US.UTF-8
ENV LANG en_US.utf8

RUN apt -y install python-matplotlib

# User management                                                                
RUN groupadd -g 1000 build && useradd -u 1000 -g 1000 -ms /bin/bash build && usermod -a -G users build

# Run as build user from the installation path                                   
USER build

# Make /home/build the working directory                                         
WORKDIR /home/build
{% endhighlight %}

Which I run as:

{% highlight Bash %}
docker run -it --mount type=bind,source=/home/felipe/dev/licheepi,target=/home/build lichee-dev:latest /bin/bash
{% endhighlight %}

After cloning you must remember to initialize and update all submodules so the repository can download buildroot itself.

{% highlight Bash %}
git submodule update --init
{% endhighlight %}

To compile you first configure buildroot and then compile it

{% highlight Bash %}
$ cd buildroot
$ make BR2_EXTERNAL=$PWD/../ licheepi_nano_mmc_defconfig
$ make
{% endhighlight %}

This will take some time while it downloads all packages and compile each one.

# Flashing

First you need to erase the device's flash so you can get the board in
FEL mode. With a serial adapter or with a keyboard and LCD, you must
type the following at the U-Boot shell:

{% highlight Bash %}
=> sf probe 0
=> sf erase 0 0x10000
=> reset
{% endhighlight %}

To flash to device's SPI flash you must use sunxi-tools. In Arch Linux
you will want to use sunxi-tools-f1c100s-spiflash-git from AUR.

{% highlight Bash %}
$ sudo sunxi-fel -p spiflash-write 0 output/image/flash.bin
{% endhighlight %}

Please note you _must_ replace `/dev/mmcblkX` with the correct device
of the sd card you want to write, otherwise you may lose all your data
on your computer.

And to finally flash the SD card you must use dd with the sdcard.img:

{% highlight Bash %}
$ sudo dd if=output/images/sdcard.img of=/dev/mmcblkX
{% endhighlight %}


# Conclusion

With the right tools, like buildroot and Docker, you can start fast to
prototype your embedded applications on top of LicheePi Nano SBCs and
can develop very good devices with a lower cost in hardware and an
even lower cost in development time and time-to-market by leveraging
Linux operating system and higher level languages such as Python.

Let me know in the comments if you have a LicheePi Nano or Zero, and
what you intent to do with it!


