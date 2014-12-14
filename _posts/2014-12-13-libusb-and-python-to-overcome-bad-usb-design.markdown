---
layout: post
title: " LibUSB and Python to Overcome Bad USB Design"
date: 2014-12-13 18:50:55 -0500
comments: true
published: true
---

[Last October, I began recording temperatures](http://z3ugma.github.io/2013/10/25/homebrew-temperature-monitoring/) for my homebrew setup. It used a C program called pcsensor that someone reverse-engineered from the Windows version available on the CD that came with my TEMPer1v1.4 sensors.

The problem with these sensors is that they have no unique identifier - reading the value in from pcsensor with multiple sensors installed often resulted in the data series being mixed together, because there was no "order" to the way data was being read into the parser.

Here's a demonstration of what I mean:

![Graph Interference](/images/graphinterference.png)

Notice how the order of the orange and red points switch a lot? This made it really difficult to analyze the temperatures in my setup, apart from using a human to look at the trendline created by the mix of points.

I spent some time trying to create a driver for these TEMPer sensors, but in the end I was able to find [one by Phillip Adelt that uses libusb and pyusb](https://github.com/padelt/temper-python) - libraries that allow you to talk to USB devices with Python.

{% highlight bash %}
git clone https://github.com/padelt/temper-python.git
cd temper-python
sudo python setup.py install
{% endhighlight %}

The ingenius thing that Phillip did was to determine the _port_ and _bus_ of each sensor - thus giving it a "unique ID" based on where it was physically plugged into the computer!

His module installs itself as "temperusb", so I wrote a quick script to read the value of the sensors:

{% highlight python %}
#!/usr/bin/python
# encoding: utf-8

from temperusb import TemperHandler
import json

th = TemperHandler()
devs = th.get_devices()


for i, dev in enumerate(devs):
    temps = dev.get_temperature(format="fahrenheit")
    ports = dev.get_ports()
    bus = dev.get_bus()
    print (bus, ports, temps)

{% endhighlight %}

Linux limits which users can access raw USB data, so we either need to install some [udev rules](http://www.reactivated.net/writing_udev_rules.html#example-pilot) or run the script as superuser. Sudo is easiest for me. It outputs something like:

{% highlight text %}
fred@brewery:~$ sudo python temper_read.py
[sudo] password for fred:
(2, 2, 56.8625)
(2, 1, 22.1)
fred@brewery:~$
{% endhighlight %}

The bus and port will be consistent, even if the _order_ of the entries gets switched. Now we can depend on the results of these sensors, and send them to InfluxDB:

{% highlight python %}
#!/usr/bin/python
# encoding: utf-8

from temperusb import TemperHandler
import json
import socket
hostname = socket.gethostname()

th = TemperHandler()
devs = th.get_devices()

from influxdb import InfluxDBClient
client = InfluxDBClient('hostname', 8086, 'root', 'root', 'temperatures')

for i, dev in enumerate(devs):
    d =[]
    ports = dev.get_ports()
    bus = dev.get_bus()
    d.append({"points":[[i, dev.get_temperature(format="fahrenheit"), ports, bus]], "name": "%s_temper_%s_%s" % (hostname,ports, bus), "columns": ["index", "temperature", "port", "bus"]})
    try:
        client.write_points(d)
    except:
        raise Exception("Couldn't connect to influxdb")
        print "Couldn't connect to InfluxDB"
        print json.dumps(d,indent=2)

{% endhighlight %}

I put this in the superuser's crontab:

{%highlight bash %}
sudo crontab -e
{%endhighlight%}

{%highlight text %}
# m h  dom mon dow   command
* * * * * /usr/bin/python /home/fred/temper_influx.py
{% endhighlight %}

This sends the available temperatures to InfluxDB once a minute.

Which leaves me very happy with my temperature-monitoring setup:

![Grafana Temperatures Dashboard](/images/grafana_temperatures.png)


