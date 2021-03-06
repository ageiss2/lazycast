lazycast: A Simple Wireless Display Receiver

# Description
lazycast is a simple wifi display receiver. It was originally targeted Raspberry Pi (as display) and Windows 8.1/10 (as source), but it **might** also work on other Linux platforms and Miracast sources. (For other Linux systems, skip the preparation section.) In general, lazycast does not require re-compilation of wpa_supplicant to provide p2p capability, and should work on an "out of the box" Raspberry Pi.

# Preparation
**The wpa_supplicant installed on the latest Raspbian distribution does not seem to work properly. (See [this](https://www.reddit.com/r/linux4noobs/comments/c5qila/want_to_downgrade_wpa_supplicant/).) For Raspbian Buster, try downgrading the ``wpasupplicant`` package to the version for Raspbian Stretch. Here is one solution:**
```
wget http://ftp.us.debian.org/debian/pool/main/w/wpa/wpasupplicant_2.4-1+deb9u4_armhf.deb
sudo apt --allow-downgrades install ./wpasupplicant_2.4-1+deb9u4_armhf.deb
```  
**It is also highly recommended to replace the "Wireless & Wired Network" in Raspbian with NetworkManager, which can maintain much more stable p2p connection. Here is one solution (adopted from [here](https://raspberrypi.stackexchange.com/questions/29783/how-to-setup-network-manager-on-raspbian)):**
```
sudo apt install network-manager network-manager-gnome openvpn openvpn-systemd-resolved network-manager-openvpn network-manager-openvpn-gnome
```
And,
```
sudo apt purge dhcpcd5
```
Additionally, ``systemd-resolved`` should be disabled since it does not seem to work well with NetworkManager, which causes DNS problems. (See [here](https://unix.stackexchange.com/questions/518266/ping-displays-name-or-service-not-known) for details.) (It may take a while for the problems to show).
```
sudo systemctl disable systemd-resolved
```
Then reboot:
```
sudo reboot
```
Install packages used to compile the players:
```
sudo apt install libx11-dev libasound2-dev libavformat-dev libavcodec-dev
```
Compile libraries on Pi:
```
cd /opt/vc/src/hello_pi/libs/ilclient/
make
cd /opt/vc/src/hello_pi/hello_video
make
```
Clone this repo (to a desired directory):
```
cd ~/
git clone https://github.com/homeworkc/lazycast
```
Go to the ``lazycast`` directory and then ``make``:
```
cd lazycast
make
```

# Usage
Run `./all.sh` to initiate lazycast receiver. Wait until the "The display is ready" message.
The name of your device will also be displayed on the pi.
Then, search for the wireless display on the source device you want to cast. The default PIN number is ``31415926``.  
If backchannel control is supported by the source, keyboard and mouse input on Pi are redirected to the source as remote controls.  
It is recommended to initiate the termination of the receiver on the source side. These user controls are often near the pairing controls on the source device. You can utilize the backchannel feature to remotely control the source device in order to close lazycast.  



# Tips
Set the resolution on the source side. lazycast advertises all possible resolutions regardless of the current rendering resolution. Therefore, you may want to change the resolution (on the source) to match the actual resolution of the display connecting to Pi.  
Modify parameters in the "settings" section in ``d2.py`` to change the sound output port (hdmi/3.5mm) and preferred player.  
The maximum resolutions supported are 1920x1080p60 and 1920x1200p30. The GPU on Pi may struggle to handle 1920x1080p60, which results in high latency. In this case, reduce the FPS to 1920x1080p50.  
To change the default PIN number, replace the string ``31415926`` in ``all.sh`` to another 8-digit number.  
After Pi connects to the source, it has an IP address of ``192.168.173.1`` and this connection can be reused for other purposes like SSH. On the other hand, since they are under the same subnet, precautions should be taken to prevent unauthorized access to Pi by anyone who knows the PIN number.    
Two in-house players are written for Raspberry Pi 3. VLC, omxplayer or gstreamer can be used instead on other platforms. (See [here](https://gstreamer.freedesktop.org/documentation/installing/on-linux.html) for details of installing gstreamer.) 


# Known issues
lazycast tries to remember the pairing credentials so that entering the PIN is only needed once for each device. However, this feature does not seem to work properly all the time with recent Raspbian images. (Using the latest Raspbian is still recommended from the security perspective. However, recent Raspbians randomize the MAC address of the ``p2p-dev-wlan0`` interface upon reboot, while old Raspbians ([example](https://downloads.raspberrypi.org/raspbian/images/raspbian-2017-09-08/)) do not. **Any insights or suggestions on this issue are appreciated**, and could make this important feature work again.) Therefore, re-pairing may be needed after every Raspberry Pi reboot. Try clearing the 'lazycast' information on the source device before re-pairing if you run into pairing problems.  
Latency: Limited by the implementation of the rtp player used. (In VLC, latency can be reduced from 1200 to 300ms by lowering the network cache value.)  
The on-board WiFi chip on Pi 3 only supports 2.4GHz. Therefore, devices/protocols that use 5.8GHz for P2P communication (e.g. early generations of WiDi) are not ("out of the box") supported.  
Due to the overcrowded nature of the wifi spectrum and the use of unreliable rtp transmission, you may experience some video glitching/audio stuttering. The in-house players employ several mechanisms to conceal transmission error, but it may still be noticeable in challenging wireless environments. Interference from other devices may cause disconnections.  
Devices may not fully support backchannel control and some keystrokes/clicks will behave differently in this case. The left Windows key is not captured and when it is pressed, it makes the current window to be out-of-focus and thus disables the backchannel controls. If it is pressed again the window will be in-focus.   
HDCP(content protection): Neither the key nor the hardware is available on Pi and therefore is not supported.  

# Start on boot

## Method 1:
Append this line to ``/etc/xdg/lxsession/LXDE-pi/autostart``:
```
@lxterminal -l --working-directory=<absolute path of lazycast> -e ./all.sh
```
For example, if lazycast is placed under ``~/`` (which corresponds to ``/home/pi/``), append the following line to the file:
```
@lxterminal -l --working-directory=/home/pi/lazycast -e ./all.sh
```


## Method 2: SystemD

You can run lazycast when booting your Pi using the [systemd unit](lazycast.service). To install, log into your Pi and run:

```bash
git clone https://github.com/homeworkc/lazycast.git
mkdir -p ~/.config/systemd/user
cp lazycast/lazycast.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable lazycast.service
systemctl --user start lazycast.service
sudo loginctl enable-linger pi
```

NOTE: In this method, the systemd unit expects lazycast to be located under `/home/pi/lazycast`, adjust the WorkingDirectory if this is not the correct path.

# Miracast over Infrastructure (Under Development)
The spec of Miracast over Infrastructure (MICE) is available [here](https://winprotocoldoc.blob.core.windows.net/productionwindowsarchives/MS-MICE/%5bMS-MICE%5d.pdf). A reference connection can be established by two Windows 10 computers. The simpliest setting would be connecting via Ethernet (with static IP assignments). To further simplify the situation, WiFi should be enabled but the two computers should be disconnected from APs. Then, one computer should run the "Connect" app while the other tries to connect to this computer as a "wireless" display. Once connected, run task manager and see whether the majority of the traffic goes through Ethernet (as opposed to WiFi Direct.) The network trace (e.g., using Wireshark) collected from the reference system would be useful for implementing MICE on Pi.  

At this stage, the implementation of device discovery phase, as depicted in Figure 2 in the spec, is completed. Specifically, the Vendor Extension Attribute can be successfully added to the Beacon and Probe Response frames, as described in Section 2.2.8 and Section 4.1 in the spec. What we should do next is implementing the host name resolution phase, since using the IP Address Attribute does not seem to work. Specifically, we need to implement Section 3.1.3 in the spec so that a TCP connection on port 7250 can be established successfully.    

The device discovery phase requires a recompiled (with the default configuration) version of wpa_supplicant.
First, install required packages:
```
sudo apt install libdbus-1-dev libnl-3-dev libnl-genl-3-dev libssl-dev
```
Then, download the source of wpa_supplicant:
```
cd ~/
wget https://w1.fi/releases/wpa_supplicant-2.9.tar.gz
tar -xvf wpa_supplicant-2.9.tar.gz
```
Compile wpa_supplicant:
```
cd wpa_supplicant-2.9/wpa_supplicant
cp defconfig .config
make
```
Finally copy the new binaries to ``/usr/local/bin`` and reboot:
```
sudo mv /usr/local/bin/wpa_supplicant /usr/local/bin/wpa_supplicant_old
sudo cp wpa_cli wpa_supplicant /usr/local/bin
sudo rm /usr/local/bin/wpa_supplicant_old
sudo reboot
```
After reboot, make sure ``./all.sh`` is not running and has not been running. To double check, if you run ``sudo wpa_cli interface``, it should say ``Selected interface 'p2p-dev-wlan0'`` and not ``Selected interface 'p2p-wlan0-0'``.
Then, run ``./mice.sh``, which will set the Vendor Extension Attributes as described in Section 2.2.8 and Section 4.1. Note that the name of the display is the hostname of Pi (e.g., raspberrypi) since other names do not seem to work for Windows 10 computers. If you want to check whether the Vendor Extension Attributes are set properly, you can run ``python scan.py`` on a separate Pi or analyze the beacon frames using Wireshark on Linux computers (with WiFi) and see whether it shows outputs similar to the example hexdump in Section 4.1. The last line of ``mice.sh`` awaits a connection request on TCP port 7250, which is the beginning of the projection phase. Now you can try various mDNS and DNS settings and see whether you can successfully establish the connection.


# Others
Some parts of the video player1 are modified from the codes on https://github.com/Apress/raspberry-pi-gpu-audio-video-prog. Many thanks to the author of "Raspberry Pi GPU Audio Video Programming" and, by extension, authors of omxplayer.  
Using any part of the codes in this project in commercial products is prohibited.
