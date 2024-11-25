# Box Software

There are four devices that require some software configuration: the Valon synthesizer, the SNAP FPGA, the 10 GbE switch, and the Raspberry Pi.
These configuration steps should already be performed before we ship a box, but for completeness, here are the steps that we performed.

## Valon

We need to configure the valon synthesizer to act as the LO for the downconverter and the reference clock for the SNAP ADC.

Use the GUI tool [here](https://valontechnology.com/5009users/5009.htm) to load [this](../assets/grex_valon.VR0) configuration file.
Next, go to synthesizer -> write registers.
Then, save the configuration to flash to preserve this configuration across reboots.

## Switch

With the box connected and powered on, create an SSH relay to the switch's configuration interface with

```sh
ssh -L 8291:192.168.88.1:8291 user@<the ip address of the server>
```

Then, using [winbox](www.mikrotik.com/download/winbox.exe) connect to localhost, 
select `files` on the left, and upload [this config file](../assets/GReX_Switch.backup). This should trigger a reboot.

## Raspberry Pi

We prepared the RPi image using the standard [raspbian lite OS](https://www.raspberrypi.com/software/operating-systems/).
As part of the initial image creation, we set the hostname to `grex-pi` and enabled password-based SSH.

We manually configure the rest of the Pi by plugging a monitor and keyboard into the Pi.

Using `raspi-config`, we did the following:
- disabled the serial login shell
- enabled the hardware serial interface

Then, we disabled the hardware's radios by modifying the `config.txt` file [like so](https://raspberrytips.com/disable-wifi-raspberry-pi/).

Then, we configured the Pi to have the static IP address of `192.168.0.2` by following [this](https://www.makeuseof.com/raspberry-pi-set-static-ip/), which boils down to adding the following to this file `/etc/dhcpcd.conf`:
```ini
interface enp1s0f0
static ip_address=192.168.0.2/24
static routers=192.168.0.1
static domain_name_servers=192.168.0.1
```

Then, we disabled HCI UART by running

```sh
sudo systemctl disable hciuart
```

## SNAP

See [SNAP Setup](snap.md)
