## Pre-installation setup

Before installing Raspberry Pi OS Lite, it's recommended to configure your system with the following settings:
- Username: plv
- Password: visualizer
- Local hostname: pianoledvisualizer.local

These settings can be easily configured using the **Raspberry Pi Imager** tool:
1. Open Raspberry Pi Imager
2. Select Raspberry Pi OS Lite (for RPi Zero) as your operating system
3. Click the gear/cog icon to access advanced options
4. Set the hostname to "pianoledvisualizer.local"
5. Enable SSH
6. Set username to "plv" and password to "visualizer"
7. Configure your WiFi settings if needed
8. Save settings and write the image to your SD card

This configuration will make it easier to connect to your Raspberry Pi using SSH via `ssh plv@pianoledvisualizer.local`

---

Install [Raspberry Pi OS Lite](https://www.raspberrypi.org/software/) on your SD card.

If you are not able to connect your monitor, mouse and keyboard to RPi you can connect to it using SSH over [Wi-Fi](https://github.com/onlaj/Piano-LED-Visualizer/blob/master/Docs/wifi_setup.md)

Run installation script:

`sudo bash -c "$(curl -fsSL https://raw.githubusercontent.com/onlaj/Piano-LED-Visualizer/master/autoinstall.sh)"`

**or follow those steps:**
 
### 1. **Updating OS** 
After succesfully booting RPi (and connecting to it by SSH if necessary) we need to make sure that everything is up to date.
- `sudo apt-get update`
- `sudo apt-get upgrade` //*it will take a while, go grab a coffee*


###  2. **Enabling SPI interface** ### 
 - Here you can find instruction: [Enable SPI Interface on the Raspberry Pi](https://www.raspberrypi-spy.co.uk/2014/08/enabling-the-spi-interface-on-the-raspberry-pi/)
 - Or simply use this command:

```bash
sudo raspi-config nonint do_spi 0
```

### 3. **Installing packages** //*ready for another cup?* ### 

```bash
sudo apt-get install -y ruby git python3-pip autotools-dev libtool autoconf libasound2 libavahi-client3 libavahi-common3 libc6 libgcc-s1 libstdc++6 python3 libopenblas-dev libavahi-client-dev libasound2-dev libusb-dev libdbus-1-dev libglib2.0-dev libudev-dev libical-dev libreadline-dev libopenjp2-7 libtiff6 libjack0 libjack-dev fonts-freefont-ttf gcc make build-essential scons swig abcmidi
```


### 4. **Disabling audio output** ### 

    sudo nano /etc/modprobe.d/snd-blacklist.conf
- paste and save:

    `blacklist snd_bcm2835`
- And one more file:

    `sudo nano /boot/config.txt`
- Change `dtparam=audio=on` to `#dtparam=audio=on`

- Reboot RPi

`sudo reboot`


### 5. **Installing RTP-midi server** (optional) ### 
*This part is not needed if you're not going to connect your RPi to PC.*

We are going to use  [RTP MIDI User Space Driver Daemon for Linux](https://github.com/davidmoreno/rtpmidid/releases)

- Navigate to /home folder:

`cd /home/`

- Download and install the prerequisite `libfmt9` package:

`sudo wget http://ftp.de.debian.org/debian/pool/main/f/fmtlib/libfmt9_9.1.0+ds1-2_arm64.deb`

`sudo dpkg -i libfmt9_9.1.0+ds1-2_arm64.deb`

`sudo apt -f install`

- Download and install `rtpmidid` package:


`sudo wget https://github.com/davidmoreno/rtpmidid/releases/download/v24.12/rtpmidid_24.12.2_armhf.deb`

`sudo dpkg -i rtpmidid_24.12.2_armhf.deb`

`sudo apt -f install`

### 6. **Installing Piano-LED-Visualizer** ###
- Navigate to /home folder:

`cd /home/`

- GIT clone repository

`sudo git clone https://github.com/onlaj/Piano-LED-Visualizer`

`cd Piano-LED-Visualizer`
- Install required libraries

`sudo apt-get install -y python3-rpi.gpio python3-webcolors python3-psutil python3-mido python3-pillow python3-rtmidi python3-spidev python3-numpy python3-flask python3-waitress python3-websockets python3-werkzeug`

`sudo pip3 install rpi-ws281x --break-system-packages`

- Enable autologin on boot

`sudo raspi-config`

`Select "System options" then “Boot / Auto Login” then “Console Autologin” `
- Enable autostart script on boot:

`sudo nano /lib/systemd/system/visualizer.service`

Paste and save:

```bash
[Unit]
Description=Piano LED Visualizer
After=network-online.target
Wants=network-online.target

[Install]
WantedBy=multi-user.target

[Service]
ExecStart=sudo python3 /home/Piano-LED-Visualizer/visualizer.py
Restart=always
Type=simple
User=plv
Group=plv
```

*If you are using WaveShare 1.3inch 240x240 LED Hat instead of 1.44inch 128x128, edit accordingly:*
`ExecStart=sudo python3 /home/Piano-LED-Visualizer/visualizer.py --display 1in3`

*If you want to use your RPi upside down add `--rotatescreen true` :*

`ExecStart=sudo python3 /home/Piano-LED-Visualizer/visualizer.py --rotatescreen true`

- Reload daemon and enable service:

   `sudo systemctl daemon-reload`
   
   `sudo systemctl enable visualizer.service`
    
   `sudo systemctl start visualizer.service`


- Change permissions:

  `sudo chmod a+rwxX -R /home/Piano-LED-Visualizer/`

Now you can type `sudo reboot` to test if everything works. After 1-3 minutes you should see Visualizer menu on RPi screen.

### 7. **Start/stop when piano is turned on/off (optional)** ###

If you want to leave the Raspberry Pi on at all times, you can set it up to automatically start and stop the visualizer service when the piano is turned on and off. There are two parts to this.

#### Udev rule ####

This will start the service when the piano is turned on.

- First, run `lsusb` while the piano is on and plugged in and find the line for your piano. For example:

```
Bus 001 Device 096: ID 0f54:0104 Kawai Musical Instruments Mfg. Co., Ltd USB func for MIDI
```

- Note the values after "ID", in this case `0f54:0104`. Those are the the vendor and product IDs.

- Create a file called `/etc/udev/rules.d/32-startvisualizer.rules`, with the following content:

```
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="0f54", ATTRS{idProduct}=="0104", TAG+="systemd", ENV{SYSTEMD_WANTS}="visualizer.service"
```

- Replace the `idVendor` and `idProduct` values with the values you got from `lsusb`.

#### Create service dependency ####

This will prevent the service from starting when the Pi turns on if the piano is not on, and will stop the service when the piano is turned off.

- Run `systemctl list-units --type=device --all --full | grep dev-snd-by` and note the device path for your piano. There will likely be multiple values, like the sample output below. Pick the first that looks like it corresponds to your piano. If you find it doesn't work, you can go back and pick a different one.

```
dev-snd-by\x2did-usb\x2dKAWAI_MI._Mfg._Co._LTD._USB_func_for_MIDI_00000001\x2d00.device   loaded active   plugged USB_func_for_MIDI
dev-snd-by\x2dpath-platform\x2d3f980000.usb\x2dusb\x2d0:1.3:1.0.device                    loaded active   plugged USB_func_for_MIDI
dev-snd-by\x2dpath-platform\x2d3f980000.usb\x2dusbv2\x2d0:1.3:1.0.device                  loaded active   plugged USB_func_for_MIDI
```

- Edit `/lib/systemd/system/visualizer.service` and under the `[Unit]` section add these lines:

```
After=dev-snd-by\x2did-usb\x2dKAWAI_MI._Mfg._Co._LTD._USB_func_for_MIDI_00000001\x2d00.device
BindsTo=dev-snd-by\x2did-usb\x2dKAWAI_MI._Mfg._Co._LTD._USB_func_for_MIDI_00000001\x2d00.device
```

- Replace the values with the device path you found when running `systemctl`.

- Restart the Pi and confirm that the visualizer service starts and stops when the piano turns on and off
