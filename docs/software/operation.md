# Operation

We'll flesh out all the details on operation soon, but in the meantime here are the critical details.

## Turning on the SNAP

To turn on the SNAP, SSH into the Pi (as discussed in the server setup), Then on the pi create (if it
doesn't already exist) a bash script called `snap.sh` with the following:

!!! warning

    This is assuming you have a V2 power supply, ON and OFF are reversed otherwise

```bash
#!/bin/env bash
# Usage: ./snap.sh <on|off>
BASE_GPIO_PATH=/sys/class/gpio
PWN_PIN=20
if [ ! -e $BASE_GPIO_PATH/gpio$PWN_PIN ]; then
  echo "20" > $BASE_GPIO_PATH/export
fi
echo "out" > $BASE_GPIO_PATH/gpio$PWN_PIN/direction
if [[ -z $1 ]];
then
    echo "Please pass `on` or `off` as an argument"
else
    case $1 in
    "on" | "ON")
    echo "0" > $BASE_GPIO_PATH/gpio$PWN_PIN/value
    ;;
    "off" | "OFF")
    echo "1" > $BASE_GPIO_PATH/gpio$PWN_PIN/value
    ;;
    *)
    echo "Please pass `on` or `off` as an argument"
    exit -1
    ;;
    esac
fi
exit 0
```

Make it executable is `chmod +x snap.sh`, and use `./snap.sh <on|off>` to control the power state of the SNAP.

If you get permission errors when doing so, make sure your Pi's user is a member of the `gpio` group.
You can add your user to the group with:

```bash
sudo usermod -a -G gpio $(whoami)
```

then relog and try again.

## Running the Pipeline

In the `grex` folder, under `pipeline` there is the single bash script that runs the pipeline.
Simply calling it `./grex.sh` should start everything up. By default, it will run the normal detection pipeline.
If you want to just run T0 (packet capture and first stage processing), remove the final line that calls `psrdada` and replace it (the whole line) with `filterbank`.

## Triggering voltage dumps

Normally, T2 will send triggers to T0 to dump the voltage ringbuffer to disk. You can emulate this by sending a UDP packet to the trigger socket any other way. One simple way is with bash

```bash
echo " " > /dev/udp/localhost/65432
```

## SSH Port Tunneling

In some circumstances, it may be useful to access ports on the GReX server on your local computer remotely. We can accomplish this using [SSH Tunneling](https://www.ssh.com/academy/ssh/tunneling-example).

One example of why we might want to do this is to access the 10 GbE switch configuration that is located in the
far-side GReX box. It runs a normal web page on a static ip of `192.168.88.1`. You can access this from a web
browser if you are sitting at the GReX server, but not remotely.

To access it using SSH tunneling, we can forward that IP's port 80 (standard HTTP) to our local computer at some
unused, non-privaleged port.

```shell
ssh -L 8080:192.168.88.1:80 username@grex-server-address
```

Another example is perhaps you want to run a [Jupypter Hub](https://jupyter.org/hub) instance on the GReX server. In that case, the website it is hosting is on the server itself, so you would run:

```shell
ssh -L 8080:localhost:80 username@grex-server-address
```

Another useful one is access to the Prometheus time-series database used for monitoring. That is active on port `9090`

```shell
ssh -L 9090:localhost:9090 username@grex-server-address
```

## Pulse Injection

If you need to generate and inject fake pulses into raw voltages to test the pipeline, Liam Connor's [injection codes](https://github.com/liamconnor/injectfrb/blob/master/injectfrb/simulate_frb.py) contain all the relevant tools. In a Python Jupyter notebook, you can import this Python script and use the functions within to generate a fake pulse and write it to an output .dat file.

```python
import simulate_frb
dt = 8.192e-6
width_sec = 2*dt # depending on your preferred pulse width
Nfreq = 2048
data, params = simulate_frb.gen_simulated_frb(NFREQ=Nfreq,
    NTIME=16384,
    sim=True,
    fluence=70, # This controls the pulse SNR
    spec_ind=0.0,
    width=width_sec,
    dm=100., # dispersion measure
    background_noise=np.zeros([2048, 16384]),
    delta_t=dt,
    plot_burst=False,
    freq=(1530.0, 1280.0),
    FREQ_REF=1405.0,
    scintillate=False,
    scat_tau_ref=0.0,
    disp_ind=2.0,
    conv_dmsmear=False,
)
```

Before writing to an output file, you might want to check what the pulse looks like and its histogram distribution, ensuring it’s not all zeros due to conversion to int8 (bit type of raw voltages).

```python
from matplotlib import pyplot as plt
plt.hist(data.astype(np.int8).flatten(), bins=50)
plt.semilogy()
plt.show()
### also, visualize the pulse itself
plt.imshow(data.astype(np.int8), aspect='auto')
plt.colorbar()
plt.show()
```

If the pulse looks reasonable, we can convert to `int8` type and write to an output .dat file.
```python
with open('test_injpulse0.dat', 'wb') as f:
    f.write(data.astype(np.int8).tobytes())
```

Move this .dat file to `~/grex/pipeline/fake/`, the specified directory in `grex.sh`, to actually inject the pulse. Then you could change the injection cadence (in seconds) in `~/grex/pipeline/.env` by adding the following line.

```shell
injection_cadence=300
```

Now you are good to go.

## Querying Historical Data

As part of the server setup, we've hooked up a time-series database (Prometheus) to T0, which is used to monitor the state of the telescope.
This includes recording timestamped values for temperature, ADC counts, and integrated Stokes I data.
It may be useful to query this data, which is relativly easy to do programatically using Prometheus' [HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/).

You can explore the database by accessing `http://localhost:9090` on the server (or with SSH-forwarding described above).

### H1 Example

Say you want to query historical Stokes I data around the H1 frequency of 1420 MHz.
This data is stored in "channels" where the index is the FPGA's channel number.
If you recall, we operate in the first Nyquist zone, so the spectrum is flipped, where channel 0 represents 1530 MHz and channel 2048 represents 1280 MHz.

Using the `requests` library in Python, we can perform the HTTP request to the database.
```python
import requests
import time
import numpy as np

# Get the current (UNIX) time
now = time.time()
# We want data, say two weeks back
past = now - 60 * 60* 24 * 7 * 2

# We want data around 1420 MHz, so around channel 893
# Might as well grab the block of data around it, so we can watch it change in frequency with time
channels = [889, 890, 891, 892, 893, 894, 895]
data = []
uri_base ='http://127.0.0.1:9090/api/v1/query_range'
for channel in channels:
    params = {"query": f'spectrum{{channel="{channel}"}}',
              "start": then,
              "end": now,
              "step": "10m"} # Depending on the timespan, there's a maximum size per query
    resp = requests.get(uri_base, params=params)
    # Extract the data  and convert to floats
    vals = resp.json()['data']['result'][0]['values']
    data.append(np.array(vals).astype(float))

# Restructure the data into a tensor
data = np.stack(data)

# The time axis is duplicated many times, we can extract it by looking at one chunk
timestamps = data[0,:,0]

# And then finally extract our 2D block of time/freq linear power data
# Indexed in [channel, time]
spectra = data[:,:,1]
```
