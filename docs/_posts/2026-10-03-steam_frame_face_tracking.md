---
layout: post
title: 'Steam Frame face tracking'
comments: true
---

![Face tracking working in VRChat!](/assets/face_vr.png)

# Hardware

What you'll need:
 - USB webcam (I used the unobtanium [Vive Facial Tracker](https://docs.vrcft.io/docs/hardware/addons/vive/face-tracker), but any should work)
 - A way to mount it (many 3d-printable mounts are online, I used [this one](https://www.printables.com/model/1859624-steam-frame-vive-tracker-mount-front-with-babble)
 - USB hub with PD input, if you want to charge or use a power bank while playing (I used [this one](https://www.anker.com/products/a8365)). Currently mounted with zip-ties, will eventually figure out a better way :')

![Picture of my hardware setup](/assets/frame_hardware_1.png)

# Software

## Summary

Linux is Linux, so things should just work! BUT! We have a few holes to plug. The root filesystem is read-only, and there are very few drivers shipped by default. Also, Baballonia needs to be built from source, to get a proper arm64 build with all the permissions.

## Changes to `~/.bashrc`

```bash
# If not running interactively, don't do anything
[[ $- != *i* ]] && return

#---- History things, not required ----

HISTFILE=/home/$USER/.bash_history_bak/current

# don't put duplicate lines in the history. See bash(1) for more options
# force ignoredups and ignorespace
HISTCONTROL=ignoreboth

# append to the history file, don't overwrite it
shopt -s histappend

HISTSIZE=99999999
HISTFILESIZE=99999999
HISTTIMEFORMAT="%y/%m/%d %T "

# for setting history length see HISTSIZE and HISTFILESIZE in bash(1)
PROMPT_COMMAND='let ret=$?; history -a; echo -ne "\033]0;terminal\a"'
#PROMPT_COMMAND='let ret=$?; history -a; history -n'

#---- Needed for distrobox

export PATH="$HOME/.local/bin:$PATH"
# edit this alias if your setup is different
alias d='distrobox enter --root rarch'

#---- Needed for dotnet/Baballonia

export DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1
export DOTNET_ROOT="$HOME/.dotnet"
export PATH="$PATH:$DOTNET_ROOT:$DOTNET_ROOT/tools"

#---- Where I put the Baballonia script later

export PATH="$HOME/bin:$PATH"

```

Don't forget to apply the changes after you make them with `. ~/.bashrc`

## A Debian

Create a debian so we can install things; can use an Arch too, but you'll be on your own with packages there

```bash
curl -s https://raw.githubusercontent.com/89luca89/distrobox/main/install | sh -s -- --prefix ~/.local
distrobox create --image docker.io/library/debian:sid --name rarch --root
distrobox enter --root rarch


sudo apt update
# some are needed for Baballonia
sudo apt install vim git patchelf libudev-dev v4l-utils libgstreamer1.0-0 libgstreamer-plugins-base1.0-0
sudo apt build-dep linux
```

## Building the kernel module for the uvc camera
Valve, plz ship the kernel sources and/or the common modules, this took waaaay too long to figure out

```
git clone --depth 1 https://github.com/torvalds/linux.git -b v6.18 linux-6.18.0
cd linux-6.18.0

# run this if retrying
make distclean
```

Edit the makefile to make the kernel name match:

```diff
--- EXTRAVERSION =
+++ EXTRAVERSION = -ge66bc2ca6f8c
```
Adjust the suffix if the new kernel was relelased.
Check that the versions match:
```
make kernelversion
uname -r
```

In `tools/lib/bpf/Makefile`, around line 87, after all `override CFLAGS`, add another one:
```
# override CFLAGS += -Wno-error=discarded-qualifiers
```
This was needed to fix a build error due to a newer GCC version for me.

Now get the valve kernel headers:

```bash
headers_url=$(sudo pacman -Spdd linux-618-deckard-headers)
echo "$headers_url"
wget "$headers_url" ../headers.pkg.tar.zst
```

Extract somewhere

```bash
mkdir ../headers/
tar xf ../headers.pkg.tar.zst ../headers/
```

Ready to build the kernel modules.

```bash
cp ../headers/usr/lib/modules/$(uname -r)/build/Module.symvers ./
make menuconfig
```
Save and exit
```bash
zcat /proc/config.gz > .config
make menuconfig
```

Device Drivers -> Multimedia support -> Media drivers -> Media USB Adapters (toggle) -> USB Video Class (toggle to M)
General setup -> Automatically append version informatio nto the version string -> toggle off

Exit -> save to .config

```bash
make scripts
make prepare
make modules_prepare

make -C . M=drivers/media/common
cat drivers/media/common/Module.symvers >> Module.symvers # don't ask
make -C . M=drivers/media/usb/uvc
```

If everything went well, modules will be compatible. If not, you'll get errors from `insmod` below.

```bash
sudo insmod ./drivers/media/common/videobuf2/videobuf2-vmalloc.ko
sudo insmod ./drivers/media/common/uvc.ko
sudo insmod ./drivers/media/usb/uvc/uvcvideo.ko
```

dmesg will show the camera being found at this point, hopefully. Make sure it's plugged in! :D



## Baballonia


Get dotnet:

```bash
cd

wget https://builds.dotnet.microsoft.com/dotnet/Sdk/10.0.401/dotnet-sdk-10.0.401-linux-arm64.tar.gz

mkdir ~/.dotnet
tar zxf dotnet-sdk-10.0.401-linux-arm64.tar.gz --no-same-owner -C ~/.dotnet
```

Now, clone and build Baballonia:

```bash
git clone https://github.com/Project-Babble/Baballonia.git
cd Baballonia
git submodule update --init --recursive
bash download_dependencies.sh
cd src/Baballonia.Desktop/
dotnet publish -r linux-arm64 -c Release --self-contained -f net10.0
```

Make a launch script for baballonia: `~/bin/baballonia`

```bash
#!/bin/bash

# TODO: auto-detect the camera and update the baballonia settings
# TODO: auto-detect steam streaming host and update the IP in settings?

"$HOME/Baballonia/src/Baballonia.Desktop/bin/Release/net10.0/linux-arm64/Baballonia.Desktop"
```

## Baballonia configuration

I had to run Baballonia in Desktop mode, running it as an app directly doesn't work for some reason (UI breaks).

On the main screen, only edit the Face camera, eye tracking is sent by Steam itself. If you click the dropdown, you will see `video99` as the steam virtual camera, and hopefully whatever other USB camera you have plugged in. BUT! Do not choose it in the dropdown - type `/dev/videoXX` in the field directly. Once you do, if you are using the Vive Facial Tracker, select that option in the dropdown below, otherwise keep it as V4L2 device. Then, click `Start Camera`, and, if everything went well, you will see the camera feed of your face!

Only thing left is to change where the data is sent to: click the gear on the left of the window to go to settings, open the OSC section, and change the IP address for the PC you will be steaming from.

On that PC, you will need to have VRCFaceTracking running, with 2 modules - SteamLink VRCFT module for the eye tracking, and Project Babble Module for the face tracking. Once you install those modules and run it once, close it, and edit the config file (on linux it's in `~/.config/VRCFaceTracking/CustomLibs/{long-hex-uuid-starting-with-3-probably}`/BabbleConfig.json: change `"Host": "127.0.0.1"` to the IP of the computer, so it can actually receive the Babble data from the headset. If all went well, the face tracking should work now! :D
