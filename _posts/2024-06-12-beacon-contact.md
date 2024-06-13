---
layout: default  
title: Beacon Contact  
date: 2024-06-12  
categories: [3dprinting, firmware]  
tags: [3dprinting, firmware, klipper, beacon]  
description: Setting Up Beacon Contact  
---

# Beacon

![Beacon Rev H](https://raw.githubusercontent.com/frankzotynia10/mayfairlabs.github.io/main/assets/img/posts/beacon-contact/RevD-her.png)  
photo: [beacon3d.com](https://beacon3d.com/)

Creating a map of a build surface is nothing new. There have been a number of options dating back years, but a few months ago, something caught my eye.

Beacon... It's an eddy current surface scanner that speeds up bed probing significantly. Instead of the probe coming down at predetermined spots to take a measurement, the Beacon can just run along the top of the surface and grab its measurements. It's fast, accurate, and easy to set up.

In fact, all of the firmware flashing is scripted. The hardest part about setting up Beacon is figuring out where to mount it. It does require a no-metal zone directly above the scanner, so keep that in mind. Since I'm using Stealthburner on my printer, Annex Engineering already created a mount for Beacon that integrates into the X carriage.

[Stealthburner Beacon Mount](https://github.com/Annex-Engineering/Annex-Engineering_User_Mods/tree/main/Printers/Non_Annex_Printers/VORON_Printers/VORON_V2dot4/annex_dev-stealthburner_beacon_x_carriage 'STL Download')

Once the part is printed and installed, run the USB cable back to your Raspberry Pi. No canbus for Beacon, unfortunately. SSH into the Pi and run this:

{% highlight bash %}
cd ~
git clone https://github.com/beacon3d/beacon_klipper.git
./beacon_klipper/install.sh
{% endhighlight %}

Now in Mainsail, go to `moonraker.conf` and add this block to the bottom. This will keep the Beacon software up to date with the rest of your system.

{% highlight bash %}
[update_manager beacon]
type: git_repo
channel: dev
path: ~/beacon_klipper
origin: https://github.com/beacon3d/beacon_klipper.git
env: ~/klippy-env/bin/python
requirements: requirements.txt
install_script: install.sh
is_system_service: False
managed_services: klipper
info_tags:
  desc=Beacon Surface Scanner
{% endhighlight %}

# Klipper Configuration

Run `lsusb` on the Pi and copy the serial for your Beacon. Then in Mainsail, edit your `printer.cfg` with the following:

{% highlight bash %}
[beacon]
serial: /dev/serial/by-id/usb-Beacon_Beacon_RevD_<..addyourserial..>-if00
x_offset: 0 # update with offset from nozzle on your machine. This is my Stealthburner setup.
y_offset: 31 # update with offset from nozzle on your machine. This is my Stealthburn
{% endhighlight %}

The offsets are important, and here's why. When Beacon calibrates for contact, it will probe three times and then move the nozzle to the probe location to take its final measurement. It needs to know where that probe is. Also, it's important for the printer to know so it doesn't try to probe off the bed.

- **Main mesh direction**: Do you want it to scan left to right (x) or front to back (y)?
- **Mesh runs**: This number is how many times it will scan the surface. 2 is default, and it will run the print head in one direction and then back the other way.

For the initial functionality test, let's set up `safe_z_home`. This will ensure that the print head is in the center of the bed whenever the Z is homed. Otherwise, the probe could be hanging off the side and crash the head.

{% highlight bash %}
[safe_z_home]
home_xy_position: 130, 130 # update for your machine
z_hop: 5
{% endhighlight %}

Now, we need to update the Z endstop to use the probe.

{% highlight bash %}
[stepper_z]
endstop_pin: probe:z_virtual_endstop # use Beacon as virtual endstop
homing_retract_dist: 0 # Beacon needs this to be set to 0
{% endhighlight %}

If you have a Rev H Beacon, you can configure the accelerometer for input shaping.

{% highlight bash %}
[resonance_tester]
accel_chip: beacon
probe_points: 130, 130, 20 # center of your bed. 20 on the Z
{% endhighlight %}

Make sure to remove any old `[probe]` configurations from your `printer.cfg`. Save and restart.

# Calibration

Home X and Y:

{% highlight bash %}
G28 X Y
{% endhighlight %}

Move the nozzle to the center of the bed:

{% highlight bash %}
G0 X130 Y130
{% endhighlight %}

Now we need to run a calibration:

{% highlight bash %}
BEACON_CALIBRATE
{% endhighlight %}

Grab a piece of paper and put it under the nozzle. Bring the nozzle down until you can feel it dragging. Once you feel like it's in a good spot, type this into the console:

{% highlight bash %}
ACCEPT
SAVE_CONFIG
{% endhighlight %}

The Beacon software will store a bunch of configs in your `printer.cfg` SAVE_CONFIG at the bottom.

# Testing

As with any probe, you have to prepare for the worst. In this case, it's the tool head crashing into the bed. Have your hand on the power switch just in case something goes south.

Home XYZ. Watch it carefully as the tool head nears the bed. If it looks like it's not going to stop, kill the power and ensure everything is mounted properly.

{% highlight bash %}
G28 X Y Z
{% endhighlight %}

Let's check the accuracy:

{% highlight bash %}
PROBE_ACCURACY
{% endhighlight %}

This should be a very small number <0.001.

Finally, let's check the backlash in your Z leadscrew(s):

{% highlight bash %}
BEACON_ESTIMATE_BACKLASH
{% endhighlight %}

Again, this should be a very small number. If it's high, check that the anti-backlash nut is installed properly or replace your leadscrews and nuts.

# Bed Mesh

Everyone's favorite. Let's see how this thing moves.

In `printer.cfg`, add this block:

{% highlight bash %}
[bed_mesh]
algorithm: bicubic
horizontal_move_z: 5
zero_reference_position: 130, 130 # center of your bed
speed: 200 # This can be increased later to your max printer velocity. Keep it slow for initial testing.
mesh_min: 33,65 # Change this to your min!!
mesh_max: 255,260 # Change this to your max!!
probe_count: 51, 51
fade_start: 1.0
fade_end: 10
fade_target: 0
{% endhighlight %}

Some notes about `mesh_min` and `mesh_max`. Home X and Y and move the print head so Beacon is completely on the bed surface in all directions. This is your `mesh_min`.

Then move X and Y to their max position and dial it back until Beacon is on the bed. This is your `mesh_max`.

`probe_count` is how many rows and columns it will collect data from. The higher the number, the more accurate it will be, but the longer it will take to generate a mesh. Keep this number odd so there is a point directly in the center of your bed.

Fade is optional. These are Klipper defaults:

{% highlight bash %}
BED_MESH_CALIBRATE
{% endhighlight %}

Now sit back and watch it fly.

# Contact

Now that everything is operating as intended, let's set up contact. The Beacon docs have a lot to say about it because it can potentially cause damage to your printer if you're not careful. Up until now, there has been no nozzle-to-bed contact at all. This is about to change. BE WARNED!

Contact requires a clean nozzle to be accurate. Keep the nozzle clean with a brass brush before probing. This will be difficult with something like PETG, as it tends to ooze a lot.

In `printer.cfg`, under `[beacon]`, replace it with this:

{% highlight bash %}
[beacon]
serial: /dev/serial/by-id/usb-Beacon_Beacon_RevD_<..addyourserial..>-if00
x_offset: 0 # update with offset from nozzle on your machine
y_offset: 31 # update with offset from nozzle on your machine
mesh_main_direction: x
mesh_runs: 2
contact_max_hotend_temperature: 180 # increase to probe at print temps
home_xy_position: 130, 130 # update with your safe position
home_z_hop: 5
home_z_hop_speed: 30
#home_xy_move_speed: 300
home_method: contact # use proximity for induction homing
home_method_when_homed: proximity # after initial calibration use induction
home_autocalibrate: unhomed # contact will calibrate beacon on first home
{% endhighlight %}

- `contact_max_hotend_temperature`: Set this to your print temp.
- `home_method`: On first home, the nozzle will come down slowly and actually touch the bed.
- `home_method_when_homed`: Once Z is homed, it will just come down as normal, just above the bed.
- `home_autocalibrate`: This will run an autocalibrate when it's unhomed. The nozzle will come down three times and then move the nozzle to the offset location and probe.

Now we need to either remove safe_z_homing or comment it out.  Beacon will handle that move now.

# Required Start Gcode

Here is where my opinion differs from the Beacon docs. They suggest putting the contact commands in the slicer start gcode. I don't like to put anything printer-specific in there. I have a tendency to forget to change profiles, and I don't want anything to crash or cause damage. So instead, I will put all of this stuff in my PRINT_START macro.

Well, I would, but I'm using a great add-on from jschul called [klipper-macros](https://github.com/jschuh/klipper-macros).  t's certainly a topic for another day, but it basically breaks out the PRINT_START macro into a bunch of smaller macros that get called sequentially.

First, let's look at what Beacon suggests our start gcode looks like.

{% highlight bash %}
BED_MESH_CLEAR
SET_GCODE_OFFSET Z=0

G28     ; home axes
G0 Z2   ; position beacon at 2mm for heat soak

M140 S{BED}     ; start bed heater
M109 S150       ; preheat nozzle to probing temp
M190 S{BED}     ; wait on bed temperature
G4 P60000       ; optional, let temps settle for 1 min

WIPE_NOZZLE     ; call another macro to wipe nozzle if available

G28 Z METHOD=CONTACT CALIBRATE=1    ; calibrate z offset and beacon model hot
Z_TILT_ADJUST                       ; or QGL to balance towers
BED_MESH_CALIBRATE RUNS=2           ; bed mesh in scan mode

WIPE_NOZZLE
G28 Z METHOD=CONTACT CALIBRATE=0    ; calibrate z offset only after tilt/mesh

M104 S{EXTRUDER}                    ; set extruder to print temp
M109 S{EXTRUDER}                    ; wait for extruder temp

SET_GCODE_OFFSET Z=0.06     ; add a little offset for hotend thermal expansion
                            ; needs fine tuning, long meltzones require more
SET_GCODE_OFFSET Z_ADJUST={OFFSET}  ; apply optional material squish via slicer

PRIME_NOZZLE    ; call another macro to purge or prime nozzle
; start printing
{% endhighlight %}

- <span style="color:red">The first commands we need to pull out of here is to clear the mesh and set the offset to 0.</span>
- <span style="color:red">Move probe to 2mm to heatsoak it</span>
- <span style="color:green">Preheating the nozzle and bed will be done by KM.</span>
- <span style="color:red">contact calibrate will need to be added</span>
- <span style="color:green">z_tilt_adjust and bed mesh are handled by KM</span>
- <span style="color:red">bump the z offset</span>
- <span style="color:green">purge is handled by KM</span>

Everything in green is handled by the start KM macro. Everything in red is what we will need to override.

# Interpreting PRINT_START Macros

Next, let's take a look at my PrusaSlicer Start Gcode. There is nothing fancy in here. Just what is required from klipper-macros.

{% highlight bash %}
SET_PRINT_STATS_INFO TOTAL_LAYER=[total_layer_count]

_PRINT_START_PHASE_INIT EXTRUDER={first_layer_temperature[initial_tool]} BED=[first_layer_bed_temperature] MESH_MIN={first_layer_print_min[0]},{first_layer_print_min[1]} MESH_MAX={first_layer_print_max[0]},{first_layer_print_max[1]} LAYERS={total_layer_count} NOZZLE_SIZE={nozzle_diameter[0]}
; Insert custom gcode here.
_PRINT_START_PHASE_PREHEAT
; Insert custom gcode here.
_PRINT_START_PHASE_PROBING
; Insert custom gcode here.
_PRINT_START_PHASE_EXTRUDER
; Insert custom gcode here.
_PRINT_START_PHASE_PURGE
{% endhighlight %}

You can see that the start macro is phased into different sections. Let's break this down a little bit.

- `SET_PRINT_STATS_INFO TOTAL_LAYER=[total_layer_count]` - this is for Mainsail and klipperscreen to get layer counts.  This does not need to change.

- `_PRINT_START_PHASE_INIT EXTRUDER` - this passes through variables from your slicer.  Lets leave this one alone as well.

- `_PRINT_START_PHASE_PREHEAT` - This preheats the nozzle and bed.  Lets move Beacon in position to heat soak here.

- `_PRINT_START_PHASE_PROBING` - This starts `z_tilt_adjust` and probing.  Here is where I will want to insert the clear mesh, clear z_offset, and run a probe calibrate. Then we can call the original probing macro and insert a z_offset change once its done with z_tilt and probing.

- `_PRINT_START_PHASE_EXTRUDER` - This brings the nozzle up to print temps. No need to change this.

- `_PRINT_START_PHASE_PURGE` - this does an adapative purge. No need to change this.

# Overriding the Macros

In order to override this KM macro, we need to get into `macros.cfg` in Mainsail. At the bottom of this file, let's add these blocks.

This first one will override the PREHEAT macro to move our probe into position and wait for it to heatsaok.

{% highlight bash %}
[gcode_macro _PRINT_START_PHASE_PREHEAT]
rename_existing: KM_PRINT_START_PHASE_PREHEAT
gcode:
 # Custom commands to run before the original _PRINT_START_PHASE_PREHEAT
  
# Call the renamed original _PRINT_START_PHASE_PREHEAT macro
  KM_PRINT_START_PHASE_PREHEAT {rawparams}

# Custom commands to run after the original _PRINT_START_PHASE_PREHEAT (if any)
  G1 X130 Y130 ; move x and y to middle of the bed
  G0 Z2 ; position beacon over bed
  M118 ** Heatsoaking Beacon **
  TEMPERATURE_WAIT sensor="temperature_sensor beacon_coil" MINIMUM=50 MAXIMUM=69
  WAIT_FOR_CHAMBER_TEMPERATURE
{% endhighlight %}

The breakdown:
- `[gcode_macro _PRINT_START_PHASE_PREHEAT]` - We're declaring a macro called `_PRINT_START_PHASE_PREHEAT`.

- `rename_existing: KM_PRINT_START_PHASE_PREHEAT` - Since we already have a macro called `_PRINT_START_PHASE_PREHEAT`, we need to rename the original to something else. KM = klipper-macros.  This allows us to inject commands at the start and end of the macro and then call the original in the middle.

- `KM_PRINT_START_PHASE_PREHEAT {rawparams}` - Call the original KM Macro.

- `G1 X130 Y130` - This puts the toolhead in the middle of the bed.
- `G0 Z2` - Puts Z height at 2.  This will put the nozzle right next to the bed.
- `M118 ** Heatsoaking Beacon **` - this is a console output to show whats going on.

- `TEMPERATURE_WAIT sensor="temperature_sensor beacon_coil" MINIMUM=50 MAXIMUM=69` - This command will wait until the beacon_coil temp to reach at least 50c.
- `WAIT_FOR_CHAMBER_TEMPERATURE` - this is a chamber temp macro I have setup.  This can be ignored for this setup.

Next, we will override the probe macro.

{% highlight bash %}
[gcode_macro _PRINT_START_PHASE_PROBING]
rename_existing: KM_PRINT_START_PHASE_PROBING
gcode:
  # Custom commands to run before the original _PRINT_START_PHASE_PROBING
  BED_MESH_CLEAR
  SET_GCODE_OFFSET Z=0
  WIPE_NOZZLE
  G28 Z METHOD=CONTACT CALIBRATE=1    ; calibrate z offset and beacon model hot

  # Call the renamed original _PRINT_START_PHASE_PROBING macro
  KM_PRINT_START_PHASE_PROBING {rawparams}

  # Custom commands to run after the original _PRINT_START_PHASE_PROBING (if any)
  WIPE_NOZZLE
  G28 Z METHOD=CONTACT CALIBRATE=1    ; calibrate z offset only after tilt/mesh
  SET_GCODE_OFFSET Z=0.02 ; add a little offset for hotend thermal expansion
{% endhighlight %}

The breakdown:

- `BED_MESH_CLEAR` - clear any default mesh.
- `SET_GCODE_OFFSET Z=0` - sets z_offset to 0.
- `WIPE_NOZZLE` - wipe the nozzle on the brush.  This is a macro to wipe the nozzle.  If you dont' have one, you can leave it and klipper will ignore it.  If you do have one, change this to whatever you macro is called.
- `G28 Z METHOD=CONTACT CALIBRATE=1` - run the beacon calibrate.

- `KM_PRINT_START_PHASE_PROBING {rawparams}` - Call the original probing macro.

- `WIPE_NOZZLE` - wipe the nozzle again
- `G28 Z METHOD=CONTACT CALIBRATE=1` - run a second calibrate after the z_tilt has been ran.
- `SET_GCODE_OFFSET Z=0.02` - this will bump up the nozzle a hair to compensate for nozzle expansion once its heated. This should be tweaked based on your machine.

Some notes from the beacon docs.  `G28 Z METHOD=CONTACT CALIBRATE=1` will use contact to find zero offset, and then calibrate a beacon model for scan mode.  Alternatively, you can use `G28 Z METHOD=CONTACT CALIBRATE=0` to skip the beacon model creation to save time.  THe choice is yours here.

Save `macros.cfg` and then restart..

# Test the new macro

Send a file to the printer and kick it off.  Tweak anything required after that. 

After all this, I probably should have just forked klipper-macros...