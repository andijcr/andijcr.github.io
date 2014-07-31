---
layout: post
title:  "Personal Notes on get adafruit Trinket to work on Ubuntu 14.04"
date:   2014-07-27 18:41:59
categories: blog
tags: mug
---

I bought a couple of adafruit Trinket some time ago, but i got the time to work on them only yesterday.

As described in their [manual](https://learn.adafruit.com/introducing-trinket/introduction), Linux (and arduino 1.5) is not supported, because sometimes the kernel does not create the device node when trinket is inserted in the usbport.

I suspect the problem lies in the fact that trinket uses only attiny85, and implements the usb stack in software. so probably a subtle bug in linux or in the trinket prevents a proper handshake. 

The first time i tried using this board, i was using Ubuntu 13.10, and i believe i had the same problem. Having Upgraded to Ubuntu 14.04, i hoped the situation somewhat improved, as it did.
Now running

	lsusb

return also

	Bus 003 Device 032: ID 1781:0c9f Multiple Vendors USBtiny

which is the right vendorId:productId identifier, and rightly indicates that the device is a USBtiny programmer. 

After patching the avrdude conf, as described [here](https://learn.adafruit.com/introducing-trinket/setting-up-with-arduino-ide) (i used meld to merge the section about attiny85)
the command 

	$ sudo avrdude -c usbtiny -p m8

returns

	avrdude: AVR device initialized and ready to accept instructions

	Reading | ################################################## | 100% 0.00s

	avrdude: Device signature = 0x1e930b
	avrdude: Expected signature for ATmega8 is 1E 93 07
			 Double check chip, or use -F to override this check.

	avrdude done.  Thank you.

this means that the trinket should be ready to be programmed!

let's fire up the blink program, from the examples:

{% highlight c++ %}
/*
Blink
Turns on an LED on for one second, then off for one second, repeatedly.
This example code is in the public domain.
 
To upload to your Gemma or Trinket:
1) Select the proper board from the Tools->Board Menu
2) Select USBtinyISP from the Tools->Programmer
3) Plug in the Gemma/Trinket, make sure you see the green LED lit
4) For windows, install the USBtiny drivers
5) Press the button on the Gemma/Trinket - verify you see
the red LED pulse. This means it is ready to receive data
6) Click the upload button above within 10 seconds
*/
int led = 1; // blink 'digital' pin 1 - AKA the built in red LED
 
// the setup routine runs once when you press reset:
void setup() {
// initialize the digital pin as an output.
	pinMode(led, OUTPUT);
}
 
// the loop routine runs over and over again forever:
void loop() {
	digitalWrite(led, HIGH);
	delay(1000);
	digitalWrite(led, LOW);
	delay(1000);
}
{% endhighlight %}

as avrdude still requires root to use the device, i use the arduino ide to verify the code. this action produces an uploadable hex in the temp build folder. grab the binary - named sketch_jul30a.cpp.hex in my case - and the command to write the program is this: 

	$ sudo avrdude  -c usbtiny -p attiny85 -U flash:w:sketch_jul30a.cpp.hex

