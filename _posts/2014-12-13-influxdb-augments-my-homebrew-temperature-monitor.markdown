---
title: InfluxDB Augments My Homebrew Temperature Monitor
date: 2014-12-13 20:50:55 Z
layout: post
comments: true
---

[Last October, I began recording temperatures](http://z3ugma.github.io/2013/10/25/homebrew-temperature-monitoring/) for my homebrew setup. It was early in my explorations of Python and scripting, and ran in a pretty silly fashion. It went something like this:

1. Echo the output of a C program that read the temperatures into a log file
2. Parse that logfile using python to create a JSON file for Google Charts to use
3. Use PHP embedded in HTML via an AJAX callback function to parse that JSON file
4. Display the result in Google Charts

This was incredibly slow and process intensive when more than 1 week's worth of data was captured, so I moved the log to a backup file weekly, leaving a blank file for the next week's data.

[InfluxDB](http://influxdb.com/) has been [getting a lot of buzz](http://www.xaprb.com/blog/2014/03/02/time-series-databases-influxdb/) on HackerNews lately. It is a time-series database, which means that the [primary key](http://influxdb.com/docs/v0.8/introduction/getting_started.html#writing-and-exploring-data-in-the-ui) of any entry is the timestamp of when the data was entered into the database. Queries are done using a SQL-like syntax, but everything is oriented around time. Queries like "give me the last data value of every hour" and "what is the average value of this data point for the past 3 Wednesdays" are difficult to do with large data sets in a traditional SQL database, but InfluxDB is designed from the ground up to handle them.

For a future idea - how to join time-series data from Influx to regular relational data in MySQL? This is relevant for another project I'm currently working on. Keep an eye out for updates on that project as well.

What won me over to databases like InfluxDB and [RethinkDB](http://rethinkdb.com/) is the simple, no-nonsense, easy-to-use web interface that comes ready right out of the box, and the SQL-like query language. Another big win for Influx was support from [Grafana](http://grafana.org/), a turnkey open-source time-series graphing platform, which I highly encourage you to check out.

I set up InfluxDB on a Vagrant VM on my Mac server, with the Vagrantfile like so:

{% highlight ruby %}
# -*- mode: ruby -*-
# vi: set ft=ruby :
# encoding: utf-8

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
 config.vm.box = "hashicorp/precise64"
 
 dirname = File.basename(Dir.getwd)
 config.vm.hostname = dirname

 config.vm.network "forwarded_port", guest: 8083, host: 8083
 config.vm.network "forwarded_port", guest: 8086, host: 8086
 config.vm.network "forwarded_port", guest: 8090, host: 8090
 config.vm.network "forwarded_port", guest: 8099, host: 8099
 config.vm.network "forwarded_port", guest: 80, host: 8089


$script = <<SCRIPT
echo "Provisioning InfluxDB"
sudo apt-get update
sudo apt-get -y install python python-pip git curl upstart apache2

sudo pip install nest-thermostat influxdb python-forecastio

wget http://s3.amazonaws.com/influxdb/influxdb_latest_amd64.deb
sudo dpkg -i influxdb_latest_amd64.deb

wget http://grafanarel.s3.amazonaws.com/grafana-1.9.0.tar.gz
tar -xf grafana-1.9.0.tar.gz
sudo cp -R grafana-1.9.0/* /var/www/
sudo cp /vagrant/config.js /var/www/config.js
sudo chmod -R 777 /var/www

sudo service influxdb start

sudo initctl emit vagrant-ready
SCRIPT

config.vm.provision "shell", inline: $script
config.vm.provision "shell", inline: "export EDITOR=nano", privileged: false
config.vm.provision "shell", inline: "(crontab -l 2>/dev/null; echo \"* * * * * /usr/bin/python /vagrant/nesttemp.py\") | crontab -", privileged: false
config.vm.provision "shell", inline: "(crontab -l 2>/dev/null; echo \"*/2 * * * * /usr/bin/python /vagrant/forecastweather.py\") | crontab -", privileged: false
config.vm.provision "shell", inline: "/usr/bin/python /vagrant/db_migrations.py", privileged: false

end
{% endhighlight%}

This forwards the InfluxDB ports, port 80 for serving Grafana, installs influx, Grafana, and python dependencies, and sets up cron jobs (see below).

With InfluxDB running, the first step was to capture some data! Since I bought a Nest themostat, I figured that would be a good piece of time-series data to measure.

{% highlight python %}
#!/usr/bin/python

import nest_thermostat
import influxdb

nest = nest_thermostat.Nest("username", "password")
nest.login()
nest.get_status()

temp = nest.temp_out(nest.status["shared"][nest.serial]["current_temperature"])
mode = nest.status["shared"][nest.serial]["target_temperature_type"]
target = nest.temp_out(nest.status["shared"][nest.serial]["target_temperature"])

from influxdb import client as influxdb
db = influxdb.InfluxDBClient("localhost", 8086, "root", "root", "temperatures")

data = [
  {"points":[[temp, target, mode]],
   "name":"nest",
   "columns":["temperature", "target_temperature", "type"]
  }
]
db.write_points(data)

{% endhighlight %}

Running this every minute via cron pulls the current temperature as well as the target temperature (what the thermostat is set to) into InfluxDB, through its easy-to-use but peculiarly-structured API.

I did the same thing with [Forecast.io](http://forecast.io/), my favorite weather API. This tells me the temperature outside my house:

{% highlight python %}
#!/usr/bin/python
import forecastio

api_key = "#####################"
lat = 43.05660
lng = -89.38337 

forecast = forecastio.load_forecast(api_key, lat, lng)

temp = forecast.hourly().data[0].temperature


from influxdb import client as influxdb

db = influxdb.InfluxDBClient("localhost", 8086, "root", "root", "temperatures")

data = [
  {"points":[[temp]],
   "name":"forecastio_lakeside",
   "columns":["temperature"]
  }
]
db.write_points(data)
{% endhighlight %}

This required me to get a Forecast.io API key, with a limit of 1000 calls per day. That's why I call it once every 2 minutes, for 720 calls per day.

The last part of setting up the VM is to perform database migrations - set up users and databases for the temperature data to go into:

{% highlight python %}
#!/usr/bin/python

import influxdb

from influxdb import client as influxdb
db = influxdb.InfluxDBClient("localhost", 8086, "root", "root")

try:
	db.create_database("grafana")
	db.create_database("temperatures")
	db.switch_db("temperatures")
	db.add_database_user("temperatures", "password")
	db.set_database_admin("temperatures")
	print "Success"
except:
	print "Influx DB Migrations Failed"
	pass
{% endhighlight %}










