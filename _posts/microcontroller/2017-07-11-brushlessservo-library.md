---
layout: post
title: "BrushlessServo Arduino library"
description: "Documentation of my BrushlessServo Arduino library"
comments: true
category: microcontroller
icon: assets/icons/icon-code.svg
cover: 2017-07-11-brushlessservo-library.png
keywords: ["arduino", "brushless", "servo", "library"]
lang: en
---

This library offers the possibility to use a Brushless 3-wire motor as a servomotor.
It's very convenient for making **fast precision movements** in many applications such as Gimbals. No need to buy or hack an existing ESC (Electronic Speed Control).

Please note that this library operates with **sensor-less brushless motors**. So the motor may skip steps!

> **Warning:**
> When you prototype electronic circuits with high current and Brushless motors,
> things can go wrong and blue magic smoke can appear! Be careful to choose the correct components.

* TOC
{:toc}

## Installation

* Open Arduino IDE,
* Go to `Sketch` > `Include Library` > `Manage Libraries...`,
* Then search for `Brushless Servo` and hit install.

### Settings

In `BrushlessServo.h` you can set the `PRECISION` constant to the number of divisions you want in one motor revolution.

## Reference

### attach()

Attach the BrushlessServo variable to 3 PWM pins.

These pins represent the 3 connectors of the brushless motor, but at low voltage (control signal for MOSFETs...).

Example :

``` c
#include <BrushlessServo.h> 

BrushlessServo myservo;

void setup() { 
  myservo.attach(9, 10, 11);
} 

void loop() {} 
```

### write()

Move to a position in degree. This method accepts floats.

``` c
myservo.write(0);
delay(500);
myservo.write(54.2);
```

### setOutputPower()

Set a power multiplier between 0 (min) and PWMRANGE (max).

This will be applied to the PWM generator.

Better used just after `attach()`.

``` c
#include <BrushlessServo.h> 

BrushlessServo myservo;

void setup() { 
  myservo.attach(9, 10, 11);
  myservo.setOutputPower(512);  // Mid power
} 

void loop() {} 
```

### setCycles()

Set how many sinusoid periods are needed for a full revolution. This method takes an integer.

Default to 8 cycles if not set.

## Demo

Here is a demo with an L293DNE and an ESP8266 running a simple sketch.

Each iteration, the motor is incremented of one degree.

<div class="responsive-iframe">
    <iframe src="https://player.vimeo.com/video/225227082" width="640" height="360" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>
</div>

