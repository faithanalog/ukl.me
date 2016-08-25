#Max Pixel Fill Speed With OTIR

One of the fastest practical ways to write arbitrary pixels (no 1-color rectangles) to the LCD is
with an OTIR loop copying pixel data directly from memory. That being said, I was wondering how many
pixels you could theoretically update per frame. As mentioned, this is theoretical. This does not
take into account the overhead of adjusting the LCD window, or any of the other logic you may have.
It also assumes that interrupts are disabled.

##The Code

The code I'll be using to calculate max fill rate is

```z80
;Input: HL = data
;       DE = Number of pixels to output
;            Generally this would be width * height
;            This means that DE can never be the size of the ENTIRE screen,
;            just 85.3% of it.

    ld b,e
    ld c,11h

;If B != 0, we need an extra iteration, so D needs to be incremented
    dec de
    inc d

_dispSprtLp:
    otir
    dec d
    jp nz,_dispSprtLp
```

I use JP in the loop to save a few extra cycles. When calculating the numbers below, I'm basing it
only on the runtime of the loop, and I'm not calculating the negligible setup time for the loop.

##The Numbers
The TI-84+CSE uses a z80 processor, which can be set to run at 15Mhz. This means that it has about
15,000,000 cycles (or T-States) per second. All instructions take more than one T-State to run. The
ones we care about are:

`OTIR: (21 * (B - 1)) + 16` If `B == 0`, evaluate `B - 1` as `255`

`DEC D: 4`

`JP nz, ADDR: 10`

Now, the hardware adds an extra 4 T-states to ever OUT to the LCD port, so
the TRUE cost of OTIR is

`OTIR: (25 * (B - 1)) + 20`

Based on those numbers, we can calculate the amount of T-states for any given
input of DE

When B = 0, the loop takes (25 * 255 + 20) + 4 + 10 T-States, or 6409.
Therefore if E == 0

`f(D,0) = 6409 * D`

Otherwise if E > 0

`f(D,E) = ((25 * (E - 1)) + 34) + 6409 * D`


To simplify calculations, I'm going to ignore the fact that 'D' can not be greater than 255. This
allows me to use the above equation for the dimensions of the entire screen, and get a reasonably
close estimate of how much time it would take.

##The Calculations
The screen is 320x240 pixels large. Each pixel is 2 bytes. 320 * 240 * 2 = 153600. 153600 / 256 =
600, so D = 600 and E = 0. 600 * 6409 = 3,845,400.

To update the entire screen with this method it takes 0.25636 seconds. Not very promising, but
here's a table of screen percentages vs how much time it takes to update. For any dimensions that
you want to calculate yourself, the simplest way would be to use this formula.

`(WIDTH * HEIGHT) / (320 * 240) * 3845400`

Divide 15,000,000 by the result to get framerate.

Percent of Screen | T-states  | Theoretical Frames/Second | Dimensions
----------------- | --------  | ------------------------- | ----------
100%              | 3,845,400 | 3.9 FPS                   | 320 x 240
50%               | 1,922,700 | 7.8 FPS                   | 160 x 240 (Half Res)
5.3%              | 205,088   | 73.13 FPS                 | 64 x 64 (Large sprite)
2.7%              | 102,544   | 146.28 FPS                | 32 x 64 (Large sprite Half Res)
