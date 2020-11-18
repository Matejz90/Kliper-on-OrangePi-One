First go ```sudo raspi-config``` and connect to WIFI and resize filesystem then go to:  
```sudo apt-get update -y && sudo apt-get upgrade -y```

## LCD 4 inch Configuration  
```sudo rm -rf LCD-show
git clone https://github.com/goodtft/LCD-show.git
chmod -R 755 LCD-show
cd LCD-show/
sudo ./MPI4008-show
```
## Octoprint install  
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
  
## Automatic start up  
Download the init script files from OctoPrint's repository, move them to their respective folders and make the init script executable:  
  
```wget https://github.com/foosel/OctoPrint/raw/master/scripts/octoprint.service && sudo mv octoprint.service /etc/systemd/system/octoprint.service```  
Adjust the paths to your octoprint binary in ```sudo nano /etc/systemd/system/octoprint.service```.
  
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
  
```sudo service octoprint start```
```sudo service octoprint stop```
```sudo service octoprint restart```
```systemctl status octoprint.service```  

## Make everything accessible on port 80  
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


## Support restart/shutdown through OctoPrint's system menu  
In Settings > Commands, configure the following commands:  

Restart OctoPrint: ```sudo service octoprint restart```  
Restart system: ```sudo shutdown -r now```  
Shutdown system: ```sudo shutdown -h now```  

## Optional: Webcam
If you also want webcam and timelapse support, you'll need to download and compile MJPG-Streamer:  
```
cd ~
sudo apt install subversion libjpeg62-turbo-dev imagemagick ffmpeg libv4l-dev cmake
git clone https://github.com/jacksonliam/mjpg-streamer.git
cd mjpg-streamer/mjpg-streamer-experimental
export LD_LIBRARY_PATH=.
make
```  
  
This should hopefully run through without any compilation errors. You should then be able to start the webcam server using:  
  
```./mjpg_streamer -i "./input_uvc.so" -o "./output_http.so"```  
  
This should give the following output:
  
```
 MJPG Streamer Version: svn rev:
 i: Using V4L2 device.: /dev/video0
 i: Desired Resolution: 640 x 480
 i: Frames Per Second.: 5
 i: Format............: MJPEG
 o: www-folder-path...: disabled
 o: HTTP TCP port.....: 8080
 o: username:password.: disabled
 o: commands..........: enabled
 ```
   
For some webcams (including the PS3 Eye) you'll need to force the YUV mode by using the following start command:  
  
```./mjpg_streamer -i "./input_uvc.so -y" -o "./output_http.so" ```  
  
If you want to use the official RaspberryPi Camera Module you need to run  
  
```./mjpg_streamer -i "./input_raspicam.so -fps 5" -o "./output_http.so"```
  
Stream URL: ```/webcam/?action=stream```  
Snapshot URL: ```http://127.0.0.1:8080/?action=snapshot```  
Path to FFMPEG: ```/usr/bin/ffmpeg```  
  
Execute ```sudo mkdir /home/pi/scripts```  
  
If you want to be able to start and stop mjpeg-streamer from within OctoPrint, put the following in ```sudo nano /home/pi/scripts/webcam```  
  
```
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
```
  
Put this in ```sudo nano /home/pi/scripts/webcamDaemon```  
  
```
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
```  
  
Make sure both files are executable:  
  
```chmod +x /home/pi/scripts/webcam```  
```chmod +x /home/pi/scripts/webcamDaemon```  


If you want autostart of the webcam you need to add the following line to /etc/rc.local (Just make sure to put it above the line that reads exit 0).  
  
```/home/pi/scripts/webcam start```
  
If you want to be able to start and stop the webcam server through OctoPrint's system menu, add the following to config.yaml:  
  
```
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
```
