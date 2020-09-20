First go ```sudo raspi-config``` and connect to WIFI and resize filesystem then go to:  
```sudo apt-get update -y && sudo apt-get upgrade -y```

LCD 4 inch Configuration  
```sudo rm -rf LCD-show
git clone https://github.com/goodtft/LCD-show.git
chmod -R 755 LCD-show
cd LCD-show/
sudo ./MPI4008-show
```
Octoprint install  
```
cd ~
sudo apt update
sudo apt install python-pip python-dev python-setuptools python-virtualenv git libyaml-dev build-essential libavcodec-dev
mkdir OctoPrint && cd OctoPrint
virtualenv venv
source venv/bin/activate
pip install pip --upgrade
pip install octoprint
sudo usermod -a -G tty pi
sudo usermod -a -G dialout pi
```
  
If you disabled Raspbian's default behaviour of allowing the pi user passwordless sudo for every command, you'll need to explicitly allow the pi user passwordless sudo access to the /sbin/shutdown program for the above to work. You'll have to add a sudo rule by creating a file ```sudo nano /etc/sudoers.d/octoprint-shutdown``` (as root) with the following contents:  
  
```pi ALL=NOPASSWD: /sbin/shutdown```  
  
Automatic start up  
Download the init script files from OctoPrint's repository, move them to their respective folders and make the init script executable:  
  
```wget https://github.com/foosel/OctoPrint/raw/master/scripts/octoprint.service && sudo mv octoprint.service /etc/systemd/system/octoprint.service```  
Adjust the paths to your octoprint binary in /etc/systemd/system/octoprint.service. If you set it up in a virtualenv as described above make sure your /etc/systemd/system/octoprint.service looks like this:  
```
[Unit]
Description=The snappy web interface for your 3D printer
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=pi
ExecStart=/home/pi/OctoPrint/venv/bin/octoprint serve
Restart=always
RestartSec=5
Nice=-2

[Install]
WantedBy=multi-user.target
```  
Then add the script to autostart using ```sudo systemctl enable octoprint.service```  
  
This will also allow you to start/stop/restart the OctoPrint daemon via  
  
```sudo service octoprint {start|stop|restart}```  
```systemctl status octoprint.service```  

##Make everything accessible on port 80  
Setup on Raspbian is as follows:  

```sudo apt install haproxy -y```  
I'm using the following configuration in ```sudo nano /etc/haproxy/haproxy.cfg```  
```
global
        maxconn 4096
        user haproxy
        group haproxy
        daemon
        log 127.0.0.1 local0 debug

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        retries 3
        option redispatch
        option http-server-close
        option forwardfor
        maxconn 2000
        timeout connect 5s
        timeout client  15min
        timeout server  15min

frontend public
        bind :::80 v4v6
        use_backend webcam if { path_beg /webcam/ }
        default_backend octoprint

backend octoprint
        reqrep ^([^\ :]*)\ /(.*)     \1\ /\2
        option forwardfor
        server octoprint1 127.0.0.1:5000

backend webcam
        reqrep ^([^\ :]*)\ /webcam/(.*)     \1\ /\2
        server webcam1  127.0.0.1:8080
```

Add to ```sudo nano /etc/default/haproxy```  
```
ENABLED=1
```  
And then
```sudo service haproxy start```


##Support restart/shutdown through OctoPrint's system menu  
In Settings > Commands, configure the following commands:  

Restart OctoPrint: sudo service octoprint restart  
Restart system: sudo shutdown -r now  
Shutdown system: sudo shutdown -h now  

Optional: Webcam
If you also want webcam and timelapse support, you'll need to download and compile MJPG-Streamer:

cd ~
sudo apt install subversion libjpeg62-turbo-dev imagemagick ffmpeg libv4l-dev cmake
git clone https://github.com/jacksonliam/mjpg-streamer.git
cd mjpg-streamer/mjpg-streamer-experimental
export LD_LIBRARY_PATH=.
make
:point_up: Heads-up

The required packages depend on the underlying version of Debian! The above is what should work on the current Debian Stretch based images of Raspbian.

For Jessie use:

sudo apt install subversion libjpeg62-turbo-dev imagemagick libav-tools libv4l-dev cmake
For Wheezy or older (you should update...) use:

sudo apt install subversion libjpeg8-dev imagemagick libav-tools libv4l-dev cmake
This should hopefully run through without any compilation errors. You should then be able to start the webcam server using:

./mjpg_streamer -i "./input_uvc.so" -o "./output_http.so"
This should give the following output:

MJPG Streamer Version: svn rev:
 i: Using V4L2 device.: /dev/video0
 i: Desired Resolution: 640 x 480
 i: Frames Per Second.: 5
 i: Format............: MJPEG
[...]
 o: www-folder-path...: disabled
 o: HTTP TCP port.....: 8080
 o: username:password.: disabled
 o: commands..........: enabled
For some webcams (including the PS3 Eye) you'll need to force the YUV mode by using the following start command:

./mjpg_streamer -i "./input_uvc.so -y" -o "./output_http.so" 
Please be aware that YUV mode will put additional strain on your Raspi's CPU which will then lower its performance, possibly up to the point of causing printing issues. If your camera requires the -y parameter to function, consider replacing it with one that doesn't.

:spiral_notepad: Note

If your webcam requires switching to YUV mode in order to work at all, it is strongly recommended to instead use a webcam that natively supports MJPG. For YUV cameras mjpg_streamer will need to transcode all data from the camera to MJPG on your Raspberry Pi, which will put a lot of strain on its CPU (YUV mode at around 30-40% vs MJPG mode at around 1-2%). This MIGHT negatively influence print quality, so better get yourself a cheap MJPG compatible webcam. See this wiki page 556 for a compatibility list and steer clear of cams that require -y to work.

:spiral_notepad: Note

If you want to use the official RaspberryPi Camera Module you need to run

./mjpg_streamer -i "./input_raspicam.so -fps 5" -o "./output_http.so" 
If you now point your browser to http://<your Raspi's IP>:8080/?action=stream, you should see a moving picture at 5fps. (If you get an error message about missing files or directories calling the output plugin with -o "./output_http.so -w ./www" should help.)

Open OctoPrint's settings dialog and under Webcam & Timelapse configured the following:

Stream URL: /webcam/?action=stream
Snapshot URL: http://127.0.0.1:8080/?action=snapshot
Path to FFMPEG: /usr/bin/ffmpeg
:point_up: Heads-up

If for whatever reason you are still using a Raspbian image based on Debian Jessie or older, "Path to FFMPEG" should instead be /usr/bin/avconv.

Restart the OctoPrint server, clear the cache on your browser and reload the OctoPrint page. You should now see the stream from the webcam in the "Control" tab, and a "Timelapse" tab with options.

If you want to be able to start and stop mjpeg-streamer from within OctoPrint, put the following in /home/pi/scripts/webcam:

#!/bin/bash
# Start / stop streamer daemon

case "$1" in
    start)
        /home/pi/scripts/webcamDaemon >/dev/null 2>&1 &
        echo "$0: started"
        ;;
    stop)
        pkill -x webcamDaemon
        pkill -x mjpg_streamer
        echo "$0: stopped"
        ;;
    *)
        echo "Usage: $0 {start|stop}" >&2
        ;;
esac
Put this in /home/pi/scripts/webcamDaemon:

#!/bin/bash

MJPGSTREAMER_HOME=/home/pi/mjpg-streamer/mjpg-streamer-experimental
MJPGSTREAMER_INPUT_USB="input_uvc.so"
MJPGSTREAMER_INPUT_RASPICAM="input_raspicam.so"

# init configuration
camera="auto"
camera_usb_options="-r 640x480 -f 10"
camera_raspi_options="-fps 10"

if [ -e "/boot/octopi.txt" ]; then
    source "/boot/octopi.txt"
fi

# runs MJPG Streamer, using the provided input plugin + configuration
function runMjpgStreamer {
    input=$1
    pushd $MJPGSTREAMER_HOME
    echo Running ./mjpg_streamer -o "output_http.so -w ./www" -i "$input"
    LD_LIBRARY_PATH=. ./mjpg_streamer -o "output_http.so -w ./www" -i "$input"
    popd
}

# starts up the RasPiCam
function startRaspi {
    logger "Starting Raspberry Pi camera"
    runMjpgStreamer "$MJPGSTREAMER_INPUT_RASPICAM $camera_raspi_options"
}

# starts up the USB webcam
function startUsb {
    logger "Starting USB webcam"
    runMjpgStreamer "$MJPGSTREAMER_INPUT_USB $camera_usb_options"
}

# we need this to prevent the later calls to vcgencmd from blocking
# I have no idea why, but that's how it is...
vcgencmd version

# echo configuration
echo camera: $camera
echo usb options: $camera_usb_options
echo raspi options: $camera_raspi_options

# keep mjpg streamer running if some camera is attached
while true; do
    if [ -e "/dev/video0" ] && { [ "$camera" = "auto" ] || [ "$camera" = "usb" ] ; }; then
        startUsb
    elif [ "`vcgencmd get_camera`" = "supported=1 detected=1" ] && { [ "$camera" = "auto" ] || [ "$camera" = "raspi" ] ; }; then
        startRaspi
    fi

    sleep 120
done
Make sure both files are executable:

chmod +x /home/pi/scripts/webcam
chmod +x /home/pi/scripts/webcamDaemon
If you want different camera options put them in /boot/octopi.txt or modify the script accordingly.

If you want autostart of the webcam you need to add the following line to /etc/rc.local (Just make sure to put it above the line that reads exit 0).

/home/pi/scripts/webcam start
If you want to be able to start and stop the webcam server through OctoPrint's system menu, add the following to config.yaml:

system:
  actions:
   - action: streamon
     command: /home/pi/scripts/webcam start
     confirm: false
     name: Start video stream
   - action: streamoff
     command: sudo /home/pi/scripts/webcam stop
     confirm: false
     name: Stop video stream
:spiral_notepad: Note

If you want to view the stream directly on your Pi, please be aware that Midori will not allow you to see the webcam picture. Chromium works although it is a bit slow, but it still might be useful for testing or aiming the camera:

sudo apt install chromium-browser
In any case this is only recommended for debugging purposes during setup, running a graphical user interface on the Pi will put a lot of unnecessary load on the CPU which might negatively influence print results.

:spiral_notepad: Note

mjpegstreamer does not allow to bind to a specific interface to limit the accessibility to localhost only. If you want your octoprint instance to be reachable from the internet you need to block access to port 8080 from all sources except localhost if you don't want the whole world to see your webcam image.

To do this simply add iptables rules like this:

sudo /sbin/iptables -A INPUT -p tcp -i wlan0 ! -s 127.0.0.1 --dport 8080 -j DROP    # for ipv4
sudo /sbin/ip6tables -A INPUT -p tcp -i wlan0 ! -s ::1 --dport 8080 -j DROP         # for ipv6
Replace the interface with eth0, if you happen to use ethernet.

To make them persistent, they need to be saved. In order to be restored at boot time, the easiest way is to install iptables-persist:

sudo apt install iptables-persistent
The only thing left to do now, is save the rules you have added:

sudo /sbin/ip6tables-save > /etc/iptables/rules.v6
sudo /sbin/iptables-save > /etc/iptables/rules.v4

Klipper install
git clone https://github.com/KevinOConnor/klipper
./klipper/scripts/install-octopi.sh

Building and flashing the micro-controller
To compile the micro-controller code, start by running these commands on the Raspberry Pi:

cd ~/klipper/
make menuconfig
Select the appropriate micro-controller and review any other options provided. Once configured, run:

make
It is necessary to determine the serial port connected to the micro-controller. For micro-controllers that connect via USB, run the following:

ls /dev/serial/by-id/*
It should report something similar to the following:

/dev/serial/by-id/usb-1a86_USB2.0-Serial-if00-port0
It’s common for each printer to have its own unique serial port name. This unique name will be used when flashing the micro-controller. It’s possible there may be multiple lines in the above output - if so, choose the line corresponding to the micro-controller (see the FAQ for more information).

For common micro-controllers, the code can be flashed with something similar to:

sudo service klipper stop
make flash FLASH_DEVICE=/dev/serial/by-id/usb-1a86_USB2.0-Serial-if00-port0
sudo service klipper start
Be sure to update the FLASH_DEVICE with the printer’s unique serial port name.

When flashing for the first time, make sure that OctoPrint is not connected directly to the printer (from the OctoPrint web page, under the “Connection” section, click “Disconnect”).

Configuring OctoPrint to use Klipper
The OctoPrint web server needs to be configured to communicate with the Klipper host software. Using a web browser, login to the OctoPrint web page and then configure the following items:

Navigate to the Settings tab (the wrench icon at the top of the page). Under “Serial Connection” in “Additional serial ports” add “/tmp/printer”. Then click “Save”.

Enter the Settings tab again and under “Serial Connection” change the “Serial Port” setting to “/tmp/printer”.

In the Settings tab, navigate to the “Behavior” sub-tab and select the “Cancel any ongoing prints but stay connected to the printer” option. Click “Save”.

From the main page, under the “Connection” section (at the top left of the page) make sure the “Serial Port” is set to “/tmp/printer” and click “Connect”. (If “/tmp/printer” is not an available selection then try reloading the page.)

Once connected, navigate to the “Terminal” tab and type “status” (without the quotes) into the command entry box and click “Send”. The terminal window will likely report there is an error opening the config file - that means OctoPrint is successfully communicating with Klipper. Proceed to the next section.
