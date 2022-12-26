## Raspberry Pi as an Automatic Wake on Lan Server

I have a recurring issue where power outages result in my NUC (used as an esxi host) powering off.  This devices runs a Home Assistant VM so it is an important part of keeping automations for lights, door locks, thermostat schedules, etc. running. Recently, I was away for a few days, and a power outage occurred and the NUC lost power.  Nothing catastrophic happened, but it is always nice to be able to check in on things at home particularly in the winter. 

So with this post, I will share what I put together to mitigate issues like this in the future and hopefully help someone else solve a similar issue.

Note, this solution was all implemented on a Raspberry PI running Raspbian GNU/Linux 10 (buster), so your milage may vary if attempting to implement on different version of Linux.

---

## Installing WOL Software

First you will need to install a tool to send the Wake On Lan packet. You can read more on WOL [here](https://en.wikipedia.org/wiki/Wake-on-LAN). I have selected [etherwake](https://manpages.debian.org/buster/etherwake/etherwake.8.en.html) in this scenario.  

```console
foo@bar:~ $ sudo apt-get install etherwake
```

After installation you can run the following to get a full outline of etherwake's options

```console
foo@bar:~ $ etherwake -u
```

For etherwake the only required parameter is the MAC address of the device you want to power on.  You can also specify which adapter to utilize and password if your device requires it. 

```console
foo@bar:~ $ etherwake 66-3F-E8-D9-56-CE
```

At this point you can test your scenario by powering off your device and issuing an etherwake command with your MAC address.

---

## Shell Script to Send WOL Command if Device is Offline  

Next, I wanted to develop a script to check if a device is on the network (pingable), and if not send a Wake on Lan packet.  I settled on the following:

```console
#!/bin/bash
# Ping host if un responsive then send wol packet
HOST=XXX.XXX.XXX.XXX
MAC=xx:xx:xx:xx:xx:xx
NAME=Name
ping -c 4 -i 1 $HOST 0>/dev/null
OFFLINE=$?
if [ $OFFLINE -eq 1 ]
then
  echo "Sending Wake On Lan packet to $NAME"
  sudo etherwake -i eth0 $MAC
else
  echo "$NAME Online"
```

This script will ping the designated host IP address 4 times if the device is unresponsive then a WOL packet will be sent to the designated MAC address otherwise the device will be reported as online. 

There are three variables that must be set specific to your use case. 

```console
HOST = 192.168.1.15  <- IP Address to check with ping command
MAC = 66-3F-E8-D9-56-CE  <- MAC address of device to wake
NAME = NUC <- Friendly name of device to use in echo output
```

Once your script is configured save it with a .sh extension and run it with:
```console
foo@bar:~ $ bash wol/wol-nuc.sh
```
In my example the file name is wol-nuc.sh located in a folder wol within my home directory.  The location and path will be important in the next section. I recommend you test at this point with your device and ensure it can detect the host as available when powered on and it can wake it when powered off.  

---
## Configuring Cron Job to Execute Script
Cron is a utility that let's you specify repeating tasks at specific times or intervals.  In this scenario we are going to use to periodically run our wake on lan script from above.

Open the crontab file in the text editor of your choice in my example I am using nano.
```console
foo@bar:~ $ sudo nano /etc/crontab
```
![Cron Definition Before modification](/img/raspiwol/crondefbefore.png)

The default chrontab file within my distro is commented so comprehending is not too difficult, but if you wish you can further explore how to configure cron schedules [here](https://crontab.guru/examples.html). 

In this file we need to append our new job. In my case I am going to set this script to run every 15 minutes as the root user. 
```console
*/15 *    * * *   root    /home/pi/wol/wol-nuc.sh
```
![Cron Definition After modification](/img/raspiwol/crondefafter.png)

After saving the crontab file we must set our shell script as executable.
```console
foo@bar:~ $ chmod +x /home/pi/wol/wol-nuc.sh
```

You can view the past execution of your cron tasks with the following

```console
foo@bar:~ $ grep CRON /var/log/syslog
```
You should see an output similar to the following if your script is executing successfully
```console
Dec 26 15:30:01 raspberrypi CRON[1475]: (root) CMD (   /home/pi/wol/wol-nuc.sh)
```
---
## Summary
In this short guide we:
1. Installed a Wake on Lan tool 
2. Configured a shell script to check for a device on the network and issue a Wake on Lan packet if not found
3. Configured a cron task to run our script a regular interval

This is by no means a perfect solution for all scenarios but at least it gives me a little more confidence I won't lose visibility to my Home Assistant instance when away for extended periods of time. 