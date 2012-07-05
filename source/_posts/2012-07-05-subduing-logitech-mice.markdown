---
layout: post
title: "Subduing Logitech mice"
date: 2012-07-05 13:51
comments: true
categories: [Linux, tinkering]
---

Logitech support for their products on Linux platform may be discouraging for casual users. Recently I faced this problem myself. In Windows environment I used to max out my lovely G3 mouses dpi through vendored Logitech software, turn off acceleration and set sensitivity low. That's what I got pretty strong habit for. I also switched off that nasty prone to random triggering __quickly change dpi__ button and was quite happy with that setup.

On my freshly installed Ubuntu 12.04 though nothing seemed to work obviously. However, I managed to achieve almost identical behaviour, thus going to share my findings.
<!--more-->

## Tackling some mice
First of all, I needed to manage dpi and buttons behaviour in an apt manner. Fortunately, some valiant reverse-engineering folks produced handy little python script for that matter. [G5Mouse](http://code.google.com/p/g5mouse/) works with hiddev driver directly to read and set device flags. Following command let me achieve required dpi and turn off that nifty button.

{% codeblock lang:bash %}
g5mouse.py -d 2000 -n /dev/usb/hiddev0
{% endcodeblock %}

Along with that change mouse movement becomes hectic. The way to diminish sensitivity and acceleration rates of cursor is through __xinput__. Firstly, we need to determine _id_ of the device. Running `xinput` should produce something like the following:

```bash
$ xinput
⎡ Virtual core pointer                      id=2[master pointer    (3)]
⎜   ↳ Virtual core XTEST pointer                id=4[slave  p ointer  (2)]
⎜   ↳ Logitech USB Gaming Mouse                 id=10[  slave  pointer  (2)]
⎣ Virtual core keyboard                     id=3  [master keyboard (2)]
    ↳ Virtual core XTEST keyboard               id=5[slave  keyboard (3)]
    ↳ Power Button                                id=6[slave  keyboard (3)]
    ↳ Power Button                              id=7[slave  keyboard (3)]
    ↳ DATACOMP SteelS쀁̄Љ̒DA  TA              id=8[slave  keyboard (3)]
    ↳ DATACOMP SteelS쀁̄Ð̒DATA
```

Here we can see _Logitech USB Gaming Mouse_ going along with `id=10`. Great, that's what we need for the next few commands:

```bash
$ xinput set-prop 10 "Device Accel Velocity Scaling" 0.000001
$ xinput set-prop 10 "Device Accel Adaptive Deceleration" 1.5
$ xinput set-prop 10 "Device Accel Constant Deceleration" 1.5
```
All those values are experimental and personal preference.

Am I supposed to pound these commands every single time I boot or resume from sleep|hibernation? In Linux? No way.

## Automating with Upstart

[Upstart](http://upstart.ubuntu.com/cookbook/) is a robust event-driven init daemon adopted in Ubuntu for a long time now. Its event-based nature greatly facilitates the ease of my automation setup.

Here I moved `g5mouse.py` script into `/usr/bin` directory thus I can call it directly:

```bash
$ sudo cp g5mouse.py /usr/bin/g5mouse
$ sudo chmod 755 /usr/bin/g5mouse
```

Then I created `g5mouse.conf` file in `/etc/init` directory containing this:

```bash /etc/init/g5mouse.conf
start on login-session-start or resume
task

script
  g5mouse -d 2000 -n /dev/usb/hiddev0
end script
```
and ran `sudo initctl reload-configuration`.

Here's the thing. This task starts whenever events `login-session-start` or `resume` are emitted. Former is a stock system event occuring once X display manager is started. We ain't gonna use our mouse without graphical environment, right? Latter is a custom event, emitted _manually_ by command `initctl emit --no-wait resume`. Why do we need it? Well, it was discovered, that driver settings don't persist on sleep/resume, thus I needed some utility to retain them. And here [pm-utils](https://wiki.archlinux.org/index.php/Pm-utils) kicks in.

Scripts, located in `/etc/pm/sleep.d`, are executed whenever system in going down in suspend or resumes its operation. Upon _resume from RAM_ scripts receive `resume` as the first parameter. Ergo we can place there something like this:

```bash /etc/pm/sleep.d/upstart-event-emitter
#!/bin/bash
case $1 in
  resume|thaw)
    initctl emit --no-wait resume
    ;;
esac
```
and `chmod +x` it.

Finally, we need to automatically apply user-specific __xinput__ values upon each log in. To do this, we create the following files:

```bash ~/bin/xinput.sh
#!/bin/bash
xinput set-prop 10 "Device Accel Velocity Scaling" 0.000001
xinput set-prop 10 "Device Accel Adaptive Deceleration" 1.5
xinput set-prop 10 "Device Accel Constant Deceleration" 1.5
```

```bash ~/.config/autostart/xinput.desktop
[Desktop Entry]
Type=Application
Exec=~/bin/xinput.sh
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
Name[en]=mouse xinput
Name=mouse xinput
Comment[en]=
Comment=
```

and `chmod +x ~/bin/xinput.sh`.

Now we're all set and ready to ride some mice.
