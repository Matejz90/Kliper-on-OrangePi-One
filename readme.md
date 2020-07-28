sudo apt-get install dkms  
git clone https://github.com/pvaret/rtl8192cu-fixes.git  
sudo dkms add ./rtl8192cu-fixes  
sudo dkms install 8192cu/1.11  
sudo depmod -a  
sudo cp ./rtl8192cu-fixes/blacklist-native-rtl8192.conf /etc/modprobe.d/  
sudo reboot  
  
lsmod
```
Module                  Size  Used by
8188eu               1011712  0
cfg80211              425984  1 8188eu
snd_soc_hdmi_codec     16384  1
rfkill                 20480  3 cfg80211
evdev                  20480  2
rc_cec                 16384  0
sun4i_gpadc_iio        16384  0
lima                   36864  0
dw_hdmi_i2s_audio      16384  0
dw_hdmi_cec            16384  0
gpu_sched              24576  1 lima
industrialio           49152  1 sun4i_gpadc_iio
sun8i_codec_analog     24576  0
sun8i_adda_pr_regmap    16384  1 sun8i_codec_analog
sun8i_thermal          16384  0
sun4i_i2s              24576  2
snd_soc_simple_card    16384  0
snd_soc_simple_card_utils    16384  1 snd_soc_simple_card
sunxi_cedrus           32768  0
snd_soc_core          118784  5 sun4i_i2s,sun8i_codec_analog,snd_soc_hdmi_codec,snd_soc_simple_card_utils,snd_soc_simple_card
snd_pcm_dmaengine      16384  1 snd_soc_core
snd_pcm                65536  4 sun4i_i2s,snd_pcm_dmaengine,snd_soc_hdmi_codec,snd_soc_core
v4l2_mem2mem           20480  1 sunxi_cedrus
snd_timer              24576  1 snd_pcm
snd                    45056  4 snd_soc_hdmi_codec,snd_timer,snd_soc_core,snd_pcm
soundcore              16384  1 snd
gpio_keys              20480  0
uio_pdrv_genirq        16384  0
cpufreq_dt             16384  0
uio                    16384  1 uio_pdrv_genirq
zram                   24576  2
sch_fq_codel           20480  6
ip_tables              24576  0
x_tables               20480  1 ip_tables
```
ifconfig
```
eth0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether 02:81:32:76:01:f5  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 33

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlx0013ef803db5: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.2.204  netmask 255.255.255.0  broadcast 192.168.2.255
        inet6 fe80::eafa:dd90:c7b3:a2ad  prefixlen 64  scopeid 0x20<link>
        ether 00:13:ef:80:3d:b5  txqueuelen 1000  (Ethernet)
        RX packets 663  bytes 195878 (195.8 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 165  bytes 25133 (25.1 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
iwconfig
```
eth0      no wireless extensions.

lo        no wireless extensions.

wlx0013ef803db5  IEEE 802.11bgn  ESSID:"dhysxsxhfs"  Nickname:"<WIFI@REALTEK>"
          Mode:Managed  Frequency:2.462 GHz  Access Point: B0:BE:76:59:B7:F7
          Bit Rate:72.2 Mb/s   Sensitivity:0/0
          Retry:off   RTS thr:off   Fragment thr:off
          Power Management:off
          Link Quality=97/100  Signal level=43/100  Noise level=0/100
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:0  Invalid misc:0   Missed beacon:0
```
sudo armbian-config -> network -> WIFI
```sudo apt-get update```  
```sudo apt-get upgrade```  

Install required supporting software:  
  
```sudo apt-get install git python-pip python-dev python-setuptools psmisc virtualenv```  

Switch to home directory  
  
```cd ~```  
  
Download and install py-serial:  
```  
wget https://pypi.python.org/packages/source/p/pyserial/pyserial-2.7.tar.gz  
tar -zxf pyserial-2.7.tar.gz  
cd pyserial-2.7  
sudo python setup.py install  
cd ~  
pip install wheel  
python setup.py bdist_wheel  
```
The generic setup instructions boil down to  
  
Installing Python including pip and virtualenv. Please note: While OctoPrint itself supports running under Python 3.7+ starting with version 1.4.0, many of the available plugins still are Python 2 only. If you want to make use of plugins from the plugin repository, you should for now still install OctoPrint under Python 2.7. Note that migrating to Python 3 at a later date is easily done.  
Creating a virtual environment somewhere: ```sudo virtualenv OctoPrint```  
Installing OctoPrint into that virtual environment: ```sudo OctoPrint/bin/pip install OctoPrint```  
OctoPrint may then be started through ```./OctoPrint/bin/octoprint serve``` or with an absolute path ```/path/to/OctoPrint/bin/octoprint serve```  

Automatic start up  
Download the init script files from OctoPrint's repository, move them to their respective folders and make the init script executable:  
  
```wget https://github.com/foosel/OctoPrint/raw/master/scripts/octoprint.service && sudo mv octoprint.service /etc/systemd/system/octoprint.service```  
Adjust the paths to your octoprint binary in /etc/systemd/system/octoprint.service. If you set it up in a virtualenv as described above make sure your /etc/systemd/system/octoprint.service looks like this:  
```
GNU nano 2.9.3                      /etc/systemd/system/octoprint.service

[Unit]
Description=OctoPrint
After=network.target

[Service]
User=pi
ExecStart=/OctoPrint/bin/octoprint serve
Restart=always
Nice=-2

[Install]
WantedBy=default.target
```  
Then add the script to autostart using sudo systemctl enable octoprint.service.  
  
This will also allow you to start/stop/restart the OctoPrint daemon via  
  
```sudo service octoprint {start|stop|restart}```  
```systemctl status octoprint.service```  

##Make everything accessible on port 80  
Setup on Raspbian is as follows:  

```sudo apt install haproxy```  
I'm using the following configuration in /etc/haproxy/haproxy.cfg  
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

Add to /etc/default/haproxy  
```
ENABLED=1
```  
##Support restart/shutdown through OctoPrint's system menu  
In Settings > Commands, configure the following commands:  

Restart OctoPrint: sudo service octoprint restart  
Restart system: sudo shutdown -r now  
Shutdown system: sudo shutdown -h now  

:spiral_notepad: Note  
  
If you disabled Raspbian's default behaviour of allowing the pi user passwordless sudo for every command, you'll need to explicitly allow the pi user passwordless sudo access to the /sbin/shutdown program for the above to work. You'll have to add a sudo rule by creating a file /etc/sudoers.d/octoprint-shutdown (as root) with the following contents:  
  
pi ALL=NOPASSWD: /sbin/shutdown  
