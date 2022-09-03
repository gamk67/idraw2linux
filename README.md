# How to set up and control the idraw v2 A3 plotter under Linux

I recently (June 2022) bought the idraw v2 A3 plotter. It's an
excellent piece of kit and I'm very happy with it. However, it's
not supported under linux - the Inkscape extension does not check
for the linux port name, so you just get connection errors. I
prefer to work from the command line anyway, so here's how I set
it up. I think this will also work well for other GRBL plotters.
Hopefully somebody will find this useful.

## HARDWARE


I bought the idraw v2 plotter version without a base. This needs
to be setup carefully. The kit comes with four metal bars. Two of
them are short (I'm calling these the 'X-bars') and two are long
(the 'Y-bar' and the 'arm', the latter being the part that moves).
The Y-bar is connected to the X-bars with two smallish screws that
allow some movement (perhaps deliberately). This means that the
arm is not guaranteed to be level. If it's pointing upward the 
pen is lifted from the paper on the right hand side, if it's
pointing downwards the pen starts to dig into the paper - neither
is good. I loosened the screws and supported the arm with a small
piece of wood while positioning it using a small spirit level. A bit
fiddly but in the end I got it level.

The circuit board does not remember the position of the head when
the power goes off, so you need to move the arm and head manually
to the (0,0) position before starting (to the other end of the
Y-bar opposite the circuit board). WARNING: do this slowly and
gently because moving the arm this way creates induction currents
in the circuit board. When the power goes off the firmware always
performs the pen-up action.

## SOFTWARE

Briefly, you start with an SVG file. This is first optimized, then
the optimized version is converted into GCODE, and finally the 
GCODE is sent to the plotter.

The serial port on my system (Tumbleweed) is called `/dev/ttyUSB0`.
I understand that on Raspberry Pi's it's called `/dev/ttyACM0`, and it
may be different on other systems still. I'm sticking here to the
first version.

### TIO

First of all, it's useful to check out the firmware with a terminal
serial port comms program such as `tio` (available through most
package managers). Start with
```
     tio -e /dev/ttyUSB0
```
You should see the GRBL firmware 'chatter' string, after which you
can type in some commands. The device needs to be initialized, for
which I use

     G21 G17 G90 F100 G00 Z0 G92 X0 Y0 Z0
This means:

- `G21`  use millimeters
- `G17`  use the XY plane (this is not a 3D device)
- `G90`  use absolute positioning
- `F100` use speed 2500 for drawing (max 25000, used for moving with pen
      up), you may want to increase this
- `G00 Z0` pen up
- `G92 X0Y0Z0` save the current location of the head as (0,0,0)

Please note that Z increases in the *downward* direction. The pen-down command is
```
     G00 Z5
```
Please note that the 5mm may not be the right one for your device, you may have to experiment a bit to get it right.

After this you can send move commands such as
```
     G00 X50 Y50
```
and you should see the arm move to position (50,50,0).

## vpype

I use the `vpype` command to optimise the SVG. Install with

     pip3 install vpype

The command I use is (remember, this is for A3 paper)

```
vpype read inputfile.svg scaleto 26.7cm 39cm linemerge -t 0.1mmlinesort write --page-size a3 --format svg --center output.svg
```
`vpype` requires a configuration file called `.vpype.toml` (note
the dot at the start) which is placed in your home directory.
It looks like this:
```
[gwrite.idraw]
default_profile = "gcode"
unit = "mm"
invert_y = true
document_start = "G21\nG17\nG90\nF2500\nG00 Z0\nG92 X0 Y0 Z0\n"
layer_start = ""
line_start = ""
line_end = ""
segment_first = """G00 X{x:.4f} Y{y:.4f}\nG00 Z5\n"""
segment_last = """G00 Z0\n"""
segment = "G01 X{x:.4f} Y{y:.4f}\n"
document_end = """G00 X0 Y0 Z0"""
```

## juicy-gcode

I use the `juicy-gcode` command to convert the SVG to GCODE
[link here](https://github.com/domoszlai/juicy-gcode).

The command is (using `output.svg` from the previous step)
```
    juicy-gcode output.svg -f flavor.txt -o output.gcode
```
This requires a configuration file called flavor.txt that
looks like this:
```
gcode
{
begin = "G21\nG17\nG90\nF2500\nG00 Z0\nG92 X0 Y0 Z0\n"
end = "G00 X0 Y0 Z0"
tooloff ="G0Z0"
toolon = "G0Z5"
}
```

##  gcode-cli

Finally, I use the `gcode-cli` command to send output.gcode
to the plotter [link here](https://github.com/hzeller/gcode-cli). There
are alternatives as well, but I like this one.
This program needs to be compiled from source by the command
```
   make
```
Before doing that, it's worthwhile to edit the file `main.cc`
and make a couple of changes.
First find the line that says

    int initial_squash_chatter_ms = 300;
Change the 300 to 2000 to allow plenty of time for the
firmware chatter to finish (otherwise the program will hang).
Then find all occurrences of `ttyACM0` and replace them with
`ttyUSB0` (or whatever on your system).

To use the program simply type
```
    gcode-cli output.gcode
```
and enjoy the result.

Cheers
