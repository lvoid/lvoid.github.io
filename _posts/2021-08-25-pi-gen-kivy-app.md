---
layout: post
toc: true
title: "Using Pi-Gen for Touch Screen Kivy Application"
categories: programming
tags: [raspberry pi, pi-gen, kivy]
---

## [Another Interesting Project at Work](#another-interesting-project-at-work)

I am in the process of assisting the development of a touch-screen Kivy/KivyMD application on the Raspberry Pi.  The touch screen that is being utilized is the [UCTRONICS 5-inch](https://www.uctronics.com/display/uctronics-5-inch-touch-screen-for-raspberry-pi.html), which comes with a driver to enable a resolution of 800x480.

This touch screen tool will hopefully see a day when it is largely produced on some scale, which means that we need some way to easily replicate a Raspberry Pi OS image that has the software preinstalled, along with all configurations, tweaks, dependencies, and additional tools.  In other words, we need to create that single image that can be burnt onto as many SD cards as we need it to be.

## [Cracking Pi-Gen](#cracking-pi-gen)

Pi-gen is an official Raspberry Pi tool used to create Raspberry Pi OS images. You can download the Github repository [here](https://github.com/RPi-Distro/pi-gen).

By default, pi-gen creates the base Raspberry Pi OS without any additional modifications, including a default keyboard that is UK-style etc.  To create my own custom image for mass production, however, I had to ensure that the image would contain several additional things:

- Python 3.6+
- Kivy
- KivyMD
- TinyDB v.3.15.2
- OpenCV
- pyzbar
- Basic pentesting tools

The touch screen application also utilizes a camera mounted on the Raspberry Pi to take pictures of QR codes which are decoded by pulling information from an internal database, hence the need for OpenCV, pyzbar, and TinyDB.

That takes care of the software dependencies.  But that's not all.  Other things I needed included:
- Our application
- Additional fonts
- A common .crt file to connect to an external daemon
- A .desktop entry linked to the Python application
- A custom Kivy configuration file placed in a specific location
- The UCTRONICS touch screen driver preinstalled onto the OS

It seems like a lot of stuff, and I actually had no idea where to make modifications in the scripts until I looked at how pi-gen was operating under the hood.

The tool has six build stages, ranging from 0 to 5. It begins with building the fundamentals of the operating system and ens with nice-to-have packages such as a web browser etc.  On your host operating system, it utilizes Linux tools such as `apt` to build an OS from the ground up.  In each stage, the `.sh` scripts that are run typically reference a file that contains various lists of packages.

My objective would be modifying both the shell scripts to perform certain tasks, as well as appending to the lists of software applications and dependencies to add components unique to our own application.

## [A Docker Preference](#a-docker-preference)

Just a side note.

During the first few builds, I have having a horrible time attempting to build the application on my host operating system.  I encountered a number of cryptic and inexplicable errors that would stop my build.  I scoured the internet for information about these errors, only to find that most questions are simply opened up as "Issues" on the Github repository.  I asked a question about something that was going on with my build once, and the developer who replied did not know what was going on.

There was an alternative I had not tried though - building with Docker.  The tool that always comes in strong.  `sudo ./build-docker.sh` was my savior, and all of the successful `.img`'s I produced came from using it.

## [Pi-Gen Configuration File](#pi-gen-configuration-file)

In their Github repository, pi-gen specifies that you can create your own configuration file for some basic parameters on the OS.  This is what mine ended up looking like.

```
IMG_NAME=pentest-app
USE_QCOW2=0
RELEASE=buster
DEPLOY_ZIP=1
WPA_COUNTRY=US
TARGET_HOSTNAME=pi
KEYBOARD_KEYMAP=us
KEYBOARD_LAYOUT="English (US)"
TIMEZONE_DEFAULT="America/New_York"
ENABLE_SSH=1
CLEAN=1
```

I changed the OS to use SSH by default, and also assigned it the correct timezone and keyboard layout.

I named this file `custom-config` and put it inside of the root directory of pi-gen.

## [Kivy Configuration File](#kivy-configuration-file)

I discovered during this process that there are two ways of making a portrait-mode touch screen application work: editing Kivy's configuration file and leaving the operating system parameters alone, and then the opposite.

I opted for playing the configuration file of Kivy.

We are going to edit the Kivy `config.ini` to allow portrait mode application operation and touch screen settings for this rotated application.

The Kivy configuration file is located at `~/.kivy/config.ini` if you have a local installation on your host machine.

Edit this file and scroll down to `[graphics]`.  You will need to modify three parameters here: the `height`, `width`, and `rotation`.  Give them the following values.

```bash
height=800
rotation=90
width=480
```

By default, the application I am working with is meant to run fullscreen.  So set `fullscreen=1` under this section as well.

Now you can scroll down to the section titled `[input]` and modify it to look like the following in order to sync the touchscreen with the rotated application.

```bash
mouse=mouse
mtdev_%(name)s = probesysfs,provider=mtdev
%(name)s = probesysfs,provider=hidinput,param=rotation=90
```

Keep this `config.ini` file handy for `Stage 4`.

## [Stage 1](#stage-1)

In this stage, we will be modifying some boot parameters and installing the touch screen driver to ensure that the image works well with the UCTRONICS screen.

1. Two lines need to be commented out in the `pi-gen/stage1/00-boot-files/files/config.txt` file.  They are `dtoverlay=vc4-fkms-v3d` and `max_framebuffers=2`.
2. We need to get rid of the black border around the display.  To do this, uncomment `disable_overscan=1` in `pi-gen/stage1/00-boot-files/files/config.txt`.
3. We need to add the UCTORNICS driver script, called [hdmi_480x800_cfg.sh](https://github.com/UCTRONICS/UCTRONICS_HDMI_CTS#add-480x800-resolution) to the project in `pi-gen/stage1/00-boot-files` and then run it by appending `./hdmi_480x800_cfg.sh` to the end of the script `/pi-gen/stage1/00-boot-files/00-run.sh`.
4. Finally, we need to edit the driver shell script `hdmi_480x800_cfg.sh` to have paths relative to the Raspberry Pi OS image being built, as the rest of the `pi-gen` scripts have.

```bash
#!/bin/sh

# Setup GPIO for Arducam Multi Adapter Board Pi zero
# Disable the camera_led
echo "Begin to configure the 480x800 resolution..."
awk 'BEGIN{ count=0 }       \
{                           \
    if($1 == "hdmi_force_hotplug=1"){       \
        count++;            \
    }                       \
}END{                       \
    if(count <= 0){         \
	system("sudo sh -c '\''echo hdmi_force_hotplug=1 >> ${ROOTFS_DIR}/boot/config.txt'\''"); \
        system("sudo sh -c '\''echo max_usb_current=1 >> ${ROOTFS_DIR}/boot/config.txt'\''"); \
    	system("sudo sh -c '\''echo hdmi_group=2 >> ${ROOTFS_DIR}/boot/config.txt'\''"); \
	system("sudo sh -c '\''echo hdmi_mode=87 >> ${ROOTFS_DIR}/boot/config.txt'\''"); \
	system("sudo sh -c '\''echo hdmi_cvt 800 480 60 6 0 0 0 >> ${ROOTFS_DIR}/boot/config.txt'\''"); \
	system("sudo sh -c '\''echo hdmi_drive=1 >> ${ROOTFS_DIR}/boot/config.txt'\''"); \
	}                       \
}' ${ROOTFS_DIR}/boot/config.txt 
echo "480x800 resolution configure OK."
```

## [Stage 2](#stage-2)

In this stage, we are adding the dependency packages that are requires for Kivy and KivyMD to function correctly.

1. Append these packages to `pi-gen/stage2/01-sys-tweaks/00-packages` so the file looks like the following.

```
python3-setuptools
git-core
python3-dev
pkg-config
libgl1-mesa-dev
libgles2-mesa-dev
libgstreamer1.0-dev
gstreamer1.0-plugins-{bad,base,good,ugly}
gstreamer1.0-{omx,alsa}
libmtdev-dev
xclip
xsel
libjpeg-dev
libsdl2-dev
libsdl2-image-dev
libsdl2-mixer-dev
libsdl2-ttf-dev
```

## [Stage 3](#stage-3)

Kivy and KivyMD are not the only software installations that require dependencies.  OpenCV and pyzbar also require a few dependencies.  Let's add them here.

1. Edit `pi-gen/stage3/00-install-packages/01-run.sh` to include the following installations after `www-browser` is installed.

```bash
#!/bin/bash -e

on_chroot << EOF
update-alternatives --install /usr/bin/x-www-browser \
  x-www-browser /usr/bin/chromium-browser 86
update-alternatives --install /usr/bin/gnome-www-browser \
  gnome-www-browser /usr/bin/chromium-browser 86
apt-get update -y
apt-get install libzbar0=0.22-1 --allow-downgrades -y
apt-get install python-zbar --allow-downgrades -y
apt-get install libhdf5-dev --allow-downgrades -y
apt-get install libhdf5-serial-dev --allow-downgrades -y
apt-get install libatlas-base-dev --allow-downgrades -y
apt-get install libjasper-dev --allow-downgrades -y
apt-get install libqt5gui5 --allow-downgrades -y
EOF
```

## [Stage 4](#stage-4)

This particular project requires a few penetration testing, network, and anti-virus tools, as it is serving as a portable ICS/SCADA scanning tool.  This seems to be a fitting stage to add these "extras" to.

1. Edit `pi-gen/stage4/00-install-packages/00-packages`, appending the following to the end of the list.

```bash
lynis
nmap
wireshark
clamav
clamav-daemon
```

2. Inside of the `pi-gen/stage4/02-extras` directory, create a new folder called `files/` and place the following items inside of it.

- The Kivy `config.ini` that we went over above
- The project files, let's say `pentest-app`

3. We are going to do a number of things now which you can read about in the comments of the bash script.  Edit the bash script `pi-gen/stage4/02-extras/00-run.sh` to look like the following.

```bash
# Pip installs for Kivy & dependencies
on_chroot << EOF
python3 -m pip install kivy[base] --no-binary kivy
python3 -m pip install -Iv kivymd==0.104.1
python3 -m pip install -Iv tinydb==3.15.2
python3 -m pip install opencv-python
python3 -m pip install pyzbar
python3 -m pip install -U numpy
EOF

# Create folder for pentest-app files
mkdir ${ROOTFS_DIR}/usr/share/pentest-app

# Copy in the Kivy configuration file to local user space
mkdir ${ROOTFS_DIR}/home/${FIRST_USER_NAME}/.kivy
cp -p files/config.ini "${ROOTFS_DIR}/home/${FIRST_USER_NAME}/.kivy"

# Copy in the Kivy configuration file to root space
cp -p files/config.ini "${ROOTFS_DIR}/root/.kivy/config.ini"

# Copy in the pentest-app project files to the home directory
cp -rp files/kivy-app "${ROOTFS_DIR}/usr/share/pentest-app"

# Make main file executable
chmod u+x "${ROOTFS_DIR}/usr/share/pentest-app/main.py"

# Desktop not created at this stage, add /Desktop so we can make a .desktop file
mkdir ${ROOTFS_DIR}/home/${FIRST_USER_NAME}/Desktop

# Add desktop entry for pentest-app
content="[Desktop Entry]
Name=pentest-app
Version=v0.1
Icon=/usr/share/pentest-app/src/icons/circuit.png
Exec=sudo python3 /usr/share/pentest-app/main.py
Terminal=true
Type=Application
"

# Place desktop entry into .desktop file
echo "$content" | tee ${ROOTFS_DIR}/home/${FIRST_USER_NAME}/Desktop/pentest-app.desktop
chmod u+x ${ROOTFS_DIR}/home/${FIRST_USER_NAME}/Desktop/pentest-app.desktop

# Copy in the pentest-app server certificate files to /usr/share/pentest-app
cp -p files/client.crt "${ROOTFS_DIR}/usr/share/pentest-app"

# Install fonts into the system
install -m 644 files/kivy-app/fonts/Calibri.ttf "${ROOTFS_DIR}/usr/share/fonts"
install -m 644 files/kivy-app/fonts/Corbel.ttf "${ROOTFS_DIR}/usr/share/fonts"

# Enable SSH by default
on_chroot << EOF
systemctl enable ssh
systemctl start ssh
EOF
```

Modifying the script to contain the above will install Kivy and other packages we need via `pip`.  It also places the `config.ini` file in the proper location, copies in the project files, creates a desktop entry with an icon which executes the application, and a few other things like install fonts not present and activate `ssh`.

Note that the Kivy `config.ini` file is in `/root`.  I placed it here because the project files are in `/usr/share`.  If the Kivy configuration file is in `~/.kivy`, the application in `/usr/share` will not be able to read the configuration file.

## [Stage 5](#stage-5)

We are skipping this stage as we do not need any of the extraneous installations included here. To skip it on your machine, place two empty files into the `stage5/` folder called `SKIP` and `SKIP_IMAGES`.

## [Building](#building)

I ran `sudo ./build_docker.sh -c custom-config` to perform a Docker build with the configuration file mentioned previously.

The build took quite some time - probably a little over an hour.  The resulting zipped `.img` can be found in the `deploy/` directory.
