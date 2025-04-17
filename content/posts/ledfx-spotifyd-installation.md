---
layout: post
tags: ["Blog","LEDs","RaspberryPi"]
slug: ledfx-spotifyd-guide
title: "Making LEDs React to Music: Configuring LEDFx and Spotifyd on the Raspberry Pi"
summary: "Setting up LEDFx and Spotifyd to build a Spotify Connect speaker that drives music-reactive LED strips."
date: 2023-11-20T17:33:32-08:00
math: false
draft: false
---
This was originally a rough guide I made for myself to retrace my steps in case I needed to re-build my LEDFx setup at some point. I ended up polishing it up and publishing it as a [Gist](https://gist.github.com/0xjmux/e5f49cb756ab94540df6637b77af8ee1) at the request of some people in the LEDFx discord, and since people seemed to find it so helpful I'm re-publishing it here.

<!-- If it looks a lot like the LEDFx documentation's installation instructions, that's because I've also contributed it upstream. LEDFx is why my iot_leddriver_hw switched directions, and is a really cool project. It was already more full featured than I was planning on making my Spotify Visualizer firmware, and upstream community contribution is always a good thing :) -->

This project wasn't difficult, but it also wasn't necessarily straightforward. The main problems I ran into were with Linux audio (to absolutely no ones surprise) and having to recompile dependencies to get them to work. Additionally the whole process of setting up a user daemon and getting everything to play nice can be a bit challenging, especially for those new to Linux; so I've put together a reasonably thorough installation guide below.


# Installation
This setup was tested on 2023-11-20 on a Raspberry Pi 4 with Raspberry Pi OS Lite 2023-10-10 (Debian 12 Bookworm).

> [!TIP]
> Music visualization requires a good amount of processing power to work without significant lag. Anything older than a Pi 4 is not recommended.


This guide goes over the installation of both Spotifyd  and Ledfx. Spotifyd allows you to use your Pi as a Spotify connect speaker, but can be replaced with whatever audio source you plan to use - but since most of the problems people encounter are with audio setup and not LedFX itself, showing the entire process should be helpful.


### 1. Updates and dependencies
Starting with a fresh installation:

```sh
apt update && sudo apt upgrade -y
apt install -y python3-pip python3-venv pulseaudio portaudio19-dev git
apt install libasound2-dev libssl-dev libpulse-dev libdbus-1-dev -y
```

### 2. Set up user
We create user `ledfx`  to run everything (including Pulseaudio and our Ledfx and Spotifyd services) to make handling permissions nice and easy.

```sh
adduser ledfx
usermod -a -G pulse-access,pulse,plugdev,audio ledfx
````

2. Reboot so Pulseaudio will be available for next steps
```sh
sudo reboot
```

### 3. Set up Pulseaudio
2. User login persistence (without `enable-linger`, desktop programs can't run without an active session)
(as root)
```sh
loginctl enable-linger ledfx
```

3.  Edit `/etc/pulse/system.pa` and add a line to include `module-alsa-card`

```conf
### Automatically restore the volume of streams and devices
load-module module-device-restore
...
load-module module-alsa-card
```

Exit root and change back to user.

3. (as user) Test Pulseaudio
```sh
pulseaudio -D
```

1. Logout from SSH session and login to `ledfx` account over ssh (See [Failed to initialize DBus connection Error]({{< ref "#error-failed-to-initialize-dbus-connection" >}}) below.


```sh
logout
ssh ledfx@[IP ADDR]
```

### 4. Set up spotifyd
* [Github Releases · Spotifyd/spotifyd](https://github.com/Spotifyd/spotifyd/releases)
* [Installation Instructions - Spotifyd Docs](https://docs.spotifyd.rs/installation/index.html)
* [Installing on a Raspberry Pi - Spotifyd Docs](https://docs.spotifyd.rs/installation/Raspberry-Pi.html) (doesn't completely work as of 11/13/23)

> [!Note]
> As of 2023-11-15, I had to compile spotifyd from source with all the features I needed to get it to work on the pi, none of the prebuilt binaries I tried would run. By the time you are reading this, that might have changed.

```sh
# install rust through rustup
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
cd spotifyd
```

2. Compile from source with the pulseaudio backend
3.
```sh
cargo build --release --no-default-features --features pulseaudio_backend,dbus_keyring,dbus_mpris

cargo install --path . --locked
```

2.1. Copy Spotifyd to location in default path
```sh
cp spotifyd /usr/local/bin/
```

3. Set up user service

* [spotifyd.service - Recommended Systemd config from Spotifyd](https://raw.githubusercontent.com/Spotifyd/spotifyd/master/contrib/spotifyd.service)
```sh
wget https://raw.githubusercontent.com/Spotifyd/spotifyd/master/contrib/spotifyd.service
```
4. Set up Spotifyd config file
```sh
mkdir -p ~/.config/systemd/user/

# use this config file to modify necessary parameters; see link to spotifyd docs below
vim ~/.config/systemd/user/spotifyd.service
```
Remember to modify the `ExecStart=` to the path of your `spotifyd` executable

```sh
[Service]
ExecStart=/usr/local/bin/spotifyd --no-daemon
```

* For information on Spotifyd configuration, see: [Configuration file - Spotifyd Docs](https://docs.spotifyd.rs/config/File.html)
5. Start Spotifyd service

```sh
# Test service to see if there are any errors
./spotifyd --no-daemon
```

6. Set up as Systemd service so it starts on boot

```sh
# Once it works (or gives dbus error) set up as systemd service
systemctl --user enable spotifyd.service --now
systemctl --user enable spotifyd.service
systemctl --user daemon-reload
vim .config/systemd/user/spotifyd.service
chown root:ledfx .config/systemd/user/spotifyd.service
systemctl --user start spotifyd.service
```

* See [Pulseaudio issues](#pulseaudio-issues) below if you have issues

### 5. Prepare python environment
```sh
python -m venv .ledfx
source .ledfx/bin/activate
python -m pip install --upgrade pip wheel setuptools
pip install numpy~=1.23
```

(recommended): Activate the ledfx python venv on login
```sh
echo "source $HOME/.ledfx/bin/activate" >> .bashrc
```

#### 5.1 (Optional) Install python-mbedtls
> [!NOTE]
> This step is optional, and only needed if you're going to use LEDFx with Phillips Hue lights. This library can cause issues during installation (since it doesn't provide a precompiled ARM binary), so **unless you are using HUE bulbs installing it is not recommended.**

```sh
git clone https://github.com/Synss/python-mbedtls.git && cd python-mbedtls/
./scripts/download-mbedtls.sh 2.28.3 ~/.local/src
sudo ./scripts/install-mbedtls.sh ~/.local/src
export C_INCLUDE_PATH="/usr/local/include"
export LIBRARY_PATH="/usr/local/lib"
```

### 6. Install LEDFx
Install the latest version from git:
```sh
pip install git+https://github.com/LedFx/LedFx
```
(Alternatively, Install the latest release available on Pypi)
```sh
pip install ledfx
```

### 7. Set up LEDFx as systemd daemon
Create a new file at `~/.config/systemd/user/ledfx.service` and add the following:

```conf
[Unit]
Description=LedFx Music Visualizer
After=network.target sound.target
Wants=sound.target
After=sound.target
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
ExecStart=/home/ledfx/.ledfx/bin/python /home/ledfx/.ledfx/bin/ledfx -v

[Install]
WantedBy=default.target
```

Test your service:
```sh
systemctl start --user ledfx
systemctl status --user ledfx
```

Now, connect to the Spotify connect speaker on your network and give it a test!


### 8. Finishing Touches
Now that you're done, you should:

* [Configure unattended-upgrades so your Pi can apply non-breaking updates itself](https://www.linuxcapable.com/how-to-configure-unattended-upgrades-on-ubuntu-linux/)
* Plan to log in every couple weeks to update LEDFx, since releases are frequent:

(as `ledfx` user)
```sh
pip install -U ledfx
```

# Troubleshooting
## Spotifyd
The issues listed here are mostly related to getting LedFx and spotifyd working with pulseaudio on the Pi, since that's where there seems to be the most issues.

### Error: Failed to initialize DBus connection
Error: `Failed to initialize DBus connection: D-Bus error: Unable to autolaunch a dbus-daemon without a $DISPLAY for X11`
* This error happens because we're using a `ledfx` user account to run the service, but haven't "fully" logged in yet, so some environment variables (like `$XDG_RUNTIME_DIR`) haven't been created yet. [1](https://unix.stackexchange.com/questions/615917/failed-to-get-d-bus-connection-connection-refused)
* To fix this, log out, and log in as the `ledfx` user over ssh.

### Pulseaudio issues
Try:
* make sure pulseaudio is running as user service
    * Or, try running manually with `pulseaudio -D` (background)
    * look at logs for user pulseaudio service:
        `journalctl -xe --user-unit pulseaudio`
* `pactl info` for server and running pulseaudio session info
* `pactl list-cards` to list audio cards
* `pacmd list-sinks` to list output devices
    * Make sure your device is set as default
    * try `raspi-config` to set it, or modify it using pulseaudio
* [PulseAudio Wiki: Set default output sink](https://wiki.archlinux.org/title/PulseAudio/Examples#Set_the_default_output_sink)

#### Pulseaudio: failed to load `module-alsa-card`
- this warning/error shows up when running on the Pi. The HDMI outputs will continue to have it (since there's nothing connected) but I was able to resolve it for the interface I'm using by adding
* [Relevant ArchWiki Post: Pulseaudio: failed to load module-alsa-card](https://bbs.archlinux.org/viewtopic.php?id=259938)

## LEDFx
I've had some issues the last few weeks with LEDFx crashes, and since (from experience teaching people Linux through CCDC) I know figuring out crashes can be a bit obtuse, I'm going to leave some tips here.

### Diagnosing Errors
The first thing you should do is check the service status.
```sh
systemctl status --user ledfx
systemctl status --user spotifyd
```

If the status isn't `active`, something's wrong. To look at the logs, you can use:
```sh
journalctl --user -xe
```

One of the things I find helpful is to watch the logs in realtime when trying to debug a process. To do this, you can use something like `tmux` (`apt install tmux -y`) to have two terminal windows; on one of them, run:
```sh
journalctl --user -f
```
And then on the other, restart the service with
```sh
systemctl --user restart ledfx
```


### Fixing issues
1. First, check the [LEDFx Docs Troubleshooting section](https://ledfx.readthedocs.io/en/stable/trouble.html#ledfx-configuration-file) to see if what you're struggling with is mentioned there.
2. Look at [open issues on Github](https://github.com/LedFx/LedFx/issues) to see if this has been reported already - if not, open an issue so it can be fixed!
3. If not, the issue might be in your configurations - check to make sure they're valid.
4. Ask for help in the [LEDFx discord](https://discord.com/invite/xyyHEquZKQ)
