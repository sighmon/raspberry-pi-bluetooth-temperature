# Raspberry Pi Zero W to Oregon Scientific IDTW211R

Connect a Raspberry Pi Zero W to an Oregon Scientific BLE temperature station and read temperature values in Python.

Credit: [Arnø](https://www.instructables.com/id/Connect-Raspberry-Pi-to-Oregon-Scientific-BLE-Weat/)

<img width="100%" src="IDTW211R-Oregon-Scientific-weather-station.jpg" />

## Hardware

* Raspberry Pi Zero W
* Oregon Scientific IDTW211R weather station

## Software

A Raspberry Pi Zero W with Raspbian Stretch has all of the tools to scan for your weather station. To find it's BLE address:

`$ sudo hcitool lescan`

You should be able to see it in the list, something like:

`<BLE ADDRESS> IDTW211R`

### Retrieve temperature data from the command line

Now that we know the address, we can connect to it:

```bash
$ sudo gatttool -b <BLE ADDRESS> -t random -I
[<BLE ADDRESS>][LE]> connect
Attempting to connect to <BLE ADDRESS>
Connection successful
[<BLE ADDRESS>][LE]> primary
attr handle: 0x0001, end grp handle: 0x0007 uuid: 00001800-0000-1000-8000-00805f9b34fb
attr handle: 0x0008, end grp handle: 0x0008 uuid: 00001801-0000-1000-8000-00805f9b34fb
attr handle: 0x0009, end grp handle: 0x0021 uuid: 74e7fe00-c6a4-11e2-b7a9-0002a5d5c51b
attr handle: 0x0022, end grp handle: 0x002e uuid: 0000180a-0000-1000-8000-00805f9b34fb
attr handle: 0x002f, end grp handle: 0xffff uuid: 0000180f-0000-1000-8000-00805f9b34fb
```

To receive data from it, we enable indication by writting 0x0002 (0x0001 for notification) in Client Characteristic Configuration Descriptor (UUID: 0x2902) of each characteristic. Write shall be placed in little endian format so:

```bash
[<BLE ADDRESS>][LE]> char-write-req 0x000c 0200
Characteristic value was written successfully
[<BLE ADDRESS>][LE]> char-write-req 0x000f 0200
Characteristic value was written successfully
[<BLE ADDRESS>][LE]> char-write-req 0x0012 0200
Characteristic value was written successfully
[<BLE ADDRESS>][LE]> char-write-req 0x0015 0100
Characteristic value was written successfully
[<BLE ADDRESS>][LE]> char-write-req 0x0018 0200
Characteristic value was written successfully
[<BLE ADDRESS>][LE]> char-write-req 0x001b 0200
Characteristic value was written successfully
[<BLE ADDRESS>][LE]> char-write-req 0x001e 0200
Characteristic value was written successfully
[<BLE ADDRESS>][LE]> char-write-req 0x0021 0200
Characteristic value was written successfully
[<BLE ADDRESS>][LE]> char-write-req 0x0032 0100
Characteristic value was written successfully
Indication   handle = 0x000e value: 00 19 01 00 00 00 00 00 
Indication   handle = 0x0017 value: 01 f9 00 1d 01 ff 7f ff 7f 7f 7f 7f 7f ff ff 7f 7f 7f 7f 7f 
Indication   handle = 0x0017 value: 82 7f 7f 7f f9 00 e3 00 1d 01 b6 00 ff 7f ff 7f ff 7f ff 7f 
Indication   handle = 0x0020 value: 01 ff 7f ff 7f ff 7f ff 7f 7f 7f 7f 7f ff ff 7f 7f 7f 7f 7f 
Indication   handle = 0x0020 value: 82 7f 7f 7f ff 7f ff 7f ff 7f ff 7f ff 7f ff 7f ff 7f ff 7f 
Indication   handle = 0x001d value: 11 12 0b 11 11 1e 34 ff 00 80 00 80 ff ff ff ff ff ff ff ff 
[<BLE ADDRESS>][LE]> 
```

In return, you should start getting indications/notifications. The relevant ones for us are `INDOOR_AND_CH1_TO_3_TH_DATA` indications (so handle 0x0017).

For each round of indications, we get two data packets of 20 bytes each. The most significant byte indicates the type of data (Type 0 or Type 1). See below for more details on the data packets.

<img width="100%" src="INDOOR_AND_CH1_TO_3_TH_DATA_types.jpg" />

You can also retrieve the battery level:

```bash
$ sudo gatttool -b <BLE ADDRESS> -t random --char-read --handle=0x0031
```

### Retrieve temperature data via Python

First install [bluepy](https://github.com/IanHarvey/bluepy):

```bash
$ sudo apt-get install python-pip libglib2.0-dev
$ sudo pip install bluepy
```

Now run the python script handing in the address of your weather station:

```bash
$ python temperature.py <BLE ADDRESS>
```

If you'd rather get the script to scan for the weather station, you need to run it with root privileges:

```bash
$ sudo python temperature.py
```

You should see output:

```bash
$ python temperature.py <BLE ADDRESS>
2018-11-17 17:41:38,600: WeatherStation connected !
2018-11-17 17:41:38,603: Notifications enabled
2018-11-17 17:41:38,939: handle e = 0019010000000000
2018-11-17 17:41:39,135: indoorAndOutdoorTemp_type0 = 01fb002201ff7fff7f7f7f7f7fffff7f7f7f7f7f
2018-11-17 17:41:39,330: indoorAndOutdoorTemp_type1 = 827f7f7ffb00e3002201b600ff7fff7fff7fff7f
2018-11-17 17:41:39,524: handle 20 = 01ff7fff7fff7fff7f7f7f7f7fffff7f7f7f7f7f
2018-11-17 17:41:39,671: handle 20 = 827f7f7fff7fff7fff7fff7fff7fff7fff7fff7f
2018-11-17 17:41:39,817: handle 1d = 11120b11112927ff00800080ffffffffffffffff
2018-11-17 17:41:40,819: Notification timeout
2018-11-17 17:41:40,820: Indoor temp : 25.1°C, max : 25.1°C, min : 22.7°C
2018-11-17 17:41:40,820: Outdoor temp : 29.0°C, max : 29.0°C, min : 18.2°C
```
### Run it hourly via a crontab

To set it up to run hourly, edit your crontab:

```bash
$ crontab -e
@hourly python /home/pi/temperature.py <BLE ADDRESS> >> /home/pi/cron-temperature.log
```

## TODO

* Setup a [Flask](http://flask.pocoo.org) app to expose the temperature getting via an API - maybe use https://www.flaskapi.org
* Connect that Flask app to Apple HomeKit using https://github.com/ikalchev/HAP-python
