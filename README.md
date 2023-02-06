# Raspberry PI | Auto CD Ripping

## 1. Ripping

- `abcde`

From: http://sfriederichs.github.io/2020/11/16/aumatically-rip-cds-mp3-a-raspberry-pi.html

```
$ sudo apt install abcde lame eject id3 id3v2 eyed3 mkcue normalize-audio mp3gain libdata-dump-perl
```

### Update config

```
$ cp abcde.conf /etc/abcde.conf
```

## 2. Storing

- `syncthing`

From: https://pimylifeup.com/raspberry-pi-syncthing/

```
$ sudo apt install apt-transport-https gpg curl
$ curl -s https://syncthing.net/release-key.txt | gpg --dearmor | sudo tee /usr/share/keyrings/syncthing-archive-keyring.gpg >/dev/null
$ echo "deb [signed-by=/usr/share/keyrings/syncthing-archive-keyring.gpg] https://apt.syncthing.net/ syncthing stable" | sudo tee /etc/apt/sources.list.d/syncthing.list
$ sudo apt update
$ sudo apt install syncthing
```

### Modify `~/.config/syncthing/config.xml` to open up Web UI to all interfaces

```
$ vi ~/.config/syncthing/config.xml
```

`<address>127.0.0.1:8384</address>` -> `<address>0.0.0.0:8384</address>`

### Make syncthing a service

```
$ sudo vi /lib/systemd/system/syncthing.service
```

Add

```
[Unit]
Description=Syncthing
Documentation=man:syncthing(1)
After=network.target

[Service]
User=alex
ExecStart=/usr/bin/syncthing -no-browser -no-restart -logflags=0
Restart=on-failure
RestartSec=5
SuccessExitStatus=3 4
RestartForceExitStatus=3 4

# Hardening
ProtectSystem=full
PrivateTmp=true
SystemCallArchitectures=native
MemoryDenyWriteExecute=true
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

Turn on service

```
$ sudo systemctl enable syncthing
$ sudo systemctl start syncthing
$ sudo systemctl status syncthing
```

### Control via web

http://<PI IPADDRESS>:8384/

## 3. Autostart

Setup abcde service

Copy [cd_rip.sh](cd_rip.sh) to `/usr/local/bin/cd_rip.sh`

```
$ sudo cp cd_rip.sh /usr/local/bin/
```

Copy [abcde.service](abcde.service) to `/lib/systemd/system/abcde.service`

```
$ sudo cp abcde.service /lib/systemd/system/abcde.service
```

Enable service

```
$ sudo systemctl daemon-reload
$ sudo systemctl enable abcde.service
```

### Setup service auto trigger

Create udev rule to start `abcde` when CD is inserted

```
$ sudo vi /etc/udev/rules.d/99-cd-ripping.rules
```

File: [99-cd-ripping.rules](99-cd-ripping.rules)

Reload udev rules

```
$ sudo udevadm control --reload-rules && sudo udevadm trigger
```

# Apple Superdrive (Optional)

Add `/etc/udev/rules.d/90-mac-superdrive.rules`

```
# Initialise Apple SuperDrive
ACTION=="add", ATTRS{idProduct}=="1500", ATTRS{idVendor}=="05ac", DRIVERS=="usb", RUN+="/usr/bin/sg_raw %r/sr%n EA 00 00 00 00 00 01"
```

# Prior art

- [aac/raspberrypi-cd-ripper](https://github.com/aac/raspberrypi-cd-ripper)

# Raspberry PI | Auto DVD Ripping

## Get MakeMKV

From: https://forum.makemkv.com/forum/viewtopic.php?f=3&t=224

```
$ wget https://www.makemkv.com/download/makemkv-oss-1.17.1.tar.gz
$ wget https://www.makemkv.com/download/makemkv-bin-1.17.1.tar.gz
```

## Build MakeMKV

Build tools

```
$ sudo apt install build-essential pkg-config libc6-dev libssl-dev libexpat1-dev libavcodec-dev libgl1-mesa-dev qtbase5-dev zlib1g-dev
```

"source" package:

```
$ cd makemkv-oss-1.17.3
$ ./configure
$ make
$ sudo make install
```

"makemkv-bin" package:

```
$ cd ../makemkv-bin-1.17.3
$ make
$ sudo make install
```

## Add `libavcodec` (`ffmpeg`) to the mix

```
$ wget https://ffmpeg.org/releases/ffmpeg-snapshot.tar.bz2
$ $ tar -xvjf ffmpeg-snapshot.tar.bz2
$ cd ffmpeg
$ ./configure --prefix=/tmp/ffmpeg --enable-static --disable-shared --enable-pic
$ make install
```

configure and build makemkv-oss:

```
$ cd ../makemkv-oss-1.17.3
$ PKG_CONFIG_PATH=/tmp/ffmpeg/lib/pkgconfig ./configure
$ make
$ sudo make install
```

clean up ffmpeg temp files:

```
rm -rf /tmp/ffmpeg
```

## Run

```
$ makemkvcon list
```

# Prior art

- [jlesage/docker-makemkv](https://github.com/jlesage/docker-makemkv/blob/master/Dockerfile)
- [automatic-ripping-machine](https://github.com/automatic-ripping-machine/automatic-ripping-machine)
- [RPi + MakeMKV](https://forum.makemkv.com/forum/viewtopic.php?t=29688)
- [udev rules](https://github.com/Robpol86/makemkv)
- [no gui](https://forums.freebsd.org/threads/install-and-use-makemkv-without-gui-on-freebsd.70614/)


## Temp / Alternative

NAS

### makemkv

```
$ sudo chmod go+rw /dev/sg5
$ docker run -d \
--name=makemkv \
-p 5800:5800 \
-v /volume1/docker/makemkv/config:/config \
-v /volume1/docker/makemkv/storage:/storage \
-v /volume1/Downloads/_Rips:/output \
--device /dev/sg5 \
-e TZ=America/Los_Angeles \
-e USER_ID=`id -u` \
-e GROUP_ID=`id -g` \
-e UMASK=000 \
-e AUTO_DISC_RIPPER=1 \
-e AUTO_DISC_RIPPER_MIN_TITLE_LENGTH=300 \
jlesage/makemkv
```

PS. Check if `/output` folder inside container has write permissions.

### handbrake

```
$ docker run -d \
--name=handbrake \
-p 5900:5800 \
-v /volume1/docker/handbrake/config:/config \
-v /volume1/docker/handbrake/storage:/storage \
-v /volume1/Downloads/_Rips:/input \
-v /volume1/Downloads/_Movies:/output \
-e TZ=America/Los_Angeles \
-e USER_ID=`id -u` \
-e GROUP_ID=`id -g` \
-e UMASK=000 \
-e AUTOMATED_CONVERSION_PRESET=Rip\Default \
-e AUTOMATED_CONVERSION_FORMAT=mp4 \
jlesage/handbrake
```

PS. Check if `/output` folder inside container has write permissions.

#### Passthru subtitles

> To pass-thru, every found subtitle, create a custom MKV preset with the video and audio settings required and give it a name. Click the Subtitles tab then Selection Behavior. Set Track Selection Behavior dropdown to All Matching Selected Languages. Move (Any) from the left-hand Available Languages column so it appears on its own in the right-hand Selected Languages column.
Ensure Add Closed Captions and Add Foreign scan boxes are both unticked.
Set Burn-in behavior to None
Click Apply to close the window. Right-click the preset name you created and click Update Selected Preset to store your new subtitle default. Load an MKV to check.



