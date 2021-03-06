# SNAP Configuration and Bringup

The digital backend to the GReX system is a
[SNAP](https://casper.astro.berkeley.edu/wiki/SNAP) board from the CASPER group
at Berkeley. This board contains the analog to digital converters and Xilinx
FPGA to perform the digitization and F-engine components of the system.

The setup and configuration of this board has seemingly never been well
documented, so we'll try to make it as painless as possible here.

## KATCP and the Raspberry Pi

The SNAP board itself has no nonvolatile memory, so every time it is
power-cycled, it must be reconfigured.
Rather than require the user to bring along (and know how to use) a JTAG
programmer, the folks at CASPER have added a Raspsberry Pi header such that the
GPIO from a Pi can bit-bang JTAG. To expose this functionality from remote
devices, saving you the trouble from SSHing, they've implemented a KATCP
server on the Pi known as
[tcpborphserver](https://casper.astro.berkeley.edu/wiki/Tcpborphserver). 
The current source of tcpborphserver is
[here](https://github.com/casper-astro/katcp_devel/tree/rpi-devel-casperfpga).
[KATCP](https://katcp-python.readthedocs.io/en/latest/_downloads/361189acb383a294be20d6c10c257cb4/NRF-KAT7-6.0-IFCE-002-Rev5-1.pdf)
is a monitor and control protocol developed by the folks at SARAO that purports
easy usage and extension. [Here](https://casper.astro.berkeley.edu/wiki/KATCP)
is the list of commands they've added, which has not been updated since 2012.

### Raspberry Pi Setup

The image for the rasperry pi comes from
[here](https://casper.astro.berkeley.edu/wiki/SNAP_Bringup#Configuring_a_SNAP_Raspberry_Pi).
As GReX uses the RPi Model 3, grab that image and unzip it. You'll need at least
a 16 GB SD card to write it to. For some reason it has a few more bytes past a
standard SD card, so use something like
[PiShrink](https://github.com/Drewsif/PiShrink) to resize the image to its bare
minimum, then `dd` that, and hopefully it'll resize on boot.

By default, this image sets up static networking on `192.168.0.2\24`, so the
machine its hooked up to has to talk on that same subnet. This should already be
done on guix-provisioned GReX machines.

### KATCP Networking

The Pi is (hopefully) configured to speak KATCP on `10.10.1.3:7147`. You can check this by opening a telnet connection at that address
and running the `?watchdog` command. You should receive a `!watchdog ok`
response.

## Programming

The core of the SNAP is the FPGA, whose job in GReX is to grab voltage samples
from the analog to digital converters (ADCs), run them through a polyphase
filterbank (PBF) to channelize, and send those channels out over the 10 GbE
network interface. You would think this would be simple...

The "code" for the FPGA is stored as a bitstream file (BOF). How these files are
created is beyond the scope of these docs, but at some point we will provide
this binary blob. Before anything happens, this blob needs to be uploaded to the
FPGA. As mentioned before, this is done over katcp.

TODO! How do we do this? corr? casperfpga? Kiran's new rust tool?

## FPGA Clocks, References, PPS, Synthesizers

The FPGA needs an external clock source, which is being provided via one of the
output of the Valon synthesizer in box. Additionally, the board needs "pulse per
second" (PPS) ticks, which come from the GPS reciever. If the user is in a place
which doesn't have GPS (maybe in a lab during testing), we can override that
check.

TODO! Somehow

TODO! Check clock is good and everything is locked? Does this happen after ADC configuration?

## ADC Configuration

The next step is to set up the ADCs

## Networking

After the ADCs are setup, we need to configure the SNAP to start giving us data
over 10 GbE.

TODO! How?

## Bringup Verification

TODO! Do we run the test patterns to make sure things are working?
