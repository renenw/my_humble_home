Notes to my future self. And to anyone following my tracks.

# Why?

Years back I rolled out some 1Wire devices to meter our electrical consumption. This guided changes to our lighting that drastically reduced our electiricity bill. It also lead to various other home automation projects: a pool topper-upper, a gate pin pad, notifications when sumps flooded, weather based garden irrigation etc. 

But, over time, power supplies failed, wires corroded, and technology moved forward. Its time to add a few cameras, and, it seems we should embrace Home Assistant.

# Goals

1. Keep the pool topped up
2. Track and report on network latency
3. Meter resources: in particular, electricity and water
4. Automate the locks
5. Manage footage from the cameras, and reconcile this to events generated by the alarm system.

# Getting Going

I bought a little Beelink computer with 2.5GB os SSD, and 16GB of RAM. I installed Ubuntu 22.10 onto it.

# Home Assistant

After watching loads of videos, I evenutally settled on installing "Home Assistant Core": Python based, and foregoing add ons. Which I suspect I want to manage manually anyway.

I followed the instructions on the Home Assistant web site.

But, its never quite that simple. You also need to setup HA as a service. I followed [these](https://community.home-assistant.io/t/autostart-using-systemd/199497) instructions:

First: `sudo nano /etc/systemd/system/home-assistant@homeassistant.service`

Paste the following into that file:
```
[Unit]
Description=Home Assistant
After=network-online.target

[Service]
Type=simple
User=%i
WorkingDirectory=/home/%i/.homeassistant
ExecStart=/srv/homeassistant/bin/hass -c "/home/%i/.homeassistant"
RestartForceExitStatus=100

[Install]
WantedBy=multi-user.target
```

Then:
```
sudo systemctl --system daemon-reload
sudo systemctl start home-assistant@homeassistant.service
sudo systemctl status home-assistant@homeassistant.service
```

Make sure that things still work by restarting.


# MQTT

## Why?

As I understand things, MQTT is a pub-sub messaging mechanism that allows Home Assistant to communicate with a range of devices. For example, [Zigbee2MQTT](https://www.zigbee2mqtt.io/) translates between Zigbeen devices and Home Assistant using MQTT.

## Installation

Installed mosquitto. I also needed to open the firewall to allow outside traffic to reach our new broker:
```
sudo ufw enable
sudo ufw allow 1883
```

I also had to tell MQTT to listen to remote connections, and to allow anonymous connections:

`sudo nano /etc/mosquitto/conf.d/mosquitto.conf`

Then, add the following two lines:
```
allow_anonymous true
listener 1883 0.0.0.0
```

(That was the only two lines in this file.)

## Testing

In one console window, run: `mosquitto_sub -v -t '#'` which echos all messages received by the broker to the console.

In another, run: `mosquitto_pub -t 'test/topic' -m 'hello'`

Even better, test from a remote machine:

This time: `mosquitto_sub -v -t '#' -h 192.168.0.245`

And then: `mosquitto_pub -t 'test/topic' -m 'hello' -h 192.168.0.245`

# Zigbee2MQTT

In our previous setup, we used 1Wire sensors and float switches to monitor sumps and the pool depth, as well as a a few thermometers around and about.

I want to migrate these to "manufactured" Zigbee devices.

It seems the standard approach to linking a Zigbee network to Home Assistant is to use [Zigbee2MQTT](https://www.zigbee2mqtt.io/). However, my Home Assistant server is in my office, miles from the "centre" of the house. As such, I'm going to plug my Zigbee [controller](https://www.se.com/eg/en/faqs/FA376743/) into a Raspberry Pi runing in the thick of things.

For a controller, I bought a Sonoff Zigbee 3.0 USB Dongle. Which seems pretty popular.

## Installation

I plugged the USB device into the Raspberry Pi, and then followed the Zigbee2MQTT default Linux instructions, [here](https://www.zigbee2mqtt.io/guide/installation/01_linux.html).

I had to use their (recommended) `by-id` mapping approach:

```
pi@relay:~ $ ls -l /dev/serial/by-id
total 0
lrwxrwxrwx 1 root root 13 Jan 29 14:24 usb-ITead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_ce715a9d239dec11975d614d73138bba-if00-port0 -> ../../ttyUSB0
```

I ran through the balance of the instructions. In the YAML file, I set the MQTT broker to the broker installed on my server, and set the:
```
server: mqtt://192.168.0.245
port: /dev/serial/by-id/usb-ITead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_ce715a9d239dec11975d614d73138bba-if00-port0
```
And, I added the recommended lines to the end of the file:
```
advanced:
    network_key: GENERATE
frontend: true
```

Startup seemed to happen without incident.

Curiously, it comprehensively rewrote the yaml confi file.

Finally, I daemonised the system (using their instructions; I set `StandardOutput` to `null`):

```
sudo nano /etc/systemd/system/zigbee2mqtt.service
sudo systemctl start zigbee2mqtt
sudo systemctl status zigbee2mqtt
sudo systemctl enable zigbee2mqtt.service
```

And, restarted to make sure things worked. Of course, it didn't....

## Gotcha

Line one of the Zigbee2MQTT config file had defaulted to: `homeassistant: false`. I needed to change it to `homeassistant: true` (and restart).

## Post Installation

I was able to connect to the Zigbee2MQTT management console: http://192.168.0.252:8080/#/ (obviously, your IP address will change).




# Internet Monitoring: Latency

As a place to start, I want to track the performance of my network. Initially, this means pinging a rande of servers and gateways to guage reachability and latency. Speed will come later.

## Choices

CollectD versus Telegraf versus [home grown](https://github.com/renenw/ping): Both Telegraf and CollectD include ping modules - so this feels like a better approach than rolling my own. Also, Influx exists in the same ecosystem as Grafana and Home Assistant. So the Telegraf-Influx pair seem lke a natural choice.

## Installation

Installed InfluxDB using their documentation. Did not install the CLI, instead using their local web interface.

Used the local web interface to install and configure the Telegraf Ping plugin.

Configured Ping to hit:
```
  ## Gateway, Google (probably JHB), "us-east-1", "eu-west-1", "af-south-1"
  urls = ["105.233.1.138", "8.8.8.8", "3.80.0.0", "3.248.0.0", "13.245.0.253"]
```

Note that the AWS IPs were obtained from the [AWS Reachability Page](http://ec2-reachability.amazonaws.com/)




