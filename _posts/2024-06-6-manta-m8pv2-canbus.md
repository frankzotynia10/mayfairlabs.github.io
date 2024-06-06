---
layout: default
title: Katapult and Klipper Setup for Bigtreetech Manta M8p v2
date: 2024-06-6
categories: [3dprinting, firmware]
tags: [3dprinting,firmware,klipper]
description: Flashing katapult and klipper on a m8p board
media_subpath: /assets/img/posts
---

# Katapult and Klipper Setup for Bigtreetech Manta M8p v2

I am always fumbling when flashing a bootloader and firmware to my Manta M8P v2 board. I run a canbus setup and the BTT documentation does not cover it.

## Canbus Board Prep
Place jumper on 120 ohm resistor pins.  These pins can be found just south of the CPU.
![alt text](</assets/img/posts/2024-06-6/BIGTREETECH MANTA M8P V2.0 120R.png>)

## Clone and build katapult on the Raspberry Pi
We need to build the bootloader. To do this, we will use Katapult.

SSH into your pi and run this.

{% highlight bash %}
cd ~
git clone https://github.com/Arksine/Katapult
cd katapult
make menuconfig
{% endhighlight %}

When the config comes up, set the parameters like this

![alt text](/assets/img/posts/2024-06-6/katapult_m8pv2.png)

Q to exit, Y to save

Then run

{% highlight bash %}
make clean
make
{% endhighlight %}

## Build Klipper
Next, we need to build klipper for the M8Pv2.

{% highlight bash %}
cd ~/klipper
make menuconfig
{% endhighlight %}

When the config comes up, set the parameters like this

![alt text](/assets/img/posts/2024-06-6/klipper_m8pv2.png)

Q to exit, Y to save

Then run

{% highlight bash %}
make clean
make
{% endhighlight %}

## DFU Mode
No we have to get the M8P in DFU mode to flash the bootloader.

On the board, you will find three buttons labled Boot0, Boot1 and Reset.  I use a plastic cap on a ballpoint pen to hold down Boot0 and then just tap reset once.
![alt text](/assets/img/posts/2024-06-6/m8pv2.jpg]

Next on the pi, type

{% highlight bash %}
lsusb
{% endhighlight %}

If the board went into DFU mode properly, you will see the STM board in DFU Mode.

{% highlight bash %}
Bus 001 Device 008: ID 0483:df11 STMicroelectronics STM Device in DFU Mode
Bus 001 Device 004: ID 046d:082d Logitech, Inc. HD Pro Webcam C920
Bus 001 Device 006: ID 04d8:e72b Microchip Technology, Inc. Beacon RevH
Bus 001 Device 002: ID 1a40:0101 Terminus Technology Inc. Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
{% endhighlight %}

If you see OpenMoko, Inc. Geschwister Schneider CAN adapter, try it again.

{% highlight bash %}
Bus 001 Device 007: ID 1d50:606f OpenMoko, Inc. Geschwister Schneider CAN adapter
Bus 001 Device 004: ID 046d:082d Logitech, Inc. HD Pro Webcam C920
Bus 001 Device 006: ID 04d8:e72b Microchip Technology, Inc. Beacon RevH
Bus 001 Device 002: ID 1a40:0101 Terminus Technology Inc. Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
{% endhighlight %}

Stop the klipper service with this

{% highlight bash %}
sudo service klipper stop
{% endhighlight %}

## Lets Flash
We're going to start with the bootloader first.  Run this command.

{% highlight bash %}
sudo dfu-util -a 0 -D ~/katapult/out/katapult.bin --dfuse-address 0x08000000:force:leave -d 0483:df11
{% endhighlight %}

You may get a DFU error here.  As long as it downloaded completely, it should be fine.

Press the reset button on the board and wait a few seconds.  Then press Boot0 and tap reset to reenter DFU mode.

Next we will flash klipper with the following command.

{% highlight bash %}
sudo dfu-util -a 0 -d 0483:df11 --dfuse-address 0x08002000 -D ~/klipper/out/klipper.bin
{% endhighlight %}

*Note:* If you get an error, its most likely a misconfiguration.  Rerun the configuration from klipper step above.

Press the reset button one last time and run

{% highlight bash %}
lsusb
{% endhighlight %}

If you see OpenMoko, Inc. Geschwister Schneider CAN adapter, you're all set and we can move on to the next step. If you're not seeing the board, start over.
{% highlight bash %}
Bus 001 Device 007: ID 1d50:606f OpenMoko, Inc. Geschwister Schneider CAN adapter
Bus 001 Device 004: ID 046d:082d Logitech, Inc. HD Pro Webcam C920
Bus 001 Device 006: ID 04d8:e72b Microchip Technology, Inc. Beacon RevH
Bus 001 Device 002: ID 1a40:0101 Terminus Technology Inc. Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
{% endhighlight %}

Lets start klipper back up
{% highlight bash %}
sudo service klipper start
{% endhighlight %}

## Can Network Setup

First we will configure ifupdown.  Edit this file.

{% highlight bash %}
sudo nano /etc/network/interfaces.d/can0
{% endhighlight %}

and put this in that file.

```bash
allow-hotplug can0
iface can0 can static
 bitrate 250000
 up ifconfig $IFACE txqueuelen 1024
```

Next, netplan is up. Edit this file.

{% highlight bash %}
sudo nano /etc/systemd/network/10-can.link
{% endhighlight %}

and put this in there.

```bash
[Match]
Type=can

[Link]
TransmitQueueLength=1024
```

Then edit this file.
{% highlight bash %}
sudo nano /etc/systemd/network/25-can.network
{% endhighlight %}

and put this in there.

```bash
[Match]
Name=can*

[CAN]
BitRate=1M
```

## Find the canbus ID

In order for the pi to find your board, we need to provide it with a canbus id in the printer.cfg file. This is easily accomplished using a handy katapult script.

{% highlight bash %}
cd ~/katapult/scripts
python3 flash_can.py -i can0 -q
{% endhighlight %}

The output should look something like this.

{% highlight bash %}
Resetting all bootloader node IDs...
Checking for Katapult nodes...
Detected UUID: 96d2ef01b2cc, Application: Klipper
Query Complete
{% endhighlight %}

Copy that UUID and open up the printer.cfg in Mainsail.  Format it like this.

```bash
[mcu]
#   Configuration of the primary micro-controller.
canbus_uuid: 96d2ef01b2cc
canbus_interface: can0
```

## Final Step
reboot and verify Mainsail can connect to the board.