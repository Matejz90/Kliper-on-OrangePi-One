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
sudo apt install python-pip python-dev python-setuptools python-virtualenv git libyaml-dev build-essential
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
ExecStart=/home/pi/OctoPrint/bin/octoprint serve
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
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon

        # Default SSL material locations
        ca-base /etc/ssl/certs
        crt-base /etc/ssl/private

        # Default ciphers to use on SSL-enabled listening sockets.
        # For more information, see ciphers(1SSL). This list is from:
        #  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
        # An alternative list with additional directives can be obtained from
        #  https://mozilla.github.io/server-side-tls/ssl-config-generator/?server=haproxy
        ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!MD5:!DSS
        ssl-default-bind-options no-sslv3

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http
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
##Support restart/shutdown through OctoPrint's system menu  
In Settings > Commands, configure the following commands:  

Restart OctoPrint: sudo service octoprint restart  
Restart system: sudo shutdown -r now  
Shutdown system: sudo shutdown -h now  
