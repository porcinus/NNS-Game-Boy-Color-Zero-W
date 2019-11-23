# NNS Game Boy Color Zero W : Software Part

[Home](https://github.com/porcinus/NNS-Game-Boy-Color-Zero-W/)  

If you want to skip this whole part (almost):  
- Download this image (based on Freeplay_Zero_19062002.img from Freeplaytech) : [nns-gbc-15-10-19-shrink.img.gz](https://drive.google.com/file/d/1-Cm409LgLcA2TGLhYf3OCXbFeWlYMwc1/view?usp=sharing) (MD5: 9539F67D53D15E3EA0993A911DB1AB1D)  
- Burn the image on a SD card using Etcher or equivalent software.  
- Create `wpa_supplicant.conf` in Boot partition with content:  
```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=US
network={
	ssid="PUT YOUR WIFI SSID HERE"
	psk="PUT YOUR WIFI PASSWORD HERE"
}
```



## Start from "scratch"
- Download Zero SD image from [Freeplaytech website](https://www.freeplaytech.com/support/setup/)  
- Burn the image on a SD card using Etcher or equivalent software.  


## Initial modifications
- Edit `cmdline.txt` located in Boot partition and add at end of the first line '` bcm2708.vc_i2c_override=1`', don't miss to add a space before.  

- Edit `config.txt` located in Boot partition:  
	- In general section : Comment lines `dtparam=spi=on`, `dtparam=audio=on`  
	- In `[all]` section : Replace `dtoverlay=gpio-poweroff,gpiopin=21,active_low` with `dtoverlay=gpio-poweroff,gpiopin=5,active_low`  
	- In `[all]` section : Comment `audio_pwm_mode=2`  
	- In `[pi0]` section : Comment lines `dtoverlay=audremap`, `dtoverlay=waveshare32b-fp`  
	- In `[pi0]` section : Add `dtoverlay=hifiberry-dac`, `dtoverlay=i2s-mmap`, `force_eeprom_read=0`, `disable_poe_fan=1`, `dtoverlay=i2c-rtc,ds3231`, `gpio=7=ip,pd`  

- Create `wpa_supplicant.conf` in Boot partition with content:  
```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=US
network={
	ssid="PUT YOUR WIFI SSID HERE"
	psk="PUT YOUR WIFI PASSWORD HERE"
}
```


**Next part need to be done with the device powered from USB Data port with no battery attached.**  
**All of these commands with be sent thru SSH connection, to know the Zero IP adress, please check you router.**  
Raspbian default login cremential are user : **pi** and password : **raspberry**.  


### Edit shutdown daemon
Default pin used for power switch and low battery interrupt are not the same as Freeplaytech device. These need to be changed to avoid security shutdown.  

`sudo nano /home/pi/Freeplay/Freeplay-Support/shutdown_daemon.py` and change **STOP_REQ** from **20** to **7** and **LOW_BATT** from **7** to **4**.  


### Bypass some troubles and increase screen sharpness
`setPCA9633` is used to avoid PWM output running full throttle (please check Readme/Note/PWM).  

GPIO0 and GPIO20 need to be set as input after the initial boot sequence because these are set to ALT when system boot.  
Please note that `gpio` program use WiringPi pin assignement instead of the BCM one, here GPIO0=30 and GPIO20=28.  

`vcgencmd` is here to increase image sharpness a little bit.  

`sudo nano /lib/systemd/system/nns_bootfix.service` with content:  
```
[Unit]
Description=boot fix
Before=basic.target
After=local-fs.target sysinit.target
DefaultDependencies=no

[Service]
Type=oneshot
ExecStart=/home/pi/Freeplay/setPCA9633/setPCA9633 --led1=ON --led2=ON --led3=ON --pwm1=0xFF --pwm2=0xFF --pwm3=0xFF
ExecStartPost=/usr/bin/gpio mode 30 in
ExecStartPost=/usr/bin/gpio mode 28 in
ExecStartPost=/usr/bin/vcgencmd scaling_sharpness 80

[Install]
WantedBy=basic.target
```

Then `sudo systemctl enable nns_bootfix.service` and shutdown using `sudo poweroff`.  

**Once device full shutdown, unplug USB Data and reconnect the battery.**  


### Enable Root login
This part is not really needed but it can ease the work if you use a SCP client intead of SSH client.  

`sudo nano /etc/ssh/sshd_config` and replace '`#PermitRootLogin without-password`' by '`PermitRootLogin yes`'  
Then `/etc/init.d/ssh restart` to restart SSH daemon and `sudo passwd root` to set root password.  
Once done, restart the system with `sudo reboot`, after you should be able to connect using **root** user.  


### Create Asound config
Please follow instructions [HERE](https://learn.adafruit.com/adafruit-max98357-i2s-class-d-mono-amp/pi-i2s-tweaks).  

`aplay --list-devices` display all audio device, it should content '**HifiBerry DAC HiFi pcm5102a-hifi-0**'  
Tests can be done using `omxplayer -o alsa YOUR-AUDIO-OR-VIDEO-FILE`  


### Install ADC2ALSAMixer daemon
This daemon is used to update ALSA audio volume based on ADC input.  
Its main purpuse here is to avoid audio traces running thru the board and doing so, reduce the noises.  

`cd /home/pi/NNS`  
`git clone https://github.com/porcinus/ADC2ALSAMixer`  
`cd ADC2ALSAMixer`  
`./compile.sh`  
`sudo ./install-gbc.sh`  

To check if the daemon work fine:  
`alsamixer`  


## Check that everything is fine
### I2C
`apt-get install i2c-tools`  
`sudo i2cdetect -y 1`  

You should see:  
- PCA9633 : **03**, **62**(master), **70**(subaddress)  
- MAX17048 : **36** (only when battery is plugged)  
- ADS1015 : **48**  
- MCP3221 : **4D**  
- DS3231 : **68** (will be **UU** if the driver is working)  

### GPIO
In case you need to do some debug in GPIOs or if you have a doubte, 2 commands can be useful:  

`gpio readall` display all GPIO pins status once.  
`watch -n 0.1 -d gpio readall` does the same as previous command but in "continuous" mode and showing what did change in interval of 100msec.  

### RTC
To check if RTC IC is working fine, you can use `sudo timedatectl` and check for 'RTC time' line.  
To update Timezone, run `sudo raspi-config` > **Localisation Options** > **Change Timezone** > Choise your Geographic area > Choise the right city for your TZ > **Finish**.  
Once done, restart the system with `sudo reboot` in case something goes wrong.  


# Setup TFT driver
### Disable Freeplay TFT driver
`sudo systemctl stop fbcpOld.service`  
`sudo systemctl stop fbcpCropped.service`  
`sudo systemctl stop fbcpFilled.service`  
`sudo systemctl stop fbcpZero.service`  
`sudo systemctl disable fbcpOld.service`  
`sudo systemctl disable fbcpCropped.service`  
`sudo systemctl disable fbcpFilled.service`  
`sudo systemctl disable fbcpZero.service`  

### Compile juj fbcp-ili9341
`cd /home/pi`  
`mkdir NNS`  
`cd NNS`  
`git clone https://github.com/juj/fbcp-ili9341`  
`cd fbcp-ili9341`  
`nano config.h` and comment **SAVE_BATTERY_BY_PREDICTING_FRAME_ARRIVAL_TIMES**, **SAVE_BATTERY_BY_SLEEPING_WHEN_IDLE**.  
`mkdir build`  
`cd build`  
`cmake -DSINGLE_CORE_BOARD=ON -DARMV6Z=ON -DILI9341=ON -DGPIO_TFT_DATA_CONTROL=24 -DGPIO_TFT_RESET_PIN=25 -DSPI_BUS_CLOCK_DIVISOR=6 -DDISPLAY_BREAK_ASPECT_RATIO_WHEN_SCALING=ON -DUSE_DMA_TRANSFERS=ON -DDMA_TX_CHANNEL=0 -DDMA_RX_CHANNEL=1 -DSTATISTICS=0 ..`  
`make -j`  

Test if it is working : `sudo /home/pi/NNS/fbcp-ili9341/build/fbcp-ili9341`  

If this doesn't work, please check TFT ribbon cable, once done :  
`cd /home/pi/NNS/fbcp-ili9341/build/`  
`rm -rf *`  

Restart from `cmake` command but changing **DDMA_TX_CHANNEL** and **DDMA_RX_CHANNEL**, if you are having troubles, you can also check out [Freeplaytech forum](https://forum.freeplaytech.com/showthread.php?tid=4989), I did make a post that goes deeper in this summary.  

### Start juj fbcp-ili9341 at system boot
`nano /home/pi/NNS/fbcp-ili9341/jujfbcp.service` with content:  
```
[Unit]
Description=juj fbcp ili9341 display driver
After=basic.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=2
User=root
KillSignal=SIGKILL
ExecStart=/home/pi/NNS/fbcp-ili9341/build/fbcp-ili9341

[Install]
WantedBy=multi-user.target
```

Then  
`sudo cp /home/pi/NNS/fbcp-ili9341/jujfbcp.service /lib/systemd/system/jujfbcp.service`  
`sudo systemctl enable jujfbcp.service`  
`sudo systemctl start jujfbcp.service`  

If for some reason you need to recompile the driver, you will need to stop and re-start the service using:  
`sudo systemctl stop jujfbcp.service`  
`sudo systemctl start jujfbcp.service`  


## Update MK GPIO driver
`cd /home/pi/Freeplay`  
`sudo mv mk_arcade_joystick_rpi mk_arcade_joystick_rpi_old`  
`git clone https://github.com/TheFlav/mk_arcade_joystick_rpi`  
`cd mk_arcade_joystick_rpi`  
`sudo ./install.sh`  

Note: if the kernel is updated during `install.sh` execution, you will have to restart the system and :  
`cd /home/pi/Freeplay/mk_arcade_joystick_rpi`  
`sudo ./install.sh`  

Once driver updated, run `sudo make config` to make a new config, this is only related to MK driver.  
Quick note:  
Sometime, for some reasons, the software is having hard time with GPIO. If this is the case, you may need to restart the system.  
This command will restart the system at the end, once restarted, Emulationstation with require you to remake your binding.  

You can also test/debug running `jstest /dev/input/js0`  


## Let's install/do some cool stuff

### Retroarch based emulators
- Force original screen ratio :  
`sudo nano /opt/retropie/configs/all/retroarch.cfg` and set **aspect_ratio_index** to **22**  

- Use Select as hotkey (for some reason, sometime EmulationStation is not able to do it on its own) :  
`sudo nano /opt/retropie/configs/all/retroarch/autoconfig/GPIO Controller 1.cfg` and set **input_exit_emulator_btn** to **NULL** and **input_enable_hotkey_btn** to **6**  


### Enable Linux style boot sequence
In EmulationStation, go to Retropie menu > **Retropie setup** > **Configuration / tools** > **Splashscreen** > **Disable splashscreen on boot**  
Then reboot.  


### Increase characters number on the screen when in terminal mode
`sudo nano /boot/config.txt` and set **framebuffer_width** to **420** and **framebuffer_height** to **300**.  
`sudo nano /etc/default/console-setup` and set **FONTFACE** to **"Terminus"** and **FONTSIZE** to **"6x12"**.  

Once done, restart the system with `sudo reboot`  


### Battery daemon
This daemon is used to monitor battery voltage and SOC.  

`cd /home/pi/NNS`  
`git clone https://github.com/porcinus/FreeplayBatteryDaemon`  
`cd FreeplayBatteryDaemon`  
`sudo apt-get update`  
`sudo apt-get install libi2c-dev`  
`mv nns-freeplay-battery-daemon.service nns-freeplay-battery-daemon.service.bak`  
`mv nns-freeplay-battery-daemon-max17048.service nns-freeplay-battery-daemon.service`  
`nano nns-freeplay-battery-daemon.service` and add after '**-i2caddr 0x36**' : ` -rsocextend 90,0,100,20`  
`chmod 755 install.sh`  
`sudo ./install.sh`  

A little explain about '**rsocextend**' line, the IC that provide data about battery voltage and remaining percentage is very conservative and consider **0%** remain when battery goes around **3.5v** where it should be **3.1-3.2v**.  
So this line is used to limit this problem a little bit, extending reported percentage for **90-0%** to **100-20%**. under **20%**, the reported data are to be considered as unreliable.  


### Overlay daemon
This daemon allow you to display a overlay with informations like battery, CPU, Bluetooth, WiFi and time.  

`cd /home/pi/NNS`  
`git clone https://github.com/porcinus/FreeplayInfo2Overlay`  
`cd FreeplayInfo2Overlay`  
`sudo apt-get install libgd-dev zlib1g-dev libfreetype6-dev libpng-dev libjpeg-dev wiringpi`  
`chmod 755 compile.sh`  
`sudo ./compile.sh`  

If you get error when running `compile.sh`, try to restart the system.  

`cd service`  
`nano info2overlayzero.sh` and replace '**-pin 41**' by '**-pin 7**', '**-lowbatpin 7**' by '**-lowbatpin 4**', add '** -alsavolume 1**', add '** -alsaname "PCM"**'.  

`sudo cp info2overlayzero.service /lib/systemd/system/info2overlayzero.service`  
`sudo systemctl enable info2overlayzero.service`  
`sudo systemctl start info2overlayzero.service`  


### Some EmulationStation tweaks
- Change EmulationStation power saving mode :  
If you want to save a bit of battery life, in EmulationStation, go to Settings menu > **Other Settings** > **Power Saving Modes** and select **Default**.  

- Disable "Quit EmulationStation" in "Quit" menu :  
`sudo nano /opt/retropie/configs/all/autostart.sh` then add before very last '**#auto**' : '` --no-exit `'  


### Add "Media" system to EmulationStation
Please note that updating EmulationStation could remove this entry.  
Audio or video files will have to be places into `/home/pi/RetroPie/media` or on USB dongle (you will need to restart EmulationStation to see the files).  
If you want to add a image to the system, you will have to edit the theme you are using.  

`cd /home/pi/RetroPie`  
`mkdir media`  
`cd media`  
`ln -s /media/usb +USB`  

Then `sudo nano /etc/emulationstation/es_systems.cfg` and add before **`</systemList>`**:  
```
  <system>
    <name>media</name>
    <fullname>Media</fullname>
    <path>/home/pi/RetroPie/media</path>
    <extension>.avi .mp4 .mkv .mov .wmv .flv .mp3 .ogg .wav .wma .aac .flac .m3u .pls .AVI .MP4 .MKV .MOV .WMV .FLV .MP3 .OGG .WAV .WMA .AAC .FLAC .M3U .PLS</extension>
    <command>/opt/retropie/supplementary/runcommand/joy2key.py /dev/input/js0 kcub1 kcuf1 0x6B 0x6A 0x70 0x71 & omxplayer -b %ROM% -o alsa --live ; killall joy2key.py</command>
    <platform>media</platform>
    <theme>media</theme>
  </system>
```

Once done, restart the system with `sudo reboot`  

About the controls:  
- Seek : **Dpad Left/Right**  
- Switch stream : **Dpad Up/Down**  
- Pause : **A**  
- Exit : **B**  


### My custom theme for EmulationStation
![diode](https://raw.githubusercontent.com/porcinus/es-theme-nns/master/screenshot-01.png)  
This theme is mostly based on [Pixel](https://github.com/RetroPie/es-theme-pixel), [Carbon](https://github.com/RetroPie/es-theme-carbon) and [GBZ35-Dark](https://github.com/rxbrad/es-theme-gbz35-dark).  
Not design to work in Kiosk or Kid UI mode.  

`cd /etc/emulationstation/themes`  
`sudo git clone https://github.com/porcinus/es-theme-nns`  
`sudo mv es-theme-nns nns`  

Then, you will have to restart EmulationStation, go to Settings menu > **UI Settings** > **Theme Set** and choise **NNS**.  

