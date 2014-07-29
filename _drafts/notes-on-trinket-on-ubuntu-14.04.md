---
layout: post
title:  "Personal Notes on get adafruit Trinket to work on Ubuntu 14.04"
date:   2014-07-27 18:41:59
categories: blog
tags: mug
---

I bought a couple of adafruit Trinket some time ago, but i got the time to work on them only yesterday.

As described in their [manual](https://learn.adafruit.com/introducing-trinket/introduction) Linux (and arduino 1.5) are not supported, because sometimes the kernel does not create the device node when the trinket is inserted in the usbport. I suspect the problem lies in the fact that trinket uses only attiny85, and implements the usb stack in software. so probably a subtle bug in linux or in the trinket prevents a proper handshake. 
The first time i tried using this board, i was using Ubuntu 13.10, and i believe i had the same problem. Having Upgraded to Ubuntu 14.04, i hoped the situation somewhat improved, as it did.

now running `lsusb` return also `Bus 003 Device 032: ID 1781:0c9f Multiple Vendors USBtiny` which is the idetifier

