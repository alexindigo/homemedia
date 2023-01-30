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

```
$ sudo vi /lib/systemd/system/abcde.service
```

Add

```
[Unit]
Description=ABCDE Rip

[Service]
ExecStart=/usr/bin/abcde
#ExecStopPost=/usr/bin/eject

[Install]
WantedBy=multi-user.target
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



# Prior art

- [aac/raspberrypi-cd-ripper](https://github.com/aac/raspberrypi-cd-ripper)
