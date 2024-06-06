---
layout: default
title: Katapult and Klipper Setup for Bigtreetech Manta M8p v2
date: 2024-06-6
categories: [3dprinting, firmware]
tags: [3dprinting,firmware,klipper]
description: Flashing katapult and klipper on a m8p board
media_subpath: /assets/lib/
---

# Katapult and Klipper Setup for Bigtreetech Manta M8p v2

I am always fumbling when flashing a bootloader and firmware to my Manta M8P v2 board. I run a canbus setup and the BTT documentation does not cover it.

## Canbus Board Prep
Place jumper on 120 ohm resistor pins.  These pins can be found just south of the CPU.

![img-description](/assets/lib/BIGTREETECH%20MANTA%20M8P%20V2.0%20120R.png)
