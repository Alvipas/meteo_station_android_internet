# Tutorial: WiFi weather station + weewx + android phone internet access
Tutorial to configure the data acquisition in a weather station with Wi-Fi connection (Froggit WH3000), storing the data locally and using the Wunderground service. An android phone is used as an internet access point.

>My motivation for this project was to capture weather data in my village in Teruel, while having the data available online in Wunderground and keep a local record or raw metered data. I wanted to repurpose some of hardware that I had available at home: an old (but functional) internet router, an old (but functional) mobile phone and an old (but functional) Raspberry Pi. My first setup using the WiFi router as a gateway poved to be unreliable after a few months, so I decided to buy an usb WiFi adapter for the Raspberry. 

>Weewx helped me manage my prepaid SIM card data usage by setting the frequency to send data to Wunderground. In addition, it allowed to fill in data gaps during periods without internet signal at the location of the weather station.

>I am not a Linux or network expert, so I have not pretended to give a detailed explanation of all systems. My hope is that my experience will help others with similar problems when installing weather stations in remote locations.

<img src="https://github.com/Alvipas/remote_meteo_station/blob/main/set_up.jpeg?raw=true" width="500">

<img src="https://github.com/Alvipas/remote_meteo_station/blob/main/diagram.png?raw=true" width="500">


## 1. Install Raspbian
Install Raspbian in the Raspberri Pi, obtain access to the prompt and ensure to have internet connection. Then update and upgrade Raspbian.

> I created an empty file "ssh" in the root directory of the SD to activate SSH access. Then, I connected via ethernet from windows laptop using Putty. I shared internet connection in my laptop´s WiFi adapter. Example [here](http://carbonstone.blogspot.com/2014/02/connecting-to-pi-from-laptops-ethernet.html) .
> 

## 2. Configure network interfaces
The network configuration is done using [systemd networkd](https://wiki.archlinux.org/index.php/systemd-networkd), which is a powerful yet simple and intuitive network manager. 

For using systemd networkd, other network servies must be disabled. (See point 2.1 before doing this). At this point in time, dhcpcd is the default service in Raspbian:
> systemctl stop dhcpcd && systemctl disable dhcpcd

Then, enable and activate systemd-networkd:
> systemctl enable systemd-networkd.service && systemctl start systemd-networkd.service
> systemctl enable systemd-resolved.service && systemctl start systemd-resolved.service

Each network interface (ehernet, usb tethering, usb wifi hotspot) is defined in this directory /etc/systemd/network/ with , with file  extension .network. I got inspired by this [blog](https://blog.michael.franzl.name/2017/02/28/raspberry-pi-gateway-mobile-internet/)

### 2.1 Ehernet 
```shell
> sudo nano /etc/systemd/network/01_eth.network
```
    [Match]
    Name=eth0
    [Network]
    Address=192.168.137.50/24  
    [Route]
    Gateway=192.168.137.1 
    Metric=500
    
```shell
> sudo systemctl restart systemd-networkd
```

>When sharing the Internet from the WiFi adapter in Windows, the ethernet is assigned a static IP 192.168.137.50. Please note that this interface is only used to configure the Raspberry with the laptop. Perhaps, it is better to perform this step before disabling the dhcpd service, as the Raspberry may not accept the Putty connection (I don´t remember on detail how I did this).
> Metric = 500 is used to decide the priority of using the gateway for the Internet connection. If it is the lowest among the interfaces with Internet access, this will be the default path used by Raspberry.
> 
### 2.2 USB mobile tethering
```shell
> sudo nano /etc/systemd/network/02_mobile.network
```
	[Match] 
	Name=usb0 
	[Network] 
	DHCP=yes 
	[DHPC]
	RouteMetric=10
```shell
> sudo systemctl restart systemd-networkd
```
For this interface to work, you will need to connect the mobile device to the USB port of the Raspberrypi and enable USB tethering.

### 2.3 WiFi hotspot
There are two main steps to be taken: configuring the wireless access point (WAP) and configuring the network interface.
> For this project, I used a Ralink RT5370 usb dongle, as my raspberry pi had no WiFi card. You can check if the Raspberry is detecting the usb dongle running > lsusb.

#### 2.3.1 WAP
The wifi hotspot is configured with hostapd. You can find a more detailed example [here](http://va.nce.me/Creating_an%20Access_Point.html)
```shell
sudo apt install hostapd
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
```
Then, configure the access point: 
```shell
sudo nano  /etc/hostapd/hostapd.conf
```
    # Basics
    interface=wlan0
    driver=nl80211
    ssid="YourSSID"
    hw_mode=g
    channel=1

    # Security Settings - wpa2 only
    macaddr_acl=0
    auth_algs=1
    ignore_broadcast_ssid=0
    wpa=2
    wpa_passphrase= "YourPassword"
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=TKIP
    rsn_pairwise=CCMP
Then, point hostapd to use the conf file and enable it:
```shell
> sudo hostapd /etc/hostapd.conf
> sudo systemctl enable hostapd
```
#### 2.3.2 Wlan0 interface
```shell
> sudo nano /etc/systemd/network/03_wlan_ap.network
```
    [Match]
    Name=wlan0
    [Network]
    DHCPServer=true
    Address=192.168.4.1/24
    [DHCPServer]
    PoolOffset=100
    PoolSize=20
    LinkLocalAddressing=yes
    MulticastDNS=yes
```shell
> sudo systemctl restart systemd-networkd
```
>If you want to provide Internet access through the WiFi access point from other interfaces, you can include IPMasquerade = True in [Network], which allows IP forwarding and creates the basic routing rules for you.

At this point, you can check that everything is configured and looking as expected:
```shell
> networkctl
> route -n
```
### 2.4 Problems with DNS name resolution 
> I had some problems with internet name resolution, after switching to systemd-networkd, although probably I did something I shouldn´t. My two network interfaces had internet access (> ping 8.8.8.8), but DNS name resolution was not working (ping www.google.com). I sorted the problem doing [this](https://askubuntu.com/questions/1098414/18-04-unable-to-connect-to-server-due-to-temporary-failure-in-name-resolution). I had to create manually a simbolic link:

```shell
> sudo rm -f /etc/resolv.conf
> sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
> reboot
```
## 3. Connect weather station using WSview android app
Configure the weather station connection to wlan0 access point created in previous step. Follow the steps detailed in WSview app, and choose "Customized" server, with the IP of the wlan0 (192.168.4.1) port 8080. Then, save configuration and check that the station console shows the WiFi icon. 

Then check that the data is arriving at the raspberry using tcpdump.
```shell
> sudo apt install tcpdump 
> tcpdump -i wlan0 src 193.168.4.108
```
where 193.168.4.108 is the IP address of the weather station. This can be obtained in the WSview app when connected to the same WiFi network.

You should see something like this:

    21:56:19.858669 IP 192.168.4.108.32846 > 192.168.137.50.http-alt: Flags [P.], seq 9050113:9050566, ack 1662206314, win 5840, length 453: HTTP: GET ID=++++++&PASSWORD=+++++++&indoortempf=67.3&tempf=67.3&dewptf=52.0&windchillf=67.3&indoorhumidity=54&humidity=58&windspeedmph=0.0&windgustmph=0.0&winddir=7&absbaromin=27.467&baromin=29.794&rainin=0.000&dailyrainin=0.000&weeklyrainin=0.091&monthlyrainin=0.130&solarradiation=0.00&UV=0&dateutc=2021-02-22%2021:56:19&softwaretype=EasyWeatherV1.4.4&action=updateraw&realtime=1&rtfreq=5 HTTP/1.0
## 4. Configure Weewx and Weewx-interceptor
Install weewx for [raspbian](http://weewx.com/docs/debian.html). 
Install [weewx-interceptor](https://github.com/matthewwall/weewx-interceptor)

Change configuration file:
```shell
sudo nano /etc/weewx/weewx.conf
```
    # To check that traffic is being parsed in syslog
    debug=1 
    [Station]
    ...
    station_type = Interceptor
    [Interceptor]
    ...
    driver = user.interceptor
    device_type = wu-client
    port = 8080
    iface = wlan0
    ...
    [[Wunderground]]
    # Very conveniently, in weewx you can choose the posting interval to Wunderground
    post_interval=600
Check that data is being parsed
```shell
> sudo /etc/init.d/weewx restart
> sudo tail -f /var/log/syslog
```
You should see something like:

    Feb 22 21:57:55 raspberrypi weewx[1057] DEBUG user.interceptor: raw packet: {'dateTime': 1614031075, 'usUnits': 1, 'rain_total': 0.0, 'temperature_in': 67.3, 'temperature_out': 67.3, 'dewpoint': 52.0, 'windchill': 67.3, 'humidity_in': 53.0, 'humidity_out': 58.0, 'wind_speed': 0.0, 'wind_gust': 0.0, 'wind_dir': 7.0, 'barometer': 29.788, 'solar_radiation': 0.0, 'uv': 0.0, 'rain': 0.0}
    Feb 22 21:57:55 raspberrypi weewx[1057] DEBUG user.interceptor: mapped packet: {'dateTime': 1614031075, 'usUnits': 1, 'barometer': 29.788, 'outHumidity': 58.0, 'inHumidity': 53.0, 'outTemp': 67.3, 'inTemp': 67.3, 'windSpeed': 0.0, 'windGust': 0.0, 'windDir': 7.0, 'radiation': 0.0, 'dewpoint': 52.0, 'windchill': 67.3, 'rain': 0.0, 'UV': 0.0}

> I chose a post_interval of 600 seconds. This resulted in a daily data consumption of approx. 1.2MB per day, which allowed me to use a low cost prepaid SIM card.

### 4.1 Wunderground protocol parsing problems
> After checking /var/log/syslog, I noticed that the data packages were arriving, but they were not being parsed properly. I found the solution in this weewx-interceptor [issue](https://github.com/matthewwall/weewx-interceptor/issues/82) 

### 4.2 Wunderfixer for intermittent internet signal
If your station is located in a region with intermittent mobile signal, you might want to configure [wunderfixer](http://www.weewx.com/wunderfixer/) to run once per day in order to fill the missing gaps in the previous day.

```shell
> crontab -e
```
	00 12 * * * wunderfixer --verbose --upload-only=600 --date=$(date  --date="yesterday" +"\%Y-\%m-\%d") --log weewx  > /dev/null 2>&1
  
## 5. Configure Android device to enable USB tethering automatically
So far, the system should be robust to log all the data in the local database, and all the systems should restart automatically after a power outage. However, the phone´s USB tethering has to be enabled manually each time the system is restored. Depending on the location of the weather station, power outages can be relatively frequent, in which case the android device can be programmed to enable USB tethering by default.
First the phone has to be rooted. I used Kingoroot. Then, in the Google market there are some applications that allow to define decision rules. I used [Automate](https://play.google.com/store/apps/details?id=com.llamalab.automate), and then created this rule:

<img src="https://github.com/Alvipas/remote_meteo_station/blob/main/automate.jpeg?raw=true" width="500">



    
    




