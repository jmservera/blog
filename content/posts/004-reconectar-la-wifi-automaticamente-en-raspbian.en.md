+++
title =  "How to automatically reconnect to the WiFi in Raspbian"
date = 2020-11-02T00:05:16+01:00
tags = ["IoT","Raspberry Pi","Raspbian","Tips & Tricks"]
featured_image = "https://live.staticflickr.com/3706/19346279608_9805ba6259_b.jpg"
description = ""
draft = false
+++

At home, we have been connecting the garden, lights, doors, the TV, air conditioner, and even Higgs, our pet rabbit, but this last thing will be another post. We have a large variety of devices and sensors, with several Raspberry Pi 2 among them, connected to the WiFi network. Many times we have had to manually reboot the Raspis after a network connection outage because it looks like dhcpcd does not manage this situation.

After looking for a solution and finding some more or less complex solutions, my favorite one, for its simplicity, is this proposal by [Alex Bain](http://alexba.in/blog/2015/01/14/automatically-reconnecting-wifi-on-a-raspberrypi/). So, I have adapted it to work with the distro I'm currently using, which is Raspbian Buster.

You will only need to create a script named */usr/local/bin/wifi_rebooter.sh*, with a similar code to the one below, you may need to change the ping address and the interface name, which is typically called *wlan0*:

```bash
#!/bin/bash

# The IP for the server you wish to ping (For example, the router IP address)
SERVER=192.168.1.1

# Only send two pings, sending output to /dev/null
ping -c2 ${SERVER} > /dev/null

# If the return code from ping ($?) is not 0 (meaning there was an error)
if [ $? != 0 ]
then
    # Restart the wireless interface
    ip link set wlan0 down
    ip link set wlan0 up
fi
```

Then you give the script run permission:

```bash
sudo chmod +x /usr/local/bin/wifi_rebooter.sh
```

And you add a 5 minutes interval to run it in */etc/crontab*, so it will restart the network interface whenever the connection is lost:

```bash
*/5 *   * * *   root    /usr/local/bin/wifi_rebooter.sh
```

Now I no longer need to manually restart all our Raspberry Pi devices every time the connection has been broken for some time.

Makers gonna make.