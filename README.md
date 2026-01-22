# Debian Xfce MPV Kiosk Media Player

This project was born out of having a parent with dementia in professional memory care. It could be utilized by any person unable to utilize a television remote.

Steps are documented to recreate a low(ish) cost, low power media player that requires no physical interaction after being powered on.


## Purpose

The system:

- Auto-logs in on boot
- Fullscreen plays random videos from a folder
- Outputs audio over HDMI
- Requires no keyboard or mouse interaction
- Survives HDMI hotplug (TV off/on)
- Keeps MPV **always in the foreground**

Designed for **care / memory facilities**, or other unattended installs.

## Hardware
- GMKTec G3 Plus mini PC (N150 processor, 16GB RAM, 512GB SSD)

This PC is admittedly overkill for its purpose. The setup could be tweaked to run on a much lower cost system for example
- Rpi Zero 2 + External USB flash

## Base System
- OS: Debian trixie
- Desktop environment: Xfce
- Display manager: LightDM
- Media player: mpv
- GPU: Intel iGPU (VAAPI)
- Audio: HDMI → TV

During Debian installer:

- Select **Xfce** desktop
- Set a **root password**
- Create an **admin user**
- SSH (optional for setup and media transfer)


## Users

### media
- Auto-login user
- No sudo access
- Runs MPV only

### admin
- Maintenance user
- Has sudo

# Post Debian Install

## Packages

Install:

```bash
sudo apt install \
  mpv \
  intel-media-va-driver \
  mesa-utils \
  alsa-utils \
  wmctrl
```


Remove power manager
```
sudo apt purge xfce4-power-manager
```

# create media user

```
adduser media
# create a password, use default values for everything else

mkdir /home/media/Videos
chown media:media /home/media/Videos
```

## Modify LightDM to auto login
```
# /etc/lightdm/lightdm.conf

[Seat:*]
autologin-user=media
autologin-user-timeout=0
user-session=xfce

```

## Disable sleep
```
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

# Services
Create the following files.  After creation set the user:group to `media:media` if they were created with the admin or root account


### Disable blanking
```
# /home/media/.config/autostart/disable-blanking.desktop

[Desktop Entry]
Type=Application
Name=Disable Screen Blanking
Exec=sh -c "xset s off; xset -dpms; xset s noblank"
Terminal=false
X-GNOME-Autostart-enabled=true

```

### MPV Video player
```
# /home/media/bin/play-videos.sh
# Note: must be executable and owned by media

#!/bin/bash

SINK="alsa_output.pci-0000_00_1f.3.hdmi-stereo"
MEDIA="/home/media/videos"

# Wait for Pulse/PipeWire and HDMI sink
for i in {1..30}; do
  pactl info >/dev/null 2>&1 && \
  pactl list short sinks | grep -q "$SINK" && break
  sleep 1
done

# Force HDMI as default
pactl set-default-sink "$SINK" || true

# Move any existing streams
for id in $(pactl list short sink-inputs | awk '{print $1}'); do
  pactl move-sink-input "$id" "$SINK" || true
done

sleep 0.5

exec /usr/bin/mpv \
  --fullscreen \
  --ontop \
  --no-border \
  --no-osd-bar \
  --no-input-default-bindings \
  --loop-playlist=inf \
  --shuffle \
  --hwdec=vaapi \
  --ao=pulse \
  "$MEDIA"
```

```
# /home/media/.config/autostart/mpv.desktop
[Desktop Entry]
Type=Application
Name=Media Player
Exec=/home/media/bin/play-videos.sh
Terminal=false
X-GNOME-Autostart-enabled=true
```

### Raise MPV to foreground

```
# /home/media/bin/raise-mpv.sh  must be executable and owned by media
#!/bin/sh
while true; do
  wmctrl -R mpv 2>/dev/null
  sleep 2
done
```

```
# /home/media/.config/autostart/raise-mpv.desktop
[Desktop Entry]
Type=Application
Name=Raise MPV
Exec=/home/media/bin/raise-mpv.sh
Terminal=false
X-GNOME-Autostart-enabled=true
```


# copy media files
Use XFCE desktop and a USB drive or SCP to transfer media files to `/home/media/Videos`.  Again, these files should be owned by `media:media`

```
sudo chown -R media:media /home/media/Videos/
```

# Usage:

Connect the power cable to your mini pc, connect the HDMI cable to your display, and turn on the power.

Playback can be stopped but connected a USB mouse and exiting the player.  This allows access for maintenance on the device (e.g. adding or removing media files)

# Debugging

## Audio debugging

Make sure you have the correct Alsa card and your speakers actually work

```
# list available devices
aplay -l

# find which one works for you by running speaker-test
# this will also verify that your connected speakers are working and un muted
speaker-test -D hdmi:CARD=PCH,DEV=0 -c 2 -t wav
```

To find the appropriate sink value for the `SINK` variable in `play-videos.sh`

```
$ pactl list short sinks

# copy the alsa_output line into the SINK variable

0	alsa_output.usb-CSCTEK_USB_Audio_and_HID_A34004801402-00.analog-stereo	module-alsa-card.c	s16le 2ch 48000Hz	SUSPENDED
1	alsa_output.pci-0000_00_1f.3.hdmi-stereo	module-alsa-card.c	s16le 2ch 44100Hz	SUSPENDED
```
For the particular mini pc that was used, `alsa_output.pci-0000_00_1f.3.hdmi-stereo` was the correct `SINK`

# Important Notes

There is minimal security in this setup.  SSH access may be enabled depending on how you installed debian and whether or not you have a network connection.

Virtual Terminals are enabled.

Any person on site with a USB mouse/keyboard can stop playback and access the desktop as the `media` user, though sudo access is not enabled.

Use at your own risk

# Future Implementations

There is no plan to actively maintain this document.  Some enterprising dev may want to think about

 - Connected cloud drive where a caregiver can manage media files remotely
 - Hardened security
 - `.deb` installer package
 - Implementation without a desktop environment; this could be run on a bare xorg-server install
 - Auto On feature.  This would likely be a bios setting to turn the system on whenever power is applied
 - Porting to RPI Zero 2 for lowest cost/power usage