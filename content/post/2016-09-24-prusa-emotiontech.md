---
categories:
- Hardware
date: "2016-09-24"
title: "My Prusa i3"
description: "This is what I learned about 3d-printing and my experience using EMotionTech Prusa i3."
keywords: ["3d", "printer", "emotiontech", "reprap", "prusa", "i3"]
aliases: ["/prusa-emotiontech/"]
---

> **WIP:**
> This is Work in Progress. So this article may contain mistakes and you may find errors or incomplete instructions. Who knows, maybe you will find a blue police box?

## The beginning of the story: how I choose my printer and all the problems I got
After one year of online research and talking with friends I finally choose to buy a kit. Because I live in France, I choose a Prusa i3 Rework from eMotionTech. Please note that this 3d printer had changed a lot since I bought one, so you may not encounter all the problems I got.

The build was easy but the documentation was wrong sometimes and very imprecise. During my first prints, I learnt a lot of things and I discovered that some part was mounted upside-down...

The print quality was horrible because there wasn't any X-axis belt-tensioner. The parts weren't sticking to the heated bed because there wasn't any bed leveling system.

eMotionTech give a 1kg ABS plastic spool with the printer. Now they give PLA and that's really a better idea because it's really hard to calibrate a printer with ABS. The print bed was taking more than 20 minutes to heat up so it was horrible each time a print failed to clean everything and then restart the printer.

Please note that eMotionTech give an old Marlin firmware configured for the Ultimaker (default settings). Now with the new revision, there is a brand new Configuration.h with custom PID and some cool extra stuff.

## Fixing problems
### Use PLA!
Because it was just impossible to print ABS correctly (without a cracking layer), I gave up and used high-quality PLA: it changed really everything.

### Bed leveling
The first big problem was the bed leveling setup. I came with loads of ideas such as springs, LEGO on the endstop... but I finally gave up and bought a servo motor and mounted my servo on the X carriage to use auto bed leveling. This is the greatest thing about the Prusa: no more manual calibration and perfect first layer height.

### Heated bed
The second problem was the heated bed.

I discovered that the Mosfet which takes care of the heated bed was overheating was too much. So I soldered a new Mosfet on my Ramps.

For insulation, I use a piece of cork (it's soo cheap) and I put it under the bed: now the bed heats up way faster.

### X-axis tensioner
The third main problem was the X-axis belt. Now there are many models on Thingiverse to fix that.

After printing some, I really recommend these parts : [Prusa i3 Enhanced by ooznest](http://www.thingiverse.com/thing:707109).

### Power supply
I also changed the power supply (the original one was a 12V led PSU) with a computer ATX power supply.

I soldered the PS-ON pin on the Ramps and used ATX cables to make clean connections. Marlin does support ATX power supply, so I can kill the power supply at the end of a print.

Now I can let my printer print during hours without cables heating up or being disturbed by a broken fan sound and I have enough current to power my heated bed.

### Conclusion
Now I have a zombie-looking printer with more the half of the parts printed myself! The quality is really different, I can now print ABS with a very consistent result.

## Recommendation about electronic
I really recommend adding a screen to the printer, or a Raspberry Pi with OctoPrint. It simplifies printing and it doesn't monopolize a computer.

The Arduino Mega + Ramps combination isn't the perfect board for a 3d-printer. Although I had to solder another Mosfet for my hungry heated bed, I quite like the fact that you can change to Polulus. If you want a silent printer you can put silent Polulus for example.

