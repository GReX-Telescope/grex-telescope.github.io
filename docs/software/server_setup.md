# Server Setup

Once you get your server (either from Puget systems or otherwise), we need to setup additional hardware, adjust some system settings, setup networking, and install the pipeline software.

## Hardware Setup

The server straight from Puget does not have the GPU or 10 GbE network card installed, we will do this first.

1. Open the case and remove the GPU retention bracket
2. Remove the test GPU (T1000), keep this for later in case we need an aditional video output
3. Install the 3090 Ti in the first GPU slot, this will take up three slots of space
4. Install the network card in the bottom slot
5. Wire the 3090 Ti power cable to the harness provided by Puget (they knew this was the GPU we were going to install)
6. Remove the GPU retention clips from the retention bracket that would interfere with the card. It's too tall for it anyway.
7. Replace the retention bracket and close the case.

![](../assets/server.jpg)

Finally, we need to hook up a monitor to the 3090 Ti so we can setup the software

## Initial OS Setup

On boot, you will be presented with the Ubuntu graphical interface. If a password is requested, the deafult is provided in the information booklet that came with the hardware.

First, we will change the system password. Set this to something memorable.

```sh
passwd $(whoami)
```

Now, we will update the system. There shouldn't be many updates if this is a new machine.

```sh
sudo apt-get update
sudo apt-get upgrade -y
```

We will be editing a bunch of files, if you are comfy in the command line, you probably want to install some editors. Otherwise the graphical `gedit` tool will be fine.

```sh
sudo apt-get install emacs vim -y
```

Finally, we will set the hostname. We'll be using the `grex-<affiliation>-<location>` paradigm (just for clarity, no real reason not to).
As in, the first server that is managed by Caltech at OVRO will be `grex-caltech-ovro`.

```sh
sudo hostnamectl set-hostname <your-hostname>
```

Some updates may require a reboot. If it asks, do that now.

## Networking

Now, we need to setup the networking for the GReX system. We will operate under the assumption that the internet-facing connection will get an IP address from a DHCP server. If that is not the case, consult whoever runs your network on the appropriate setup. Regardless of the WAN connection, the 10 GbE fiber connection to the GReX terminal will be configured the same.

### Overview

The 10 GbE fiber port serves a few purposes. It is the main data transfer link between the FPGA in the field and the server, but it also carries the monitor and control for the box. This monitor and control connection includes the SNAP control connection and the Raspberry Pi. The SNAP requires an external DHCP server, which we have to provide on this port. Additionally, the 10 GbE switch in the box has its own default subnet for configuration (`192.168.88.X`). To make everything talk to each other, we need to add two IPs on this port: one in the subnet of the switch's config interface, and the other for DHCP of the various devices.

### Netplan

In `/etc/netplan` remove any files that are currently there.

Check whether you are using NetworkManager or networkd:

```
systemctl status NetworkManager
systemctl status systemd-networkd
```

If NetworkManager is running and networkd is not, disable NetworkManager and enable networkd. (Otherwise, skip this step.)

```
sudo systemctl stop NetworkManager
sudo systemctl disable NetworkManager
sudo systemctl enable systemd-networkd
```

Then, create a new file called `config.yaml` with the following contents

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    # Two WAN interfaces. Configure this according to your network setup
    enp36s0f0:
      dhcp4: true
    enp36s0f1:
      dhcp4: true
    # 10 GbE connection over fiber to the box
    enp1s0f0:
      mtu: 9000
      addresses:
        - 192.168.0.1/24
        - 192.168.88.2/24
```

Then apply with

```sh
sudo netplan apply
```

### DHCP Server

Now, we need to setup the DHCP server on the 10 GbE port. First, we install the DHCP server software:

```sh
sudo apt-get install dnsmasq
```

Create the configuration file in `/etc/dnsmasq.conf`

```ini
# Only bind to the 10 GbE interface
interface=enp1s0f0
# Disable DNS
port=0
# DHCP Options
dhcp-range=192.168.0.0,static
dhcp-option=option:router,192.168.0.1
dhcp-option=option:netmask,255.255.255.0
#dhcp-host=<SNAP_MAC>,192.168.0.3,snap
log-async
log-queries
log-dhcp
```

Then, enable the DHCP server service

```sh
sudo systemctl enable dnsmasq --now
```

This sets up a very simple DHCP server that will give the IP address `192.168.0.3` to the SNAP.
Unfortunately, the folks who set up the networking interface for the SNAP only provide a DHCP interface and a dynamic (non-observable) MAC address (for some reason).
As such, we have to now turn on the SNAP, wait for it to try to get an IP address from `dnsmasq` so we know it's MAC, then update the `dhcp-host` line and restart the DHCP server.

1. Power cycle the SNAP (or turn it on if it wasn't turned on yet) following the instructions in [operation](operation.md)
2. Wait a moment and open the log of dnsmasq with `journalctl -u dnsmasq`, skip to the bottom with `G` (Shift + g)
3. You should see a line like

```
Aug 16 14:39:06 grex-caltech-ovro dnsmasq-dhcp[5115]: 1085377743 DHCPDISCOVER(enp1s0f0) 00:40:bf:06:13:02 no address available
```

This implies the SNAP has a MAC address of `00:40:bf:06:13:02` (yours will be different). 4. Go back and uncomment and edit the `dhcp-host` line of `/etc/dnsmasq.conf` to contain this MAC.
For example, in this case we would put `dhcp-host=00:40:bf:06:13:02,192.168.0.3,snap` 5. Finally, restart the dhcp server with `sudo systemctl restart dnsmasq`

After waiting a bit for the SNAP to send a new request for a DHCP lease, look at the latest logs again from journalctl. If it ends with something like

```
Aug 16 14:43:02 grex-caltech-ovro dnsmasq-dhcp[6024]: 1085377743 DHCPREQUEST(enp1s0f0) 192.168.0.3 00:40:bf:06:13:02
Aug 16 14:43:02 grex-caltech-ovro dnsmasq-dhcp[6024]: 1085377743 tags: known, enp1s0f0
Aug 16 14:43:02 grex-caltech-ovro dnsmasq-dhcp[6024]: 1085377743 DHCPACK(enp1s0f0) 192.168.0.3 00:40:bf:06:13:02 snap
Aug 16 14:43:02 grex-caltech-ovro dnsmasq-dhcp[6024]: 1085377743 requested options: 1:netmask, 3:router, 28:broadcast, 6:dns-server
Aug 16 14:43:02 grex-caltech-ovro dnsmasq-dhcp[6024]: 1085377743 next server: 192.168.0.1
Aug 16 14:43:02 grex-caltech-ovro dnsmasq-dhcp[6024]: 1085377743 sent size:  1 option: 53 message-type  5
Aug 16 14:43:02 grex-caltech-ovro dnsmasq-dhcp[6024]: 1085377743 sent size:  4 option: 54 server-identifier  192.168.0.1
Aug 16 14:43:02 grex-caltech-ovro dnsmasq-dhcp[6024]: 1085377743 sent size:  4 option: 51 lease-time  1h
Aug 16 14:43:02 grex-caltech-ovro dnsmasq-dhcp[6024]: 1085377743 sent size:  4 option: 58 T1  30m
Aug 16 14:43:02 grex-caltech-ovro dnsmasq-dhcp[6024]: 1085377743 sent size:  4 option: 59 T2  52m30s
Aug 16 14:43:02 grex-caltech-ovro dnsmasq-dhcp[6024]: 1085377743 sent size:  4 option: 28 broadcast  192.168.0.255
Aug 16 14:43:02 grex-caltech-ovro dnsmasq-dhcp[6024]: 1085377743 sent size:  4 option:  1 netmask  255.255.255.0
Aug 16 14:43:02 grex-caltech-ovro dnsmasq-dhcp[6024]: 1085377743 sent size:  4 option:  3 router  192.168.0.1
```

That means the SNAP got an IP. You should now be able to `ping 192.168.0.3` to make sure it's alive.

### Advanced 10 GbE Settings

Unfortunately, the OS's default configuration for the 10 GbE network card is not optimized for our use-case of streaming time domain science data. As such, we need to adjust a few things.

Create the file `/etc/sysctl.d/20-grex.conf` with the following contents:

```conf
kernel.shmmax = 68719476736
kernel.shmall = 4294967296
net.core.rmem_max = 536870912
net.core.wmem_max = 536870912
net.core.optmem_max = 16777216
vm.swappiness=1
```

Then apply these changes with

```sh
sudo sysctl --system
```

Now, we need a program called `ethtool` to apply some more settings

```sh
sudo apt-get install ethtool -y
```

Now we will create a file to run on boot to apply a sequence of ethtool settings.

Create the file `/etc/rc.local` with the following contents:

```sh
#!/bin/env bash
ethtool -G enp1s0f0 rx 4096 tx 4096
ethtool -A enp1s0f0 rx on
ethtool -A enp1s0f0 tx on
```

Make this file executable with

```sh
sudo chmod +x /etc/rc.local
```

Now create the file `/etc/systemd/system/rc-local.service` with the following contents:

```ini
[Unit]
 Description=/etc/rc.local Compatibility
 ConditionPathExists=/etc/rc.local

[Service]
 Type=forking
 ExecStart=/etc/rc.local start
 TimeoutSec=0
 StandardOutput=tty
 RemainAfterExit=yes
 SysVStartPriority=99

[Install]
 WantedBy=multi-user.target
```

Then enable the `rc-local` service

```sh
sudo systemctl enable rc-local
```

Finally, reboot

## GPU Drivers / CUDA

Heimdall (the pulse detection part of our pipeline) relies on the CUDA toolkit. Let's install that now (version 12.3)

```sh
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
sudo apt-get -y install cuda-toolkit-12-3
```

And then install the open-source kernel drivers (Version 545) (overwriting previously installed ones)

```sh
sudo apt-get install -y nvidia-kernel-open-545
sudo apt-get install -y cuda-drivers-545
```

You may want to run the following to cleanup old dependencies/driver versions that may have been preinstalled

```sh
sudo apt-get autoremove
```

Reboot. Then run `nvidia-smi` to see if the CUDA version and Driver version came up correctly.

Finally, add the following to `~/.bashrc` to let our use use CUDA

```sh
# CUDA 12.3
export PATH=/usr/local/cuda-12.3/bin${PATH:+:${PATH}}
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda-12.3/lib64
```

and `source ~/.bashrc` or relog.

!!! warning

    If you change CUDA versions, you'll need to update these paths!

## Pipeline Dependencies

### PSRDADA

We use [PSRDADA](https://psrdada.sourceforge.net/) to connect the packet capture and first stage processing pipeline [T0](https://github.com/GReX-Telescope/GReX-T0) to the pulse detection framework [heimdall](https://sourceforge.net/p/heimdall-astro/wiki/Home/). This is a library we need to install.

We will build a few programs, so might as well create a directory to do this in to keep our home directory organized.

```sh
mkdir src && cd src
```

Then clone PSRDADA

```sh
git clone git://git.code.sf.net/p/psrdada/code psrdada && cd psrdada
# Last tested version, bump as appropriate
git checkout 008afa7
```

Now, install some build dependencies

```sh
sudo apt-get install build-essential cmake ninja-build -y
```

Then, build PSRDADA

```sh
mkdir build && cd build
cmake -GNinja ..
ninja
sudo ninja install
```

This will install the control programs and libraries to `/usr/local/bin` and `/usr/local/lib`, respectively.

We have to add the latter to out linker path, by adding the following to `~./bashrc`

```sh
# PSRDADA
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib
```

Then, relog once agian.

### Heimdall

Similar process to build the pulse-detection software, heimdall.
First, clone our fork in our `~/src` directory:

```sh
git clone --recurse-submodules  https://github.com/GReX-Telescope/heimdall-astro
cd heimdall-astro
```

Install some build dependencies

```sh
sudo apt-get install libboost-all-dev -y
```

Then build

```sh
mkdir build && cd build
cmake -GNinja ..
ninja
```

Run the test dedispersion program to make sure everything seemed to work

```sh
./dedisp/testdedisp
```

which should return

```
----------------------------- INPUT DATA ---------------------------------
Frequency of highest chanel (MHz)            : 1581.0000
Bandwidth (MHz)                              : 100.00
NCHANS (Channel Width [MHz])                 : 1024 (-0.097656)
Sample time (after downsampling by 1)        : 0.000250
Observation duration (s)                     : 30.000000 (119999 samples)
Data RMS ( 8 bit input data)                 : 25.000000
Input data array size                        : 468 MB

Embedding signal
----------------------------- INJECTED SIGNAL  ----------------------------
Pulse time at f0 (s)                      : 3.141590 (sample 12566)
Pulse DM (pc/cm^3)                        : 41.159000
Signal Delays : 0.000000, 0.000008, 0.000017 ... 0.009530
Rawdata Mean (includes signal)    : -0.002202
Rawdata StdDev (includes signal)  : 25.001451
Pulse S/N (per frequency channel) : 1.000000
Quantizing array
Quantized data Mean (includes signal)    : 127.497818
Quantized data StdDev (includes signal)  : 25.003092

Init GPU
Create plan
Gen DM list
----------------------------- DM COMPUTATIONS  ----------------------------
Computing 32 DMs from 2.000000 to 102.661667 pc/cm^3
Max DM delay is 95 samples (0 seconds)
Computing 119904 out of 119999 total samples (99.92% efficiency)
Output data array size : 14 MB

Compute on GPU
Dedispersion took 0.02 seconds
Output RMS                               : 0.376464
Output StdDev                            : 0.002307
DM trial 11 (37.681 pc/cm^3), Samp 12566 (3.141500 s): 0.390678 (6.16 sigma)
DM trial 11 (37.681 pc/cm^3), Samp 12567 (3.141750 s): 0.398160 (9.41 sigma)
DM trial 11 (37.681 pc/cm^3), Samp 12568 (3.142000 s): 0.393198 (7.25 sigma)
DM trial 11 (37.681 pc/cm^3), Samp 12569 (3.142250 s): 0.391713 (6.61 sigma)
DM trial 12 (40.926 pc/cm^3), Samp 12566 (3.141500 s): 0.441719 (28.29 sigma)
DM trial 13 (44.171 pc/cm^3), Samp 12564 (3.141000 s): 0.400574 (10.45 sigma)
DM trial 13 (44.171 pc/cm^3), Samp 12565 (3.141250 s): 0.403097 (11.55 sigma)
Dedispersion successful.
```

Finally, install this into our path with

```sh
sudo ninja install
```

The `heimdall` executable should now be available for the pipeline as well as offline analysis.

### HDF5/NetCDF

We use the fantastic [netcdf](https://www.unidata.ucar.edu/software/netcdf/) library to write out voltage
data. To do this, we need to statically link netcdf (and it's dependent HDF5) at compile time. As such, we
have to install these libraries systemw-wide.

```sh
sudo apt-get install libhdf5-dev libnetcdf-dev -y
```

## Rust

Many parts of the pipeline software are written in the Rust programming language.
We will be building this software from scratch, so we need to install the rust compiler and its tooling.
This is easy enough with [rustup](https://rustup.rs/)

We need curl for the rustup installer, so

```sh
sudo apt-get install curl -y
```

Then run the installer, using all the default settings

```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

## Python

To get out of version hell for python stuff, we're using [Poetry](https://python-poetry.org/). To install it, we will:

```sh
curl -sSL https://install.python-poetry.org | python3 -
```

We need to make a few adjustments to `~/.bashrc` to correct the paths and fix a bug. Append the following to the end.

```sh
export PATH="/home/user/.local/bin:$PATH"
# Fix the "Poetry: Failed to unlock the collection" issue
export PYTHON_KEYRING_BACKEND=keyring.backends.null.Keyring
```

Go ahead and `source ~/.bashrc` now to get these changes in your shell.

## Pipeline Software

To organize all the software needed for running the whole pipeline, we will grab the metapackage from github and clone somewhere (like the home directory):

```sh
cd
git clone --recurse-submodules https://github.com/GReX-Telescope/grex
```

Then, assuming you followed all the previous steps, build the pipeline software with

```sh
./grex/build.sh
```

Lastly, you'll need to install the `parallel` package

```sh
sudo apt install parallel -y
```

## Databases and Metrics Collection

TODO: Really this should all be orchestrated with something like docker compose.

### Prometheus

To save data about the server (CPU usage, RAM usage, etc) and to collect monitoring metrics from various pieces of pipeline software, we use the [prometheus](https://prometheus.io/) time series database. Each server will host its own database and _push_ updates to the monitoring frontend Grafana.

First, create a new group and user

```sh
sudo groupadd --system prometheus
sudo useradd -s /sbin/nologin --system -g prometheus prometheus
```

Next, we create the path for the database. Puget setup the system so the large storage drive is mounted at `/hdd`. We will keep the database there.

```sh
sudo mkdir /hdd/prometheus
```

Prometheus primary configuration files directory is /etc/prometheus/. It will have some sub-directories:

```sh
for i in rules rules.d files_sd; do sudo mkdir -p /etc/prometheus/${i}; done
```

Next, get a copy of the prometheus binaries, extract, and move into that dir

```sh
mkdir -p /tmp/prometheus && cd /tmp/prometheus
curl -s https://api.github.com/repos/prometheus/prometheus/releases/latest | grep browser_download_url | grep linux-amd64 | cut -d '"' -f 4 | wget -qi -
tar xvf prometheus*.tar.gz
cd prometheus*/
```

Install the files by moving to `/usr/local/bin`

```sh
sudo mv prometheus promtool /usr/local/bin/
```

Move Prometheus configuration files to `etc`

```sh
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo mv consoles/ console_libraries/ /etc/prometheus/
```

Now, we configure. Open up `/etc/prometheus/prometheus.yml` and edit to contain:

```yml
global:
  scrape_interval: 10s
  evaluation_interval: 10s
  external_labels:
    origin_prometheus: <some unique identifer>
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090", "localhost:9100", "localhost:8083"]
remote_write:
  - url: <grafana-url>
    basic_auth:
      username: <grafana username>
      password: <grafana api key>
```

If you are hooking up to our grafana instance, you will get an API key from the project, otherwise you'd create a `remote_write` section that reflects your monitoring stack.

Now, create a systemd unit to run the database in the file `/etc/systemd/system/prometheus.service`

```ini
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP \$MAINPID
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/hdd/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.external-url=

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
```

Now update all the permissions of the things we've mucked with to make sure prometheus can use them

```sh
for i in rules rules.d files_sd; do sudo chown -R prometheus:prometheus /etc/prometheus/${i}; done
for i in rules rules.d files_sd; do sudo chmod -R 775 /etc/prometheus/${i}; done
sudo chown -R prometheus:prometheus /hdd/prometheus/
```

Finally, reload systemd and start the service

```sh
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
```

Now we will install the node-exporter, which gives us metrics of the computer itself.

```sh
sudo apt-get install prometheus-node-exporter
```

### OpenTelemetry

TODO

## Pi SSH

One nice last thing will be to configure easy access to the Raspberry Pi's SSH. We can do that by creating a passwordless SSH key and installing it on the pi. We're going to be behind ssh anyway, and the Pi isn't public-facing, so this is a safe thing to do.

On the server, generate a new ssh keypair with

```sh
ssh-keygen -t rsa
```

Then, install it on the pi with

```sh
ssh-copy-id pi@192.168.0.2
```

Finally, create an SSH config that automatically supplies the hostname and user:

Create a file on the GReX server in `~/.ssh/config` with the contents

```
Host pi
    Hostname 192.168.0.2
    User pi
```

Now you can test the easy connection with

```sh
ssh pi
```

All done! We should now be ready to run the pipeline!
