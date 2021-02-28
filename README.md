# WiFi weather station + mobile phone internet access tutorial
Tutorial to configure data acquisition in a weather station with Wi-Fi connection (Froggit WH3000) with connection to Wunderground, using weewx-interceptor and an old mobile as an internet access point.

<img src="https://github.com/Alvipas/remote_meteo_station/blob/main/set_up.jpeg?raw=true" width="500">

## 1. Install Raspbian
Install Raspbian in the Raspberri Pi, obtain access to the prompt and ensure to have internet connection. 

> I created an empty file "ssh" in the root directory of the SD to activate SSH access. Then, I connected via ethernet from windows laptop using Putty. I shared internet connection in my laptop WiFi adapter. Example [here](http://carbonstone.blogspot.com/2014/02/connecting-to-pi-from-laptops-ethernet.html) .
