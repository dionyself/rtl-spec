RTL-Spec
========
*rtl-spec* is an open source implementation of the proposals done by D. Pfammatter, D. Giustiniano and V. Lenders in **"A Software-defined Sensor Architecture for Large-scale Wideband Spectrum Monitoring"** [[IPSN15](http://dx.doi.org/10.1145/2737095.2737119)].

# Installation
The following installation instructions are currently only tested for Debian-based Linux operating systems (such as Ubuntu or Raspbian).

## Dependencies
### libusb
[libusb](http://www.libusb.org) is a C library that gives applications easy access to USB devices on many different operating systems. *rtl-spec* uses the library to interface [DVB-T USB dongles](http://www.rtl-sdr.com/buy-rtl-sdr-dvb-t-dongles) which are used in this project as cheap RF interfaces.
```sh
$ sudo apt-get install libusb-1.0-0-dev
```

### librtlsdr
[librtlsdr](http://sdr.osmocom.org) is a C library that turns RTL2832 based DVB-T dongles into cheap SDR receivers.
```sh
$ git clone git://git.osmocom.org/rtl-sdr.git
$ cd rtl-sdr/
$ mkdir build && cd build/
$ cmake ../ -DINSTALL_UDEV_RULES=ON
$ make
$ sudo su
$ make install
$ ldconfig
$ cat > /etc/modprobe.d/rtl-blacklist.conf << EOL
$ blacklist dvb_usb_rtl28xxu
$ blacklist rtl2832
$ blacklist rtl2830
$ EOL
$ rmmod dvb_usb_rtl28xxu rtl2832
$ exit
$ cd ../../
```

### fftw
[fftw](http://www.fftw.org) is a C library for computing the discrete Fourier transform (DFT).
```sh
$ sudo apt-get install fftw-dev
```

## RTL-Spec
#### Building
The software can be built as follows:
```sh
    make <TARGET> [CFLAGS="<CFLAGS>"]
    <TARGET> = sensor_cpu | sensor_gpu | collector
    <CFLAGS> = [-O2] [-ggdb] [-DVERBOSE] [...]
```
First, select the **target** you want to build. There are two options for building the sensor and a one for the collector. The target *sensor_gpu* compiles the sensor software for usage on dedicated hardware, i.e. the [Raspberry Pi](http://www.raspberrypi.org) (RPi). This will lead to some CPU intensive tasks, such as the FFT, being rolled out to the RPi's VideoCore IV GPU, improving overall sensing performance. Note that for FFT computations on the VideoCore IV we rely on the library [GPU_FFT](http://www.aholme.co.uk/GPU_FFT/Main.htm), which typically comes preinstalled on Raspbian OS. In case you don't want to compile the sensor software for dedicated hardware, select the target *sensor_cpu*. The FFT will then be computed on general purpose CPUs using the [FFTW](http://http://www.fftw.org) library.

Second, you can choose [gcc](https://gcc.gnu.org)'s **compilation flags**. Compiling any of the targets with flag *-DVERBOSE* will provide additional debugging information on stdout.

An example for building a collector and sensor instance on the same machine is given below:
```sh
$ cd rtl-spec/
$ make collector CFLAGS="-O2 -DVERBOSE"
$ make sensor_cpu CFLAGS="-O2 -DVERBOSE"
```

#### Running
Here, we provide the simplest example for running the collector and sensor instances built above. The collector listens on localhost port *5000* for incoming sensor data and dumps it to the local file system. The sensor monitors the frequency spectrum between *24* and *1766MHz* and transmits the recorded samples to the afore mentioned collector.
```sh
$ ./run_collector 5000
$ ./run_cpu_sensor 24000000 1766000000
```
Run collector and sensor with option *-h* to learn more on the individual configuration options.
```sh
$ ./run_collector -h
$ ./run_cpu_sensor -h
```

# Description
## Processing Steps
Multiple spectrum sensing nodes can be served by a single (remote) collecting unit. The RF spectrum data recorded by sensors is transmitted over TCP to the associated collector which dumps the received data to the local file system. The following figure highlights the involved **processing steps**:

![alt text](https://github.com/pdamian/rtl-spec/blob/master/images/processing_steps.png "Processing Steps")

## Data Format
The dumped **data** is stored in the following format:

| Seconds since UNIX Epoch [secs] | Timestamp Extension [microsecs] | Frequency [Hz] | Squared Magnitude Value [dB] |
| ------------------------------- | ------------------------------- | -------------- | ---------------------------- |
| 1423490796                      | 854275                          | 23996876       | -33.9                        |
| 1423490796                      | 854275                          | 24006250       | -20.6                        |
| ...                             | ...                             | ...            | ...                          |

## Wideband Scanning Strategies
To overcome the sampling rate hardware limitations of low-cost radios (such as DVB-T USB dongles), the sensing software includes different **wideband scanning strategies**. Currently, you can choose from the following three options:
- *Sequential*: Sequentially sweep over the band of interest in steps limited by the sampling rate
- *Random*    : Inspect frequency bands (limited by the sampling rate) in a random fashion
- *Similarity*: Visit frequency bands of particular interest more frequently than others

For more information, please refer to **"A Software-defined Sensor Architecture for Large-scale Wideband Spectrum Monitoring"** [[IPSN15](http://dx.doi.org/10.1145/2737095.2737119)].
