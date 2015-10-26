---
layout: post
title: "Driving Multiple Monitors on an Optimus Laptop"
modified:
categories: articles
excerpt:
tags: [linux, optimus, prime, xrandr, vgaswitcheroo]
image:
  feature:
date: 2014-11-03T21:38:00-00:00
---

## Introduction

If you have an Optimus enabled laptop than it's quite possible it can drive 3 external monitors while using the internal display at the same time, giving you 4 displays in total. I'm creating this page because when I wanted to do exactly this I found a dearth of information on-line, with most of the resources I could find instead talking about using Optimus to allow the Intel and Nvidia GPUs to jointly drive a single display, or two displays at most.

I'm therefore creating this page because I would have found it helpful myself. The fact it's me creating it is unfortunate since my knowledge of Linux, chipsets and GPUs is quite basic, but unfortunately the people that really understand this stuff tend to write in a way that assumes everybody else does too, so this is my attempt at creating a resource for the rest of us. It probably contains some factual inconsistencies, but is hopefully close enough to the truth to be useful.

Since I first wrote this article a few weeks ago, I've learned that my knowledge of Nvidia Prime was incomplete, and so I've since updated it so that it's hopefully a far more useful resource than it previously was.

## Understanding Optimus Laptops

To start with, it's worth describing what an Optimus laptop actually is. Really, it's just a laptop that has both an integrated Intel GPU (on the same die as the processor) and a discrete Nvidia GPU (in a separate chip), where each GPU can drive one or more displays at the same time, or where both GPUs can work in tandem to drive a lesser number of displays. Exactly which display ports each GPU has direct access to varies by laptop, and laptops where there are no jointly accessible display ports are known as _muxless_ laptops, since they don't contain a hardware multiplexer to switch access to the display ports.

If you run the `xrandr` command you'll see the list of display ports available. On my work laptop for example (a Lenovo T430), I see this:

~~~bash
Screen 0: minimum 320 x 200, current 1600 x 900, maximum 32767 x 32767
LVDS1 connected primary 1600x900+0+0 (normal left inverted right x axis y axis) 310mm x 174mm
   1600x900       60.0*+
   1440x900       59.9
   1360x768       59.8     60.0
   1152x864       60.0
   1024x768       60.0
   800x600        60.3     56.2
   640x480        59.9
VGA1 disconnected (normal left inverted right x axis y axis)
VIRTUAL1 disconnected (normal left inverted right x axis y axis)
LVDS-1-2 disconnected
VGA-1-2 disconnected
DP-1-1 disconnected
DP-1-2 disconnected
DP-1-3 disconnected
~~~

Unfortunately, the `xrandr` command doesn't explicitly state which display ports are connected to which GPU, but you can pretty much figure it out by all the extra hyphens added to the port names for the discrete GPU (e.g. `LVDS-1-2` & `VGA-1-2`).

If your laptop has a BIOS setting that allows you to configure the currently active GPUs, then your laptop definitely isn't _muxless_, and it becomes possible to conclusively know which GPU has access to which display ports. For example, I see this if I run `xrandr` with only the Intel GPU enabled:

~~~bash
Screen 0: minimum 320 x 200, current 3520 x 1080, maximum 32767 x 32767
LVDS1 connected 1600x900+0+0 (normal left inverted right x axis y axis) 310mm x 174mm
   1600x900       60.0*+
   1440x900       59.9
   1360x768       59.8     60.0
   1152x864       60.0
   1024x768       60.0
   800x600        60.3     56.2
   640x480        59.9
VGA1 disconnected (normal left inverted right x axis y axis)
VIRTUAL1 disconnected (normal left inverted right x axis y axis)
~~~

versus this when I have only the Nvidia GPU enabled:

~~~bash
Screen 0: minimum 8 x 8, current 3840 x 1080, maximum 16384 x 16384
VGA-0 disconnected (normal left inverted right x axis y axis)
LVDS-0 connected (normal left inverted right x axis y axis)
   1600x900       60.0 +   40.0
DP-0 disconnected (normal left inverted right x axis y axis)
DP-1 disconnected (normal left inverted right x axis y axis)
DP-2 disconnected (normal left inverted right x axis y axis)
~~~
Here, the internal display and the VGA port are accessible by both GPUs, whereas the DVI and mini-display ports are solely accessible by the Nvidia GPU. Again, each laptop does it differently.


## Understanding Optimus

So, if an Optimus laptop has an Intel and an Nvidia GPU, what actually is Optimus? Well, it's Nvidia's name for a proprietary piece of software that is capable of dynamically switching which GPU controls which active display at any time. There are a number of ways this dynamic switching can occur:

  1. Some laptops have a hardware _multiplexer_ (or mux) that allows both of the GPUs to access the same set of display ports, and there is speculation that the Optimus software may also dynamically switch the hardware mux to transfer control of a display port from one GPU to another, assuming Windows can even support this.
  2. For laptops that don't have a hardware mux, or where not all display ports are muxed, then there are two forms of software muxing that can be used:
    1. The first is to have one GPU off-load 3D rendering to another GPU for a particular application.
    2. The second is to have one GPU use another GPU to provide it access to display ports it otherwise wouldn't be able to interface with.

The first of the two software muxing approaches allows a less powerful GPU to be used while mobile, while still making use of a more powerful GPU when actually needed, and the second is what makes it possible to use four displays at once, even though desktops rendered by X Server must render to a primary GPU.

The Linux version of Optimus is called Optimus Prime (a reference to Transformers), and provides limited forms of all three of these types of muxing:

  1. Switcheroo is the Optimus Prime way of switching the hardware mux, but can only be used after `vga_switcheroo` has become available, but before the _boot-splash_ (e.g. Plymouth) or the _display-manager_ (e.g. LightDM) have started.
  2. `xrandr --setprovideroffloadsink <offload-to> <offload-from>` is the Optimus Prime way of offloading rendering from a given GPU to some other GPU, for any programs started with `DRI_PRIME` set to `1`.
  3. `xrandr --setprovideroutputsource <give-ports-to> <take-ports-from>` is the Optimus Prime way of making all the ports reachable by one GPU, available for rendering to by another GPU.

The `--setprovideroffloadsink` and `--setprovideroutputsource` xrandr switches both require xrandr version 1.4. More detailed information is available on the [Optimus Wiki](http://nouveau.freedesktop.org/wiki/Optimus/), but it should be clear to see that Optimus Prime does not yet have the same auto-magic switching logic that the original Nvidia Optimus has, but instead simply makes it possible for switching to occur.


## Linux Multi-GPU Support

Now, whereas Windows has had the ability to span a single desktop over multiple GPUs for over a decade (since Windows 2000), in Linux that has only become possible more recently with the introduction of RandR 1.4.

Here's a potted history of events:

  * X Window System / X11 (1987): X has always supported multiple displays (screens in X11 parlance), but where displays are wholly separated, so that applications can't span a display, or be dragged from one display to another.
  * X11 Release 3 (1988): Display managers become available, allowing desktops to be written where lots of programs can share access to the same display &mdash; effectively on a single monitor since desktop users expect to be able move programs to any part of the desktop.
  * X11R6.4 (1996): The Xinerama extension is added to X11, allowing groups of _screens_ to be composed into a single composite _screen_, so that desktops can span multiple monitors, but with [numerous downsides](http://en.wikipedia.org/wiki/Xinerama#Known_problems).
  * X11R7.5  (2009): The RandR 1.3 extension added support for displays that span multiple monitors on a single GPU.
  * RandR 1.4 (2012): Added support for GPUs to offload 3D work to another GPU, and for GPUs to use the output-sources accessible via foreign GPUs.

As the documentation itself states, "RandR 1.4 adds a way for drivers to work together so that one graphics device can display images rendered by another", and because of architectural limitations within X Server, RandR 1.4 is therefore an enabler for both rendering a single desktop using multiple GPUs, and for allowing a second GPU to provide rendered output for display by the GPU actually driving the monitor.

In the future, when X is replaced by Wayland and Mir, it's possible that multiple GPUs can be used directly, without the need for the offloading and output-source features RandR 1.4 provides.


### Bumblebee & Nvidia Prime

If you've done any on-line search with the keywords 'Optimus' and 'Linux' you will have found numerous references to the following two technologies:

  * Bumblebee
  * Nvidia Prime

Both of these are dependent on the proprietary Nvidia driver, and do things their own way rather than the Linux way. Despite that, it's worth understanding what these technologies provide, and how they work.

The older Bumblebee project provides per-program off-loading of 3D rendering, equivalent to the `xrandr --setprovideroffloadsink` command, but more limited in that only the Intel GPU can off-load to the NVidia GPU. It's been a real boon because it doesn't require RandR 1.4 compatible graphics drivers, and so for a long time was the only option available. Unfortunately, it only allows two displays to be used at the same time, and limits you to the display ports directly accessible from the Intel GPU.

Nvidia Prime is a mechanism provided by more recent versions of the proprietary Nvidia driver that makes use of the RandR 1.4 output source features offered by the Intel driver, while not actually implementing all of RandR 1.4 itself. As the Nvidia documentation states:

> The NVIDIA driver currently only supports the Source Output capability. It does not support render offload and cannot be used as an output sink.

This hasn't been done to be difficult, but has been done because it allows Nvidia to use the same driver kernel on Windows, Mac OS X and Linux.

Nvidia Prime also provides a Switcheroo type mechanism, allowing the the user to switch between the Intel and Nvidia GPUs, though requiring a restart to take affect. If you install 'nvidia-prime' in Ubuntu you get a text based rather than a graphical based start-up, presumably to afford the Switcheroo an opportunity to actually switch GPUs. Because the proprietary Nvidia driver supports neither offloading nor being used as an output-source itself, you are limited to the displays that are reachable via the Intel GPU when in 'Intel (Power Saving)' mode, and 3D render offloading is unavailable too.

My experience with Nvidia Prime has been really positive, though I've only been able to drive 3 displays at once, and not all four. Specifically, I've not been able to properly work displays connected to the VGA port, which on my laptop is the one port that the Nvidia GPU can never directly access. However, the image has been flawless on the two DVI connected monitors, and with only minor lag and slight tearing on the built-in display, where the Intel GPU is used as an output-source.

Ultimately, if gaming is important to you and/or you don't need to drive lots of monitors, you are better off using one of these solutions for the time being.


### The Optimus Prime Way

Theoretically, Optimus Prime is the best of all worlds since it allows:

  * Either the Intel or Nvidia GPU to be made the primary GPU.
  * The secondary GPU to be powered down when not in use.
  * The secondary GPU to be used to off-load 3D rendering when requested to do so.
  * The secondary GPU to be used to make more display ports available to the primary GPU when 3 or 4 displays are needed, or when access to a display port not directly available to the primary GPU is needed.

To achieve the above requires Linux kernel 3.13 or greater, and RandR 1.4 compatible graphics drivers for both the Intel and Nvidia GPUs. As this rules out the proprietary Nvidia driver because of it's very limited support for RandR 1.4, we are left with only the open-source Nouveau driver as an option, yet it comes with a number of down-sides:

  * It doesn't support dynamic power management, so it consumes more power than it should, running your battery down faster.
  * The lack of dynamic power management also causes it to run hotter than it should, which could theoretically shorten the life of your batteries if unchecked.
  * The lack of dynamic power management also means that it has to be run at a much slower speed than it otherwise could, and for 3D rendering (e.g. games) it [performs at about 1/5th of the speed](http://www.phoronix.com/scan.php?page=article&item=nvidia_2d_openclose&num=1) of the proprietary driver.
  * It is reverse engineered without NVidia's help, so may be buggy.

Although this sounds dire, it need not be too bad for non-gamers provided it can be made to work since, it allows the Intel GPU to be used when mobile, where the Nvidia GPU is only co-opted in at the user's request, and it allows both GPUs to be used when docked, when power usage is of no concern anyway.

#### Life With Optimus Prime

On my laptop (a Lenovo T430) I noticed the following issues when using Optimus Prime (but YMMV):

  * The Nouveau driver struggled with docking and undocking, and to a lesser extent with hibernating and un-hibernating, so that I often had to restart the laptop when it got into a strange state.
  * The Intel GPU struggled to performantly render to displays connected via the Nvidia GPU, and even in the best case scenario (see below) exhibited tearing when windows were moved that was noticeably worse than what you see when using Nvidia Prime.
  * Gnome 3.10+ has a performance regression that exacerbates the performance issues when rendering to Nvidia connected displays, to the point of making it unusable, with visual artifacts being left around the screen, so I was forced to pair Gnome 3.8 with a 3.13+ Linux kernel.
  * The RedHat kernel present in Fedora must contain some additional commits not available within the main-line kernel, since the Nvidia connected displays only perform fast enough to make them truly usable when paired with that kernel.

In short, on my laptop, Optimus Prime is only usable when Fedora 19 is installed (the last version of Fedora with Gnome 3.8) and once the kernel has been allowed to update past what it originally ships with, to 3.13+. Like this, I can use all four displays when docked, and use very little power when undocked. If it wasn't for the temperamental nature of docking/undocking and hibernation, or the need to run Fedora (sorry Fedora) this would be the perfect set-up.

Personally, I've now settled for a mostly flawless experience with Nvidia Prime on Gnome Ubuntu 14.10 (albeit limited to three displays and with worse battery life), but read on in case you'd like to confirm whether Optimus Prime works better on your laptop...


### Single Monitor Usage

Assuming you have both GPUs enabled, and you are using the Nouveau driver, try running this command (_it may well have been run for you automatically_):

~~~bash
xrandr --setprovideroffloadsink nouveau Intel
~~~

After running this command you will continue to use the Intel driver to drive your laptop display, but with the option to make a specific program use the Nvidia driver when you need it. For example, if you run:

~~~bash
DRI_PRIME=0 glxinfo | grep "OpenGL vendor string"
~~~

you will see the following:

~~~bash
OpenGL vendor string: Intel Open Source Technology Center
~~~

Whereas if you run:

~~~bash
DRI_PRIME=1 glxinfo | grep "OpenGL vendor string"
~~~

you will instead see:

~~~bash
OpenGL vendor string: nouveau
~~~

**Note:** If this doesn't work and you are using Ubuntu 14.10, this is because there is a [known regression in Ubuntu 14.10](https://bugs.launchpad.net/ubuntu/+bug/1388647) which prevents it from working.

Using the 'lm-sensors' package, running `sensors` causes me to see something like this:

~~~bash
acpitz-virtual-0
Adapter: Virtual device
temp1:        +47.0°C  (crit = +200.0°C)

thinkpad-isa-0000
Adapter: ISA adapter
fan1:        2492 RPM

coretemp-isa-0000
Adapter: ISA adapter
Physical id 0:  +49.0°C  (high = +87.0°C, crit = +105.0°C)
Core 0:         +46.0°C  (high = +87.0°C, crit = +105.0°C)
Core 1:         +48.0°C  (high = +87.0°C, crit = +105.0°C)

nouveau-pci-0100
Adapter: PCI adapter
temp1:            N/A  (high = +95.0°C, hyst =  +3.0°C)
                       (crit = +105.0°C, hyst =  +5.0°C)
                       (emerg = +135.0°C, hyst =  +5.0°C)
~~~

Notice how the 'coretemp-isa-0000' temperatures are all below 50°C, and how the 'nouveau-pci-0100' temperature is showing as _N/A_, as it isn't even switched on. If I now run the command:

~~~bash
DRI_PRIME=0 glxgears
~~~

the temperatures stay below 50°C, and I get performance output like this:

~~~bash
304 frames in 5.0 seconds = 60.641 FPS
~~~

But if I instead run:

~~~bash
DRI_PRIME=1 glxgears
~~~

I then get performance figures like this:

~~~bash
11209 frames in 5.0 seconds = 2241.788 FPS
~~~

and the temperature keeps climbing while I leave `glxgears` running, for example:

~~~bash
Adapter: Virtual device
temp1:        +58.0°C  (crit = +200.0°C)

thinkpad-isa-0000
Adapter: ISA adapter
fan1:        3640 RPM

coretemp-isa-0000
Adapter: ISA adapter
Physical id 0:  +60.0°C  (high = +87.0°C, crit = +105.0°C)
Core 0:         +58.0°C  (high = +87.0°C, crit = +105.0°C)
Core 1:         +60.0°C  (high = +87.0°C, crit = +105.0°C)

nouveau-pci-0100
Adapter: PCI adapter
temp1:        +58.0°C  (high = +95.0°C, hyst =  +3.0°C)
                       (crit = +105.0°C, hyst =  +5.0°C)
                       (emerg = +135.0°C, hyst =  +5.0°C)
~~~

Fortunately, about 8 seconds after closing the `glxgears` program the 'nouveau-pci-0100' reverts to showing _N/A_, as it's switched itself off again. This ability to [automatically power the GPU down](http://www.phoronix.com/scan.php?page=news_item&px=MTQ0ODM) when unused was added in Linux 3.12, with Nouveau driver support being added in 3.13, which is why that version of the kernel is preferable, in addition to RandR 1.4 kernel and driver support.

So, even though the Nouveau driver doesn't currently support dynamic power management, it will usually be switched off anyway, leaving your battery untouched. When you need it though, it's still there for you.

This will already be more than adequate for most people's needs, but it's good to know that experimental support for [dynamic power management in Nouveau](http://www.phoronix.com/scan.php?page=article&item=nouveau_try_linux316&num=1) landed in Kernel 3.16, and while it's not ready for mainstream use, progress is still being made.


### Multiple Monitor Usage

To allow all four displays to be accessed, try running this command (_it may well have been run for you automatically_):

~~~bash
xrandr --setprovideroutputsource Intel nouveau
xrandr --auto
~~~

Hopefully this just works for you, but for me when using Gnome Ubuntu 14.04 I saw the following issues:

  1. Driving more than 2 displays caused the displays to go screwy at login, so that after each login I had to open the Displays control-panel and re-configure.
  2. Rendering to monitors connected via ports controlled by the Nvidia GPU was unusably slow, with visual artifacts being left around as windows were moved.
  3. Displays connected via Nvidia controlled ports couldn't be made the primary desktop, which for me meant that the primary desktop couldn't be a monitor connected via DVI, but instead had to be the laptop display or a monitor connected via VGA (this is a big issue for Gnome 3 users such as myself).

I was able to work around the first issue by choosing 'System Default' at the next login, instead of 'GNOME'. As described above, I was able to improve the second problem by running Gnome 3.8, but was only able to achieve a truly usable experience with Fedora 19 (Again, YMMV). Finally, I solved the third issues by disabling the 'Workspaces only on primary display' option in Gnome Tweak Tool, so that the primary display became of less importance.


### VGA Switcheroo

If you need to work with a pre Linux 3.13 kernel, you will need to become better acquainted with `vga_switcheroo`, so that you can enable and disable the discrete GPU.

You can check the current status of both GPUs using the command:

~~~bash
sudo cat /sys/kernel/debug/vgaswitcheroo/switch
~~~

For me this generates output like this:

~~~bash
0:IGD:+:Pwr:0000:00:02.0
1:DIS: :DynOff:0000:01:00.0
~~~

but on earlier kernels that don't have dynamic power management, you would instead see something like this:

~~~bash
0:IGD:+:Pwr:0000:00:02.0
1:DIS: :Off:0000:01:00.0
~~~

With such kernels, you can enable the discrete GPU with the command:

~~~bash
sudo echo ON > /sys/kernel/debug/vgaswitcheroo/switch
~~~

and disable it with the command:

~~~bash
sudo echo OFF > /sys/kernel/debug/vgaswitcheroo/switch
~~~

`vga_switcheroo` also has commands (e.g. `IGD` and `DIS`) to switch which GPU will act as the primary GPU, but these commands can only be used after it has become available, yet before the _boot-splash_ (e.g. Plymouth) or the _display-manager_ (e.g. LightDM) have started. This offers almost no opportunity to actually configure it, since even the `rc.local` script (which is run before the _display-manager_ has started) is run while the _boot-splash_ is running.

Additionally, for many, the Nouveau driver doesn't work when configured as the primary GPU, and users instead see a black or frozen screen when attempting to configure it this way. If you'd still like to at least see if your laptop works when you make Nvidia the primary GPU, you'll find [these instructions](http://askubuntu.com/questions/87489/vgaswitcheroo-not-selecting-discrete-card/97253#97253) really helpful.


## Conclusion

If you've read this far then I hope you've found all of this information useful, even if you still didn't manage to achieve the multi-monitor set-up you've been looking for. Although we seem to have gone backwards in some respects (the [Gnome 3.10 regression](https://bugzilla.gnome.org/show_bug.cgi?id=739865) and the [Ubuntu 14.10 Optimus regression](https://bugs.launchpad.net/ubuntu/+bug/1388647)), there have been more positive steps forward, and things look set to keep getting better.

After lots of searching, I've finally found a workable solution that works for me, and hopefully you're able to find the same.
