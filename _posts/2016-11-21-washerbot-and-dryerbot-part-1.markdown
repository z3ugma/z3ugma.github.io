---
layout: post
title: "WasherBot and DryerBot, Part 1: How to Tell If an Appliance Is Running"
date: 2016-11-21 12:35:00 -0500
comments: true
published: true
imageurl: "https://i.kinja-img.com/gawker-media/image/upload/s--NrejLvsE--/c_fit,fl_progressive,q_80,w_636/bucyzpye75xamq6yxxvm.jpg"
description: "Getting texts from my washer & dryer using a Raspberry Pi to measure power consumption"
---

The time indicators on my washer and dryer are notoriously unreliable. Also, my and Mel's busy brains occasionally forget that we started doing laundry. I wanted to build a system that would __text me when my washer or dryer finished running__.

When I last ran into a [problem that would require this kind of measurement](/2013/10/24/homebrew-temperature-monitoring/), I turned to some [preassembled USB temperature sensors](http://www.pcsensor.com/usb-thermometers.html) and solved the problem with software only. 

While searching for some hardware to help me tell if my washer or dryer were running, I came up empty - it seems like any prefab way to measure this would involve either buying a new washer and dryer or buying expensive hardware. I therefore decided to make my first foray into electronics hardware.

###How to Tell If an Appliance Is Running###

The first part of getting my washer to text me that it's done is to tell when it is running. I thought of a few ways to do this:

1. Tap into the circuitry of the appliance and check if a certain part is running. This will work well for "smart" appliances with circuit boards, but would require messing around inside the washer/dryer, possibly voiding my warranty, and needing to come up with a new method if I get a new washer/dryer.

2. Sense vibration the way that a few other online solutions do. This seemed highly error-prone to me.

3. Measure the power consumption of the appliance with a Hall effect sensor. If the appliance is consuming power, consider it "running." This would involve putting a sensor inline with the power cable feeding an appliance, such as the [ACS712](https://www.amazon.com/Current-Sensor-Module-ACS712-Electronic/dp/B00COD9GO2/ref=sr_1_5?ie=UTF8&qid=1480265285&sr=8-5&keywords=ACS712). This is a more-attractive option because it's a one-size-fits-all approach for both the washer and the dryer and could be extended to other appliances like a dishwasher or a rice cooker. The downside is that it involves interrupting the flow of power and messing with mains power, which is risky.
[![The ACS712 in action](/images/washerdryerbot/acs712_in_action.png)](https://www.youtube.com/watch?v=UF5jrnXvTlM&t=3m15s)

4. Measure power as in option 2, but instead use a noninvasive current transformer sensor. This what whole-home monitoring solutions like [Neurio](http://neur.io/products/) use. Neurio is one of the premade options I considered, but it seems too unproven and new. CT sensors use a ring-shaped clamp that goes around one of the wires in the circuit. This has the benefit of being wholly separate from the mains current, making it safer.
![Neurio uses 2 CT sensors](http://neur.io/wp-content/uploads/HEM_Contents.jpg)

I opted to use a CT sensor. Since I already had a Raspberry Pi available, I decided I would use it as the platform for building my device. It has the benefit of being easy to program, running a full Linux desktop environment, and having tons of forum support online. Another option is to use an [Arduino](https://openenergymonitor.org/emon/buildingblocks/how-to-build-an-arduino-energy-monitor-measuring-current-only). In the future, I'd like to try migrating to a standalone, low-power board like the [MT7688AN](http://hackerboards.com/tiny-iot-sbc-runs-openwrt-on-mediatek-mips-soc/).

CT sensors are very sensitive - they can measure current to a precise degree and can be used to tell exactly how much power you're consuming. My needs are a lot simpler - I don't need to know _how much_ power is being consumed, just whether or not it is.

So here are the building blocks: a Raspberry Pi to run software and send text messages; current sensors to measure the power usage, and...something in the middle.

###Something in the Middle###

One thing I quickly realized is that the Raspberry Pi does not have an analog input. Raspberry Pi's inputs, called general-purpose-input-output or GPIO, can only tell whether a voltage is above a certain threshold ("high") or below it ("low"), because they are digital inputs.

Current sensors work by taking a high voltage current at high amperage and converting it down to a few milliamps of current at only a few volts. But since they are measuring [alternating current](https://learn.sparkfun.com/tutorials/alternating-current-ac-vs-direct-current-dc), the voltage will be rapidly switching signs back and forth:

![alternating current](https://cdn.sparkfun.com/r/600-600/assets/learn_tutorials/1/1/5/AC_sinewave_1.png)

Theoretically, I could have used the GPIO to measure this state with only 1 bit of resolution (very crude, only on or off). I wanted something a little more precise, and some Googling led me to an [Analog to Digital Converter (ADC)](https://learn.adafruit.com/raspberry-pi-analog-to-digital-converters/overview).

An ADC takes an analog signal and discretizes it - from an infinite number of values to specific steps. Since I wanted to measure 2 inputs (washer and dryer), I got a 2-channel ADC, the [MCP3002](https://www.sparkfun.com/products/8636). This chip has a 10-bit resolution, which gives you 1024 possible values instead of only 2. Much better.

Stay tuned for Part 2, in which I put together a parts list and get to soldering.


