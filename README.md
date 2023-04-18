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

## Installation

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

## Configuration.yaml

Is located at `/home/homeassistant/.homeassistant/configuration.yaml`

## App

You'll need to enable port 8123 to be forwarded to your server.

Once this is done, you'll need to configure your firewall to all traffic onto the server itself:
```
sudo ufw enable
sudo ufw allow 8123
```

## HACS

HACS is required by Frigate, recommended by YouTubers, and seems to be a compelling idea.

The `wget` step is from the HACS Core installation notes [here](https://hacs.xyz/docs/setup/download/).

```
sudo -u homeassistant -H -s
source /srv/homeassistant/bin/activate
wget -O - https://get.hacs.xyz | bash -
```

Ctrl-d to exit back to your usual user, then restart Home Assistant:

```
sudo service home-assistant@homeassistant stop
sudo service home-assistant@homeassistant start
```

# Databases

I'm going to be leveraging MySQL for a bunch of other work, so I think we should migrate Home Assistant to MySQL. This step, it seems, needs to be done _after_ HA is up and running. I did this after I'd added a few devices and they didn't vapourise. So, I conclude that SQLite is still being used.

InfluxDB seems to be pretty common in this domain, so we'll add him as well.

## MySQL

### Installation

First up, install MySQL:

```
sudo apt install mysql-server
sudo systemctl start mysql.service
sudo apt-get install default-libmysqlclient-dev libssl-dev
```

Then we need to a database and user for home assistant to use:

```
sudo mysql
SET GLOBAL default_storage_engine = 'InnoDB';
CREATE DATABASE homeassistant CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
use homeassistant;
CREATE USER 'homeassistant'@'%' IDENTIFIED BY 'xxxxxxx';
GRANT ALL PRIVILEGES ON homeassistant.* TO 'homeassistant'@'%';
```

That was the easy part. Getting HA to actually use the database was a little harder.

### Configure HA

This document was instrumental to getting things up and running: https://www.home-assistant.io/integrations/recorder/

You might want to scan over this thread as well: https://community.home-assistant.io/t/anybody-with-experience-moving-off-sqlite-into-a-db-i-e-mysql-maria/217861/12

We need to install the Python mysql client for HA:

```
renen@server:~ $ sudo -u homeassistant -H -s
renen@server:~$ source /srv/homeassistant/bin/activate
(homeassistant) renen@server:~$ pip3 install mysqlclient
```

I don't understand that "active" step: but without it, this process doesn't work.

### Service Dependencies

Of course, now we need Home Assistant to start _after_ MySQL. We need to adjust the HA service config:

```
[Unit]
Description=Home Assistant
After=network.target mysql.service
```

Then:
```
sudo systemctl daemon-reload
```

## InfluxDB

### Installation

I installed InfluxDB using their documentation. I did not install the CLI, instead using their local web interface.

### HomeAssistant

Getting HA to push data to Influx was straight forward. I added an all-powerful API key, and then used that to tell HA to push to Influx by adding the following to the configuration file:

```
influxdb:
  api_version: 2
  ssl: false
  host: localhost
  port: 8086
  token: GENERATED_AUTH_TOKEN
  organization: 9c4f3c1e94a7715a
  bucket: homeassistant
```

Note that, curiously, the organisation is referenced using a UUID type identifier.

### Giant Gotcha

Eash HA InfluxDb configuration block requires the server configuration parameters: `ssl`, `host` etc.

### Testing

You should see data showing up in Influx.

### Next Steps

It sounds like we will receive too much data. We will probably want to constrain this in due course.


# Telegraf

We are going to use Telegraf for a few things: Internet latency monitoring, perhaps UDP ingestion. Its also installed with InfluxDb - so may as well get it up and running here.

Ideally, you want Telegraf to pull its config from your InfluxDb server. But, getting Telegraf to run a service presented a range of learning opportunities. I struggled to get the Influx Token set properly, and I needed to ensure that Telegraf started _after_ InfluxDb.

## Influx DB Startup Dependency

As easy as expanding the "after" clause in the service definition file (`/lib/systemd/system/telegraf.service`).
```
After=network.target influxdb.service
````

## Environment Variables

These need to be set in the service override file. The best way to edit this: `sudo systemctl edit telegraf.service`

They do not need to be set via `.bashrc` or any other places.

After installing the Telegraf Ping plugin, that file is as follows:

```
### Editing /etc/systemd/system/telegraf.service.d/override.conf
### Anything between here and the comment below will become the new contents of the file

[Service]
CapabilityBoundingSet=CAP_NET_RAW
AmbientCapabilities=CAP_NET_RAW
Environment="INFLUX_TOKEN=i-<redacted>4mcg=="
LimitNOFILE=8192

### Lines below this comment will be discarded
```

## Telegraf.service

You also need to make sure that your service file properly references this override file. My service config file (`/lib/systemd/system/telegraf.service`; for noobs, edit using `sudo nano /lib/systemd/system/telegraf.service`) is as follows:

```
[Unit]
Description=The plugin-driven server agent for reporting metrics into InfluxDB
Documentation=https://github.com/influxdata/telegraf
After=network.target influxdb.service

[Service]
Type=notify
EnvironmentFile=/etc/systemd/system/telegraf.service.d/override.conf
User=_telegraf
Group=_telegraf
ExecStart=/usr/bin/telegraf -config http://localhost:8086/api/v2/telegrafs/0ab33db4c2ae8000
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartForceExitStatus=SIGPIPE
KillMode=control-group

[Install]
WantedBy=multi-user.target
```

The `after`, `EnvironmentFile` and `ExecStart` lines are the ones you'll need to edit.

`ExecStart` will change based on the example provided by Influx when you confiure a Telegraf plugin.

## All the steps

```
sudo systemctl edit telegraf.service 
sudo nano /lib/systemd/system/telegraf.service
sudo systemctl daemon-reload
sudo service telegraf stop
sudo service telegraf start
sudo service telegraf status
```

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

## Brdige

I subsequently added a second Sonoff Zigbee 3.0 USB Dongle which needed to be flashed to get it into bridge mode. I followed these instructions: https://sonoff.tech/wp-content/uploads/2021/12/SONOFF-Zigbee-3.0-USB-dongle-plus-firmware-flashing-1-1.pdf


# Internet Monitoring

## Latency

As a place to start, I want to track the performance of my network. Initially, this means pinging a rande of servers and gateways to guage reachability and latency. Speed will come later.

### Choices

CollectD versus Telegraf versus [home grown](https://github.com/renenw/ping): Both Telegraf and CollectD include ping modules - so this feels like a better approach than rolling my own. Also, Influx exists in the same ecosystem as Grafana and Home Assistant. So the Telegraf-Influx pair seem lke a natural choice.

### Installation

Used the local web interface to install and configure the Telegraf Ping plugin.

Within the Ping plugin confguration, set:
```
  ## Gateway, Google (probably JHB), "us-east-1", "eu-west-1", "af-south-1", "safaricom NBO"
  urls = ["105.233.1.138", "8.8.8.8", "3.80.0.0", "3.248.0.0", "13.245.0.253", "197.248.116.74"]
```

Note that the AWS IPs were obtained from the [AWS Reachability Page](http://ec2-reachability.amazonaws.com/) and [Public DNS Server List](https://public-dns.info/nameserver/ke.html)

## DNS

My ISP prefers us to use their DNS rather than Google's. I know the guys so I'll give them the benefit of the doubt: I don't think they're raping my data. But, either way, I'd like to know if their service really is faster than Google's. It should be, but I'd like to know for sure.

As such, I add the Telegraf DNS Plugin, and told it to start tracking both 8.8.8.8, and my ISP's DNS service.

## Speed

Enabled the Telegraf Internet Speed plugin.

### Post Script

Unsuprisingly, the local DNS (comprehensively!) beats 8.8.8.8 (which is, I believe, about 1000 miles away in Johannesburg).

## After Install

You will need to restart Telegraf:

```
sudo service telegraf stop
sudo service telegraf start
sudo service telegraf status
```

## Bucket Permissions

I want to push ping data to a ping specific bucket: `ping` (duh). Getting these permissions to work was tricky.

## Home Assistant Sensor

Now, to create a HomeAssistenat sensor to model these values. We will need to add variants of the following to our config file:

```
sudo /home/homeassistant/.homeassistant/configuration.yaml
```

```
sensor:
  - platform: influxdb
    api_version: 2
    organization: 9c4f3c1e94a7715a
    token: GENERATED_AUTH_TOKEN
    queries_flux:
      - group_function: last
        name: "Network Latency Ireland"
        unique_id: "eu-west-1-081753309423"
        unit_of_measurement: "ms"
        query: >
          filter(fn: (r) => r["_measurement"] == "ping")
            |> filter(fn: (r) => r["_field"] == "average_response_ms")
            |> filter(fn: (r) => r["host"] == "server")
            |> filter(fn: (r) => r["url"] == "3.248.0.0")
```

# Frigate

https://github.com/blakeblackshear/frigate/discussions/4041
file:///home/renen/Downloads/Installing.frigate.from.scratch.pdf
https://wiki.seeedstudio.com/ODYSSEY-X86J4105-Frigate/

Two config files:
1. Docker: ~/Documents/frigate.yml
2. Frigate: ~/Documents/frigate/config/config.yml

To start Frigate:

`sudo docker compose -f frigate.yml up`

This is useful: `sudo docker logs frigate`

List of hikvision streams: http://www.ispyconnect.com/camera/hikvision

May need to open 1935 so that HA can stream (https://docs.frigate.video/configuration/rtmp)

# Paradox

https://github.com/maisken/Paradox_IP150


# ESPHome

I don't have a handle on the virtual environment concept.

## Installation

You really want to do this from an Ubuntu machine. Well, at least, not Windows.

https://esphome.io/guides/installing_esphome.html

```
mkdir ~/esphome
python3 -m venv venv
source venv/bin/activate
pip3 install esphome
```

Then: `sudo ufw 6052`

## Usage

Start the dashboard as a service:

```
source venv/bin/activate
esphome dashboard /home/renen/esphome/
```

Then, access it: http://192.168.0.245:6052/

Plug devices into the remote server.
