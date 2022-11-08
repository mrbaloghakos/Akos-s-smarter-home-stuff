
# A better speaker

  

This project is about giving your regular speaker system some features, to make the most of it.

## With this project your speaker will have the following features

- Spotify playback thanks to [librespot](https://github.com/librespot-org/librespot)

- Stream your Linux based computer's audio to the speaker in real time via [PulseAudio's tunnel-sink module](https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/Modules/#module-tunnel-sinksource)

- Keep the speaker awake using [SoX](https://sox.sourceforge.net/)

- *Play notification sounds on the speaker based on custom actions (coming soon)*

## Contents:
- [Hardware Requirements](https://github.com/mrbaloghakos/Akos-s-smarter-home-stuff/tree/main/a-better-speaker#hardware-requirements)
- [Initial setup steps](https://github.com/mrbaloghakos/Akos-s-smarter-home-stuff/tree/main/a-better-speaker#initial-setup-steps)
- [Setting up librespot (Spotify Connect client)](https://github.com/mrbaloghakos/Akos-s-smarter-home-stuff/tree/main/a-better-speaker#setting-up-librespot-spotify-connect-client)
- [Setting up PulseAudio as network sink (Linux to Linux only)](https://github.com/mrbaloghakos/Akos-s-smarter-home-stuff/tree/main/a-better-speaker#setting-up-pulseaudio-as-network-sink)


## Hardware Requirements:

-  **Speaker**

    - In this project I'm using an old 2.1 Samsung Soundbar speaker. It has a HDMI input, and it supports [HDMI-CEC](https://en.wikipedia.org/wiki/Consumer_Electronics_Control).
-  **Raspberry Pi**
   - This will be the brain of the speaker. If you have any other Linux based device, feel free to use it instead. As long as it has some sort of audio output that is compatible with your speaker, it should work.
-  **Wired network for the Pi**
   - You should not use Wifi to avoid connection issues, and maintain stability.
## Initial setup steps

1. Connect the Pi to the Speaker with the audio interface of your choice. Here I will use HDMI.

2. Startup The Pi with Raspberry Pi OS Lite, get it [here](https://www.raspberrypi.com/software/operating-systems/). (You can use other Raspbian variants, but in this project the GUI is not needed, so Lite is enough)

3. Set your default sink:
   1. First list your sinks to see what you have: `pactl list sinks short`
    in my case I have two sinks:
        ```
        0 alsa_output.platform-bcm2835_audio.analog-stereo module-alsa-card.c s16le 2ch 44100Hz IDLE
        1 alsa_output.platform-3f902000.hdmi.hdmi-stereo module-alsa-card.c s16le 2ch 48000Hz RUNNING
        ```
        I will use the HDMI, so I will note down the ID of that sink. The ID is in the first column, in my case it's `1`.
    
   2. Set the default sink to the previously noted number:
    Open the `/etc/pulse/default.pa` with your favourite editor, add the following line at the bottom: `set-default-sink 1` make sure that the `set-default-sink` is specified only once in the file and replace `1` with your own sink's ID.
4. Install and set up Sox (optional)
    If your speaker turns off automatically when not detecting input, SoX can help you keep the speaker alive. It will continuously play a silent, unnoticeable sound, so the speaker will stay awake all the time.
    1. Install SoX: `sudo apt install sox`
    2. Create a Systemd service for SoX:
      Create the `~/.config/systemd/user/speaker-keepalive.service` file with the following contents:
        ```[Unit]
        Description=Keep the Speaker Alive Using sox
        
        [Service]
        ExecStart=/usr/bin/play -n -c2 synth sin gain -100
  
        [Install]
        WantedBy=default.target
        ``` 
        Load the new service with `systemctl --user daemon-reload`
        Start it with `systemctl --user start speaker-keepalive`
        Start it automatically on boot: `systemctl --user enable speaker-keepalive`
        Check if it's running: `systemctl --user status speaker-keepalive`, you should see `Active: active (running)`
## Setting up librespot (Spotify Connect client)
For this you will need a SpotifyPremium account.
  1. Install librespot according to it's [docs](https://github.com/librespot-org/librespot#quick-start).
  2. Create a Systemd service for librespot:
      1. Create the following file `~/.config/systemd/user/librespot.service` with the following content 
          ```
          [Unit]
          Description=My librespot
          After=network.target
          StartLimitIntervalSec=1
          
          [Service]
          Type=simple
          Restart=always
          RestartSec=1
          ExecStart=librespot --name 'Livingroom Big Speaker' --disable-audio-cache --bitrate 160 --initial-volume 30
          
          [Install]
          WantedBy=default.target
            ```
          Feel free to modify the options of librespot according to their [docs](https://github.com/librespot-org/librespot/wiki/Options), I find it to work best with the options above.
          
      2. Reload, enable and start the new service: `systemctl --user daemon-reload && systemctl --user enable librespot && systemctl --user start librespot`

          At this point you should be able to connect to the speaker using the Spotify app. You should find the device as `Livingroom Big Speaker` 
## Setting up PulseAudio as network sink (Linux to Linux only)
With this you will be able to stream your Linux based machine's audio to the speaker over the network. This is useful if you have sound source other than Spotify, that you would like to listen to on the speaker. You could also use Bluetooth if your speaker supports it, but then you would have to change source on the speaker, adding a few steps to the process. 
When using PulseAudio's network streaming, you will have another playback device alongside your built in speakers, and you can simply switch between the built in speakers and the remote speaker in the settings. 
 ### On the Pi
  1. Open `/etc/pulse/default.pa` with your favourite editor, and uncomment/add the following lines:
        ```
        load-module module-native-protocol-tcp auth-anonymous=1
        load-module module-native-protocol-tcp auth-ip-acl=127.0.0.1;192.168.1.0/32
        load-module module-zeroconf-publish
        ```
      Make sure you enter your network's correct IP address at the end of the second line.

2. Restart the Pi with `sudo reboot` 
 ### On your computer (or the computer that will stream to the Pi)
We will create a Systemd service that will connect to the Pi's PulseAudio sink on boot.
1. Create a script that will connect your computer to the Pi, place it where you like it. I put mine in `/opt/connect-pulseaudio-to-pi.sh`.
      Contents should be (don't forget to input your Pi's IP):
      ```
      #!/bin/sh
      pactl list modules | grep module-tunnel-sink || pactl load-module module-tunnel-sink "server=192.168.1.2 sink_name=Big-Speaker"
      ```
      What it does is it checks if there is already a `module-tunnel-sink` loaded, and if so, then do nothing. If not, then load it. 
      Don't forget to `sudo chmod +x /opt/connect-pulseaudio-to-pi.sh`

2. Create a Systemd service called `connect-pulseaudio-to-pi` that will run the above script during every boot
       Create the following file : `~/.config/systemd/user/connect-pulseaudio-to-pi.service` with the following contents:
      ```
       [Unit]
       Description=Stream local audio to the Pi's Pulseaudio receiver
       After= network.target sound.target
       
       [Service]
       Type=oneshot
       ExecStart=/opt/connect-pulseaudio-to-pi.sh
       RemainAfterExit=yes
       
       [Install]
       WantedBy=default.target
       ```
  3. Do `systemctl --user daemon-reload` then `systemctl --user enable connect-pulseaudio-to-pi.service`. 
      **Stop every audio playback on your computer!** Really. 
      Stop every video and music playback, because in the next step you will connect to the big speaker which might blast your audio on max volume the moment you press return.
      Now start the service manually to connect to the Pi: `systemctl --user start connect-pulseaudio-to-pi.service`. 
      Now go to your computer's sound settings, and select the newly added sink if it isn't selected already. It should be called something like `Tunnel to 192.168.1.2`
      Now every sound on your computer should be played on the newly added big speaker. You can control the volume by simply controlling your computer's volume as you normally would with the sliders.