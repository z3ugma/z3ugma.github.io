---
layout: post
title: "WasherBot and DryerBot, Part 1"
date: 2016-11-21 12:35:00 -0500
comments: true
published: false
---

The time indicators on my washer and dryer are notoriously unreliable. Also, my and Mel's busy brains occasionally forget that we started doing laundry. I wanted to build a system that would __text me when my washer or dryer finished running__.

This is part 1 of a 4-part series. You can see the other parts below:

* [Part 2]()
* [Parts List]()

When I last ran into a [problem that needed measuring](), I turned to some [preassembled USB temperature sensors]() and solved the problem with software only. 

While searching for a solution to this problem, I came up empty - it seems like any prefab way to measure this would involve either buying a new washer and dryer or buying expensive hardware. I therefore decided to make my first foray into electronic hardware in over a decade.

###How to Tell If an Appliance Is Running###

The first part of getting my washer to text me that it's done is to tell when it is running. I thought of a few ways to do this:

1. Tap into the circuitry of the appliance and check if a certain part is running. This will work well for "smart" appliances with circuit boards, but would require messing around inside the washer/dryer, possibly voiding my warranty, and needing to come up with a new method if I get a new washer/dryer.

2. Measure the power consumption of the appliance with a Hall effect sensor. If the appliance is consuming power, consider it "running." This would involve putting a sensor inline with the power cable feeding an appliance, such as the [ACS712](). This is a more-attractive option because it's a one-size-fits-all approach for both the washer and the dryer, could be extended to other appliances like a dishwasher or a rice cooker. Its one downside is that it's hard to use with a dryer specically and the different-sized plug.

3. Measure power as in option 2, but instead use a noninvasive current transformer sensor. This is used for whole-home monitoring solutions like [Neurio]() (an option I considered, but seems too unproven in the market). It uses a ring-shaped clamp that goes around one of the wires in the circuit. This has the benefit of being wholly separate from the mains current, making it safer.

I went with option 3 - using a CT sensor. Since I already had a Raspberry Pi available, I decided I would use it as the platform for building my device. It has the benefit of being easy to program, running a full Linux desktop environment, and having tons of forum support online. Some other options that are possible future upgrades include [Arduino]() and an [ET286]().

These CT sensors are very sensitive - they can measure current to a precise degree and can be used to tell exactly how much power you're consuming. My needs are a lot simpler - I don't need to know _how much_ power is being consumed, just whether or not it is.

So here are the builidng blocks: a Rapsberry Pi to run software and send text messages; current sensors to measure the power usage, and...something in the middle.

###Something in the Middle###

One thing I quickly realized is that the Raspberry Pi does not have an analog input. Raspberry Pi's inputs, called general-purpose-input-ouput or GPIO, can only tell whether a volatge is above a certain threshold ("high") or below it ("low"). This is a digital input.

Current sensors work by taking a high voltage current at high amperage and converting it down to a few milliamps of current at only a few volts. But since they are measuring [alternating current](https://learn.sparkfun.com/tutorials/alternating-current-ac-vs-direct-current-dc), the voltage will be rapidly switching signs back and forth:

<img src="https://cdn.sparkfun.com/r/600-600/assets/learn_tutorials/1/1/5/AC_sinewave_1.png" />

Theoretically, I could have used the GPIO to measure this state with only 1 bit of resolution (very crude, only on or off). I wanted something a little more precise, and some Googling led me to an [Analog to Digital Converter (ADC)](https://learn.adafruit.com/raspberry-pi-analog-to-digital-converters/overview).

An ADC takes an analog signal and discretizes it - forces it into specific steps. Since I wanted to measure 2 inputs (washer and dryer), I got a 2-channel ADC, the [MCP3002](https://www.sparkfun.com/products/8636). This chip has a 10-bit resolution, which gives you 1024 possible values instead of only 2. Much better.




