---
layout: post
title: "WasherBot and DryerBot, Part 2: Current Sensors and Connecting Them to the ADC"
date: 2016-11-22 12:35:00 -0500
comments: true
published: true
imageurl: "/images/washerdryerbot/solder_plug.jpg"
description: "Finding a list of parts for WasherBot and DryerBot, soldering, and starting a circuit for the Pi"
---

{% include washerdryerbot_links.markdown %}

As you read in [Part 1]({% post_url 2016-11-21-washerbot-and-dryerbot-part-1 %}), I've now got a plan to get my washer and dryer to talk to me.

1. Clip a CT sensor to the power line.
2. Hook the CT sensor up to an analog-to-digital converter
3. Connect that ADC to a Raspberry Pi
4. Write some software for the Pi to measure power consumption and notify me.

Off to SparkFun.com for some parts.

First, 2 CT sensors to measure our current. 30 amps should be fine:
[__Non-Invasive Current Sensor - 30A__
![Non-Invasive Current Sensor - 30A](https://cdn.sparkfun.com//assets/parts/6/2/6/2/11005-01a.jpg){:width="300px"}](https://www.sparkfun.com/products/11005)

You'll notice that these come with 3.5mm TRS plugs on the end. This will make them easy to plug in and remove, but now I need a jack to receive them.
[__SparkFun TRRS 3.5mm Jack Breakout__
![SparkFun TRRS 3.5mm Jack Breakout](https://cdn.sparkfun.com//assets/parts/7/5/4/6/11570-01.jpg){:width="300px"}](https://www.sparkfun.com/products/11570)

These jacks come already assigned to a breakout board, which will make it convenient to use them with a breadboard - once some headers are soldered on:
[__Break Away Headers - Straight__
![Break Away Headers - Straight](https://cdn.sparkfun.com//assets/parts/1/0/6/00116-02-L.jpg){:width="300px"}](https://www.sparkfun.com/products/116)

Lastly, I actually need to buy the ADC mentioned in part 1:
[__Analog to Digital Converter - MCP3002__
![Analog to Digital Converter - MCP3002](https://cdn.sparkfun.com//assets/parts/1/7/7/0/08636-03-L.jpg){:width="300px"}](https://www.sparkfun.com/products/8636)

### Soldering

Once the parts arrived, I needed to solder the header pins onto the jack breakout for attaching to the breadboard. I'd never soldered delicate electronics, only larger wires so I picked up some 0.32 solder and watched [some tutorials](https://www.youtube.com/watch?v=f95i88OSWB4).

![soldering headers onto the jack](/images/washerdryerbot/solder_plug.jpg)

With the parts soldered and prepared, it's time to start measuring.

### Measuring Current ###

So a little math here. According to the [datasheet](http://cdn.sparkfun.com/datasheets/Sensors/Current/ECS1030-L72-SPEC.pdf), our current sensor has a 2000-turn ratio. This means a current of 30 Amps, divided by 2000 turns, will yield a resultant current of 15mA. What voltage will that current be at? Well, theoretically, [an infinite voltage](http://electronics.stackexchange.com/a/151317) (I don't quite understand why this is, but I trust the opinion of people smarter than me). If you just hook the CT up to a multimeter, that meter will have resistance in the circuit and so you'll get a "voltage drop" across the two leads of the multimeter.

And so we get to measuring a 100-Watt lightbulb with just the multimeter hooked to it for a test. Turns out, with a measly 0.8 A of current, we're already up to 2V. This won't do. I need a way to get the voltage down further.

### Burden Resistor 

To get the voltage down further, I learned I need to use a [burden resistor](https://openenergymonitor.org/emon/buildingblocks/ct-sensors-interface). This is a resistor with a very specific resistance that matches your circuit in order to get to an expected voltage range. [This site has some great details and an Excel-based calculator for burden resistor resistance](https://openenergymonitor.org/emon/buildingblocks/ct-sensors-interface).

The key bit is that our maximum measurable voltage is 3.3V - that's the potential we're feeding into the MCP3002. Since we're measuring AC current, it means that 3.3V is the peak-to peak meaning that we actually want to measure a max of __1.65V__ as our zero-point or reference point. At the maximum peak of the AC wave, we'll have 1.65V reference + 1.65V from the sensor = 3.3V. At the minimum trough of the AC wave, we'll have 1.65V - 1.65V = 0V.

The equation for burden resistor calculation is:

![](/images/washerdryerbot/eqn1.gif)

With a 30A current, we'd have a burden resistor value of 38.8 Ohms.

![](/images/washerdryerbot/eqn2.gif)

Since RadioShack had __68__ Ohms as the closest resistor value (heh), I can actually measure up to 17A safely:

![](/images/washerdryerbot/eqn3.gif)

So we have a circuit that won't overvolt the Pi! In part 3, we'll hook the sensor up to the Pi and start measuring it.





