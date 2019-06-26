---
layout: post
title: "Flappy Bird clone in VHDL"
description: "Report about how I implement BlobbyFish (Flappy Bird clone) in VHDL."
comments: true
category: microcontroller
icon: assets/icons/icon-code.svg
keywords: ["vhdl", "fpga", "blobbyfish", "flappy", "bird"]
lang: en
---

Blobbyfish was implemented for a VHDL academic course project in 8 hours.
It is based on Flappy Bird but a little bit simplified for the purpose.

This is a team project made with [Otthorn](https://gitlab.crans.org/Otthorn).
It was a great project to start coding in VHDL and explore all main concepts.
Please note that this was implemented for
the **Digilent Basys 2** using **Xilinx ISE**.
All the code is available there under the [GPLv3 licence](https://github.com/erdnaxe/vhdl-blobbyfish/blob/master/LICENSE) :
<https://github.com/erdnaxe/vhdl-blobbyfish>.

* TOC
{:toc}

## Display controller

As this was quite a short deadline, this project reuse Digilent VGA controller
VHDL module `vga_controller_640_60` that is able to drive a 640x480 display with 8bit colors.

This controller takes a 25 MHz clock and generates the vertical and horizontal
VGA sync signals and counters to know which pixel the program need to define
color for.

However, if we directly use the integrated 25 MHz clock on the Basys 2, then
the image is deformed. As you can see on the following figure, vertical
lines are oscillating.

<img alt="Oscillating lines with internal clock" src="{{ site.baseurl }}/assets/images/vhdl-blobbyfish/display_deformation.jpg" width="600" />

To fix this issue, we added an external 50 MHz oscillator on the board and used
a D flip-flop to obtain a great 25 MHz clock.

## Definition of the BlobbyFish game

*Blobby* is a red flying fish. It can be sometimes referred as a "bird".

The player can only push a button to decide if *Blobby* jumps (or swims) up
or falls down.
*Blobby* fly to the right, but there are pipes to avoid.
In the code, Blobby is static and the pipes scrolls from the right to the left.

### Structure of VHDL modules

Our main file `Game.vhd` only routes signals between VHDL modules.
It also acts as a global clock divider to give each module its corresponding
clock.

The different implemented modules are :

  - `vga_controller_640_60` by Digilent : drives the VGA screen ;
  - `Display` : output the corresponding color of current pixel ;
  - `Fly` : make Blobby fly or fall (using a VHDL process) ;
  - `Pipe` : create a scrolling pipe (using a VHDL process) ;
  - `Collision` : reset game if a collision occurs.

The following figure sums up the code structure.

![VHDL modules connections]({{ site.baseurl }}/assets/images/vhdl-blobbyfish/schema.svg)

### How the game internally works

*Blobby* is represented by `altitude` (10 bits) and is always at the
X-coordinate `bird_X`.

A pipe is represented by the X-coordinate `pos_pipe`
and the Y-coordinate `alt_pipe`.
It is the pipes that move and not the fish.
`pos_pipe` has 64 pixels offset to make it progressively disappear without
having to use signed vectors.

![Schematic of how coordinates are used]({{ site.baseurl }}/assets/images/vhdl-blobbyfish/schema_display.png)

### Implementing acceleration

To add to the realism of the game, we decided to make the fish falls to
with a constant acceleration, but also with a speed limit.

To do this in VHDL, we defined an `acceleration` vector
and then at each iteration `speed` is incremented/decremented by `acceleration`.

### Adding textures

Embedded an image in VHDL is very hard (it is not meant for),
so only stripes had been done.
Summing `hcount` and `vcount` is enough to make good-looking stripes.
We add also to sum the position of the pipe to fix the bizarre-looking parallax
effect.

This leads to the final game that we can see on the final figure.

![Screenshot]({{ site.baseurl }}/assets/images/vhdl-blobbyfish/screen_capture.jpg)

If you want to test this game at home, the Xilinx project is available
on GitHub under the [GPLv3 licence](https://github.com/erdnaxe/vhdl-blobbyfish/blob/master/LICENSE) :
<https://github.com/erdnaxe/vhdl-blobbyfish>.
