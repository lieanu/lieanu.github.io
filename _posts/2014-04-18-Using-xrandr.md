---
layout: post
title: Using xrandr and gtf to add a new mode to your X configuration at runtime

---

ProblemYou know that your chipset and monitor are capable of supporting a particular mode (say 1440×900 at 59.9Hz) but that is currently not listed as being supported in the xrandr output. The following steps show how xrandr and gtf can be used to add a new modeline at runtime to your current X configuration. Please note that this is non-persistent. For making it persistent, one option is to add these lines to your /etc/profile file.
SetupThe following has been tested on a Sony Vaio VGN FJ290P Laptop running Ubuntu 8.10 and xrandr version 1.2. The laptop is connected to an external monitor (eMachines E19T6W with max 1440×900 resolution) via VGA. The display hardware is as follows:

00:02.0 VGA compatible controller: Intel Corporation Mobile 915GM/GMS/910GML Express Graphics Controller (rev 03)
00:02.1 Display controller: Intel Corporation Mobile 915GM/GMS/910GML ExpressGraphics Controller (rev 03)
Steps

###Use xrandr to make sure that the new mode can fit within the maximum framebuffer size 

```
neoblitz@n30:~$ xrandr | grep maximum
Screen 0: minimum 320 x 200, current 1440 x 900, maximum 1680 x 1680
The maximum virtual framebuffer size of 1680 x 1680 is sufficient to 
support our 1440 x 900 mode. Please note that this cannot be increased but can be decreased.
```

###Use gtf to create a mode line 

```
neoblitz@n30:~$ gtf 1440 900 59.9# 1440×900 @ 59.90 Hz (GTF) hsync: 55.83 kHz; pclk: 106.29 MHz
Modeline “1440x900_59.90″  106.29  1440 1520 1672 1904  900 901 904 932  -HSync +Vsync
gtf is a program to calculate the VESA GTF (Generalized Timing Formula) mode lines. 
Given the desired horizontal and vertical resolutions and refresh rate (in Hz), 
the parameters for a matching VESA GTF mode are printed out.
```

####Add new mode using xrandr 

```
neoblitz@n30:~$ xrandr --newmode "1440x900_59.90"  106.29  1440 1520 1672 1904  900 901 904 932  -HSync +Vsync
Copy the text after Modeline from the above output as is and paste it after the –newmode option in xrandr.
```

###Add this newly added mode to the desired output (VGA/LVDS etc) 

```
neoblitz@n30:~$  xrandr --addmode VGA 1440x900_59.90
As my monitor is connected via VGA, i will add this new mode to VGA
```

###Choose the new mode

```
neoblitz@n30:~$  xrandr --output VGA --mode 1440x900_59.90
At this point if everything above succeeds without errors then your display would switch to the new mode.
```
