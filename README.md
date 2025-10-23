
<img alt="Tigard PCB Render" width="320" align="right" src="https://raw.githubusercontent.com/tigard-tools/tigard/master/tigard-render.png">

# Tigard

An FTDI FT2232H-based multi-protocol tool for hardware hacking.

# Objectives

There are plenty of -232H series breakout boards, but they are generally designed to be an easy way to adapt it to a specific use, and not designed for regularly plugging in to all different target systems.

The two exceptions are the Exodus Intelligence Hardware Interface Board which is not open hardware or commercially available, and [TIMEP](https://github.com/Matir/timep) which is the origin and heritage of this project.

# Contents
- [Software Features](#software)
- [Hardware Features](#hardware-features)
- [Usage](#usage)
  - [Hardware Hookup](#hardware-hookup)
  - [Switches](#switches)
  - [UART](#uart)
  - [SPI](#spi)
  - [I2C (on I2C or SPI header)](#i2c-on-i2c-spi-or-jtag-header)
  - [JTAG Debug (on JTAG or CORTEX header)](#jtag-debug-on-jtag-or-cortex-header)
  - [JTAG Boundary Scan (on JTAG or CORTEX header)](#jtag-boundary-scan-on-jtag-or-cortex-header)
  - [SWD (on CORTEX header)](#swd-on-cortex-header)
  - [iCE40 Programming](#ice40-programming)
  - [AVR ISP](#avr-isp)
- [Debugging](#debugging)
- [Pinouts](#pinouts)
- [Stripes and Tails](#stripes-and-tails)
- [Dens (cases)](#dens)
- [Serial Numbers](#serial-numbers)
- [Getting Tigard](#getting-tigard)

# Software Features

In general, Tigard was designed to work as-is with several tools and libraries that already support the x232H family of chips. This includes:

* USB-Serial drivers for UART access
* OpenOCD and URJTAG for JTAG
* Flashrom, libmpsse, pyftdi and other tools for SPI interfaces
* libmpsse and pyftdi for I2C interfaces
* iceprog for ice40 FPGAs
* avrdude for AVR microcontrollers

Also, users have used Tigard with [BMDA](/../../issues/66), [Mitou jtag](/../../issues/79), [VSCode](/../../pull/75), and [SPI via FTDI](/../../issues/40)

# Hardware Features

Highlights:

* Dual-port, with one dedicated to UART and the second shared with other interfaces
* High-performance directional level shifters for 1.8 to 5.5v operation
* Switch to choose between on-board 1.8, 3.3, and 5.0v supplies and vTarget
* Switch to choose between SPI/JTAG and I2C/SWD modes
* Logic analyser port to observe device-level signals (v1.0 and later)
* Indicator lights to aid debugging

# Usage

## Hardware Hookup

Starting with the board completely disconnected:

1. Connect board to target system with clips or jumper wires
1. Select the correct mode either SPI/JTAG or SWD/I2C
1. Ensure that the voltage switch is in VTGT mode
1. Plug in the USB cable. The PWR and EN LEDs will illuminate
1. Power on your target.
1. If you connected VTGT to your target, the VTGT LED will illuminate. If not, you can now select your voltage with the voltage switch

## Switches
There are two switches on Tigard to set the mode of operation. One controls voltage, one controls how several of the IO pins are wired. Both need to be set properly for a specific use mode. Details are in each of the interface sections further below, but here is a summary of the modes and their uses:

#### Voltage Switch

This switch chooses the reference voltage for the level shifters and the target system:
* 1V8, 3V3, and 5V all apply a voltage to the VTGT pin.
* VTGT disconnects the VTGT pin from supplies, and depends on the wire connected to the target to set the level shifter voltage.

This results in 3 distinct use cases:
1. **Target-Powered:** Set the switch to VTGT and connect the VTGT wire to the **powered** target. The **target** powers the level shifters.
1. **Tigard-Powered:** Set the switch to a voltage, and connect the VTGT to the **unpowered** target. **Tigard** supplies power to the target.
1. **Self-Powered:** Set the switch to a voltage, but do NOT connect the VTGT wire. **Tigard** powers its own level shifters. **Target** powers itself.

#### Mode Switch

This switch controls how some of the I/O pins are connected for specific uses:

* When set to JTAG/SPI mode:
  * All signals run straight through
  * CORTEX and JTAG headers are wired for normal JTAG
  * JTAG and SPI headers can be used for SPI
* When set to SWD/I2C mode:
  * DO and DI are wired together to create SWDIO or SDA
  * CORTEX and JTAG headers can be used for SWD
  * JTAG and SPI headers can be used for I2C

## UART

#### Hookup:

* In most cases, you need pins 2,3,and 4 of the UART header - Ground, TX, and RX. Connect these to your target
* If you have a VCC header on your target, connect pin 1 - VTGT to it, and slide the voltage selector to VTGT
* If you don't have a VCC header, leave the VTGT wire disconnected, and choose your target voltage with the selector

#### Software:

The first of the two ports is connected to the UART header. When you plug Tigard in, you will see two serial devices show up - the first one is the one you want. Start your software using the appropriate serial port. For example:

```bash
screen /dev/ttyUSB0 115200
```

## SPI

#### Hookup:

The SPI/I2C header is laid out to be the same orientation as the pins on a standard 8-pin SPI flash chip, making it easy to attach clips or sockets.

* Select SPI/JTAG on the mode selection switch
* Connect your clip or socket to the header. Pay attention to pin 1, which is usually marked on sockets or has a red/highlighted wire on ribbon cables
* Connect your clip to the target or insert your chip into the socket
* If you're doing this in-circuit, you *must* take extra precautions to make sure no other device is communicating with the SPI device.
* If you're using a clip in-circuit and the target is powered on, choose the VTGT option on the voltage slider
* If you're using a zif socket, or clip with a powered-off target, choose the voltage on the voltage slider. 

#### Software:

flashrom is the most common tool for SPI flash dumps. However, while pervasive, it is very slow and inefficient.

```bash
flashrom -p ft2232_spi:type=2232H,port=B,divisor=4
```

libmpsse is a powerful library for controlling the MPSSE, or high speed serial pins of the x232H series. However, it is no longer recommended because of a large number of dependencies

pyftdi is a new and simple interface very similar to libmpsse:

```python
from pyftdi.ftdi import Ftdi
Ftdi.show_devices()
from os import environ
ftdi_url = environ.get('FTDI_DEVICE', 'ftdi://ftdi:2232:1:23f/2')
from spiflash.serialflash import SerialFlashManager
flash=SerialFlashManager.get_flash_device(ftdi_url)
print("Flash device: %s @ SPI freq %0.1f MHz" % (flash, flash.spi_frequency/1E6))
f=open("data.bin","wb")
f.write(flash.read(0,len(flash)))
f.close()
```

## I2C (on I2C, SPI, or JTAG header)

#### Hookup:

The I2C header (on hardware versions 1.0 and later) is compatible with [Sparkfun's Qwiic](https://www.sparkfun.com/qwiic) and [Adafruit's STEMMA QT](https://learn.adafruit.com/introducing-adafruit-stemma-qt/what-is-stemma-qt) system:

* Connect a 4-pin JST SH compatible cable into the I2C connector
* Connect the other end to your STEMMA QT, Qwiic, or other I2C device

In addition, the SPI header is laid out to be the same orientation as the pins on most 8-pin I2C chips, making it easy to attach clips or sockets:

* Connect your clip or socket to the header. Pay attention to pin 1, which is usually marked on sockets or has a red/highlighted wire on ribbon cables
* Connect your clip to the target or insert your chip into the socket

Finally, the JTAG header can also be used to get the I2C signals on .1" individual pins:

* Connect the 8-wire JTAG harness to the JTAG connector
* Connect the white SCL and grey SDA wires to your target's I2C pins

In either case, you need to set the Tigard switches properly:

* Select I2C on the mode selection switch
* If you're attaching to a device in-circuit and the target has its own power, choose the VTGT option on the voltage slider
* If you're attaching a standalone sensor, socket, or clip with a powered off target, choose the correct voltage on the voltage slider

#### Quirks:

The FT2232H has a very limited I2C implementation. I2C depends on shared I/O lines using open collector/open drain instead of push-pull-tristate I/O, but the FT2232H doesn't support that well. Therefore:

* Only controller operation is supported, not Device
* Tigard may not play nice if there are other controllers present on the I2C interface
* Clock stretching is not supported
* The I2C switch ties the DI and DO lines together so that it can do bidirectional communication
* The pullup resistors are not included since they are usually located on the target, and the weak pullups on the level shifters are sufficient

#### Hacks:

To accomodate both I2C and SWD, the DI and DO lines are combined through resistor R16. This is required for SWD and generally acceptable for I2C. 

Many I2C targets already have pullup resistors. In addition, all of Tigard's I/O pins have a weak 100K ohm pullup. In testing, this has been sufficient for both in-circuit and external use of most I2C devices. If you need stronger pullups on I2C, you can temporarily add them by pulling up COPI and SCK on the SPI header, or TCK and TDI on the JTAG header.

#### Software:

libmpsse is a powerful library for controlling the MPSSE, or high speed serial pins of the x232H series. However, it is no longer recommended because of a large number of dependencies

pyftdi is a new and simple interface very similar to libmpsse:

```python
from pyftdi.ftdi import Ftdi
Ftdi.show_devices()
from os import environ
ftdi_url = environ.get('FTDI_DEVICE', 'ftdi://ftdi:2232:1:23f/2')
from i2cflash.serialeeprom import SerialEepromManager
flash = SerialEepromManager.get_flash_device(ftdi_url,'24AA32A',0x50)
flash.write(5,10)
flash.read(0,32)
```

## JTAG Debug (on JTAG or CORTEX header)

#### Hookup:

The JTAG header is laid out with pins in the same order as the FTDI I/O pins are labeled, in order to be consistent with many other x232H breakout boards.

In additon, the CORTEX header is also wired as a standard ARM 10-pin JTAG header.

Be sure to select JTAG on the mode selection switch. This makes sure that TDI and TDO are separated, and ensures that TMS is wired properly to the CORTEX header. Otherwise, the standard hookup sequence applies.

#### Software:

**NOTE: These OpenOCD configs require OpenOCD v0.12+**. Previous versions may use the now-deprecated commands last updated in [this commit](https://github.com/tigard-tools/tigard/commit/5e5722944c8c1ec73b0fa85d574793300c23b3ce).

OpenOCD is a powerful tool for On-Chip Debugging of ARM, MIPS, and some other architectures.

The appropriate configuration file (make this a link to the file) should look like:

```
adapter driver ftdi
transport select jtag
ftdi vid_pid 0x0403 0x6010
ftdi channel 1
adapter speed 2000
ftdi layout_init 0x0038 0x003b
ftdi layout_signal nTRST -data 0x0010
ftdi layout_signal nSRST -data 0x0020
```

To use it with openocd:

```bash
openocd -f tigard-jtag.cfg
```

## JTAG Boundary Scan (on JTAG or CORTEX header)

Boundary scan can be used to take control of a device to set I/O pin state (`EXTEST`), or to view the state of them during device operation (`SAMPLE`).

#### Hookup:

The hookup is the same as described in section [JTAG Debug (on JTAG or CORTEX header)](#jtag-debug-on-jtag-or-cortex-header).

The pinout is compatible with the `Olimex ARM-USB-OCD` running on the `B` interface.

#### Software:

[UrJTAG](https://urjtag.sourceforge.io/) is open-source jtag control software geared toward low-level and boundary scan use.

We need to tell UrJTAG Tigard's VID and PID, as well as the fact that JTAG is on interface 1 (default is 0). Due to a bug it seems that TCK is miscalculated and is actually running 5 times what you select - 500khz in the case below.

```
jtag> cable ft2232 vid=0x403 pid=0x6010 interface=1
Connected to libftdi driver.
jtag> frequency 100000
Setting TCK frequency to 100000 Hz
jtag> detect
IR length: 5
Chain length: 1
Device ID: 00010110001101010010001001001111 (0x1635224F)
  Unknown manufacturer! (00100100111) (/usr/share/urjtag/MANUFACTURERS)
jtag> 
```

The [TopJTAG](http://www.topjtag.com/) software is one of the easier to use options, but is commercial ($100) and Windows-only. This includes a "waveform view" that allows you to view arbitrary pin states as if you had a logic analyzer on the device while it is running. If using TopJTAG, set the following as the JTAG connection:

* `Connection`: Generic FTDI FT2232
* `Device`: Tigard V1.0 B
* `Static Pins`: Olimex ARM-USB-OCD
* `TCK Freq`: 2 MHz (adjust as needed)

An open-source option that supports Tigard is [JTAG Boundary Scanner](https://github.com/viveris/jtag-boundary-scanner) which offers a Windows GUI, but the backend library is cross-platform (however written in C). This library offers many features such as an ability to talk to SPI devices attached to the target chip, where the SPI interface is run using JTAG boundary scan (warning - slow!). 

Alpha Python bindings for this library are available in [PyJTAGBS](https://github.com/colinoflynn/pyjtagbs). Using Tigard from Python for checking the scan chain for example:

```python
from jtagbs import JTAGCore, JTAGBS

interface = JTAGCore()
jtag = JTAGBS(interface)

probes = jtag.list_available_probes()
print(probes)

jtag.open_probe('Tigard V1.0 B')
jtag.init_scanchain()

print(jtag.list_devids())
```

See the PyJTAGBS documentation for further examples of loading BSDL files required to toggle specific pins. The listed probes should be `'Tigard V1.0 A TG100065A'` and `'Tigard V1.0 B TG100065B'` if the correct drivers are loaded.

## SWD (on CORTEX header)

#### Hookup:

The SWD header a standard 10-pin header found on many SWD target boards. A short 'SWD' cable with the same header on both ends is the ideal way to hook up to most targets.

Be sure to select SWD on the mode selection switch. This connects the DI and DO pins with resistor R16 to make the bidirectional SWDIO pin, and connect it to pin 2 of the CORTEX header. Otherwise, the standard hookup sequence applies.

#### Software:

**NOTE: These OpenOCD configs require OpenOCD v0.12+**. Previous versions may use the now-deprecated commands last updated in [this commit](https://github.com/tigard-tools/tigard/commit/5e5722944c8c1ec73b0fa85d574793300c23b3ce).

OpenOCD is a powerful tool for On-Chip Debugging of ARM, MIPS, and some other architectures. In order to use it for SWD with Tigard, you'll need to build it from source. The [directions from the AND!XOR DC27 Badge](https://hackaday.io/project/164346-andxor-dc27-badge/log/166464-swd-all-the-things) cover it step-by-step.

The appropriate configuration file (make this a link to the file) should look like:

```
adapter driver ftdi
transport select swd
ftdi vid_pid 0x0403 0x6010
ftdi channel 1
adapter speed 2000
ftdi layout_init 0x0028 0x002b
ftdi layout_signal SWD_EN -data 0
ftdi layout_signal nSRST -data 0x0020
```

To use it with openocd:

```bash
openocd -f tigard-swd.cfg
```

## iCE40 Programming
The Lattice iCE40 family of FPGAs are popular for small scale projects because of their low cost and the availability of an open toolchain. While this is a *very* specific target, Tigard is well suited for programming devices since it has all the necessary pins readily available.

#### Hookup:
All the necessary pins are on the JTAG header. 
The bonus "!?" pin is not populated by default. The JTAG !? pin and the UART RX pin are shorted in case you need to flash an iCE40 but don't want to solder your Tigard. See the pinouts table below for details.

Be sure to select JTAG/SPI on the mode selection switch. This makes sure that COPI and CIPO are separated.

If you are planning on programming SRAM instead of FLASH:
* Swap COPI and CIPO
* Disconnect CS from your FLASH, but leave it attached to the iCE40.


#### Software:
To program your target's nonvolatile FLASH, use iceprog:
```
iceprog -I B yourbitstream.bin
```
If you prefer to directly program the ice40's SRAM, make sure you adjust your wiring and again use iceprog:
```
iceprog -I B -S yourbitstream.bin
```
Remember that if you choose to program the SRAM, your bitstream will be lost on power cycle.

## AVR ISP
To program AVR microcontrollers including many Arduino boards, use avrdude with a custom configuration file.

#### Hookup:
Be sure to select JTAG/SPI on the mode selection switch. Use SRST as your reset.

#### Software:
Avrdude is an open-source AVR flashing utility. Avrdude 7.1 and later have built-in support for Tigard - all you need to do is:
```
avrdude -c tigard
```

For older versions, create the following `tigard.conf` file in order to identify Tigard as an avrftdi-compatible device and specify which pins to use:
```
programmer
  parent "avrftdi"
  id = "tigard";
  desc = "Tigard interface board";
  usbdev = "B";
  sck = 0;
  mosi = 1;
  miso = 2;
  reset = 5;
;
```

Then, use `-C +tigard.conf` to add the configuration file, and `-c tigard` to indicate your programmer:
```
avrdude -C +tigard.conf -c tigard
```

# Debugging

#### PWR LED:

This will turn on when the board has USB power and should turn on immediately when the USB cable is connected.

* If it does *not*, then check your USB cable and power supply.
* If it comes *on then goes off*, is *dim*, or *flickers*, then you have likely shorted power or ground somewhere with your wires or your target.

#### EN LED:

This will turn on when the FTDI chip is operation and should turn on a moment after the usb power LED is on.

* If it does *not* come on, you may have a faulty USB cable, a power-only USB port or cable, or a bad FTDI chip

#### VTGT LED:

This will turn on when the level shifters are properly powered, either by your target or by the onboard level selector

* If you are in VTGT mode, and it does *not* come on, check your target power and your power and ground wiring to the target
* If you are selecting a voltage and it does *not* come on, you have likely shorted power or ground somewhere with your wires or target.
* There are corner cases where VTGT will get [latched to a voltage applied to an output pin](https://github.com/tigard-tools/tigard/issues/20). Establishing a habit of always starting with VTGT off will help avoid this.

#### ALL LEDS ON:

When all LEDS are on, then Tigard is probably working as intended. If you are still having trouble, there are a few possibilities. They are listed in order of likelihood, though it makes sense to test the easier cases first:

* It's a *wiring* issure. Make sure your wires are connected well. Confirm with a multimeter.
* It's a *target* issue. Try a different target to make sure your software is working properly
* It's a *software* issue. Try a different software tool or mode to make sure your board works properly
* It's a *protocol* issue. Observe your signals with a logic analyzer or oscilloscope to make sure they're correct
* It's a *hardware* issue. Try a different Tigard board to see if it persists
* It's a *tigard* issue. Try a different x232H board to see if it persists

# Pinouts

There are way too many choices of 'standard' pinouts for all of these interfaces. Pinouts were chosen for ease of use, specificially:

* If there's a well established and used standard, use it (SWD)
* If there's a predominant primary usage that would be easier, use it (SPI/I2C)
* If there are other common boards with pinouts, use them (UART and JTAG)

### UART

2.54mm pitch. 1x9 pin header.

This pinout prioritizes putting the FT2232H pins in sequential order - similar to many x232H breakout boards

The coloring of VTGT, GND, TX, and RX is intended to match most common usb-serial cables. The remaining wires are included, but not installed on the header, so users can optionally install them.

| Pin Number | UART Signal | Color | FT2232H Pin |
| ---------- | ----------- | ----- | ----------- |
| 1          | VTGT        | Red   | --          |
| 2          | GDD         | Black | --          |
| 3          | TX          | Green | AD0         |
| 4          | RX          | White | AD1         |
| 5          | RTS         | --    | AD2         |
| 6          | CTS         | --    | AD3         |
| 7          | DTR         | --    | AD4         |
| 8          | DSR         | --    | AD5         |
| 9          | DCD         | --    | AD6         |

### SWD

1.27mm pitch. 2x5 pin header.

This is a standard pinout. In order to accomodate both SWD and JTAG, the mode switch:

1. Combines DI and DO to create SWDIO for SWD mode
2. Selects between SWDIO and TMS for pin 2

| Pin Number | SWD Signal | FT2232H Pin | JTAG Signal | FT2232H Pin |
| ---------- | ---------- | ----------- | ----------- | ----------- |
| 1          | VTGT       | --          | VTGT        | --          |
| 2          | SWDIO      | BD1 BD2     | TMS         | BD3         |
| 3          | GND        | --          | GND         | --          |
| 4          | SWDCLK     | BD0         | TCK         | BD0         |
| 5          | GND        | --          | GND         | --          |
| 6          | NC         | --          | TDO         | BD2         |
| 7          | KEY        | --          | KEY         | --          |
| 8          | NC         | --          | TDI         | BD1         |
| 9          | GND        | --          | GND         | --          |
| 10         | nSRST      | BD5         | nSRST       | BD5         |

### JTAG

2.54mm pitch. 1x8 pin header.

This pinout prioritizes putting the FT2232H pins in sequential order - similar to many x232H breakout boards. 

In general, set the mode switch to SPI/JTAG mode when using this connector.

The coloring of the wiring harness is what [SecuringHardware.com](https://SecuringHardware.com/) used for their Adafruit FT232H wiring harness for several years. The colors were chosen because frequently black-brown-red-orange are used with logic analyzers in class, so unique colors were chosen for this wiring harness.

This header can also be used for I2C and SPI if the 8-pin header doesn't make sense in your application.

| Pin Number | JTAG Pin | Color  | FT2232H Pin |
| ---------- | -------- | ------ | ----------- |
| 1          | VTGT     | Red    | --          |
| 2          | GND      | Black  | --          |
| 3          | TCK      | White  | BD0         |
| 4          | TDI      | Grey   | BD1         |
| 5          | TDO      | Purple | BD2         |
| 6          | TMS      | Blue   | BD3         |
| 7          | TRST     | Green  | BD4         |
| 8          | SRST     | Yellow | BD5         |

### SPI

2.54mm pitch. 2x4 pin header.

This header is designed specifically to match the pinout of most 8-pin SPI and I2C chips, so that a jumper to a chip clip or a socket would align properly.

In order to accomodate both SPI and I2C, the mode switch:

1. Combines DI and DO to create SDA for I2C mode
2. Disconnects pin 2 in I2C mode for devices that use it as an address signal

| Chip Pin Number | Header Pin Number | SPI signal | FT2232H Pin |
| --------------- | ----------------- | ---------- | ----------- |
| 1               | 1                 | CS         | BD3         |
| 2               | 3                 | CIPO       | BD2         |
| 3               | 5                 | WP         | pullup      |
| 4               | 7                 | GND        | --          |
| 5               | 8                 | COPI       | BD1         |
| 6               | 6                 | SCK        | BD0         |
| 7               | 4                 | EN         | pullup      |
| 8               | 2                 | VTGT       | --          |

### LA

1.27mm pitch. 2x7 pin header.

The LA port makes it easier to hook up a logic analyzer to monitoring push-pull signals between Tigard and your target hardware. It's tested and working well with UART, SPI and JTAG, and generally works with I2C. 

While there is a driver allowing you to use the FT2232H as a makeshift logic analyzer, that's not the intended purpose - the expected use case is that you are using Tigard to communicate with a target, but need to debug that communication using an external logic analyzer.

The LA port may be used as a passive mechanical adapter although this usage is [not supported](https://github.com/tigard-tools/tigard/issues/20). For best results, power the Tigard board but disconnect VTGT from the target. Additional effort will likely be required to ensure signal integrity.

This header is specifically designed to connect directly to the Bitmagic logic analyzer, or you could use the Bitmagic logic analyzer wiring harness to connect to any other tool.

The 8 most interesting signals are connected - 6 from the JTAG/SWD/SPI/I2C port, and 2 from the UART port.

| Pin Number | Bitmagic Signal | FT2232 Pin | JTAG signal | SPI signal | SWD Signal | I2C Signal | UART Signal |
| ---------- | --------------- | ---------- | ----------- | ---------- | ---------- | ---------- | ----------- |
| 1          | xPB0            | BD0        | TCK         | SCK        | SWCLK      | SCL        |
| 2          | xPB1            | BD1        | TDI         | COPI       | SWDIO      | SDA        |
| 3          | xPB2            | BD2        | TDO         | CIPO       | SWDIO      | SDA        |
| 4          | xPB3            | BD3        | TMS         | CS         |            |
| 5          | xPB4            | BD4        | TRST        |
| 6          | xPB5            | BD5        | SRST        |
| 7          | xPB6            | AD0        |             |            |            |            | TX          |
| 8          | xPB7            | AD1        |             |            |            |            | RX          |
| 9          | GND             |
| 10         | GND             |
| 11         | xTRIG           |
| 12         | xCLK            |
| 13         | xTRIG2          |
| 14         | xIFCLK          |

### I2C

1mm pitch. 1x4 pin header. JST SH compatible.

This header is designed specifically to match [Sparkfun's Qwiic](https://www.sparkfun.com/qwiic) and [Adafruit's STEMMA QT](https://learn.adafruit.com/introducing-adafruit-stemma-qt/what-is-stemma-qt) system.

Set the mode switch to I2C/SWD mode when using this connector.

| Pin Number | I2C signal | FT2232H Pin |
| ---------- | ---------- | ----------- |
| 1          | GND        | ---         |
| 2          | VCC        | ---         |
| 3          | SDA        | BD1 and BD2 |
| 4          | SCL        | BD0         |

### iCE40
Use the JTAG header with the addition of the "!?" pin which is not populated by default.

| Pin Number | Label        | iCE40 Signal   | FT2232 Pin  |
|------------|--------------|----------------|-------------|
| 1          | VTGT         | VTGT           | ---
| 2          | GND          | GND            | ---
| 3          | TCK          | CLK            | BD0
| 4          | TDI/MOSI/SDA | MOSI/COPI      | BD1
| 5          | TDO/MISO/--  | MISO/CIPO      | BD2
| 6          | TMS/SS/--    | ---            | ---
| 7          | TRST/--/--   | CS/ICE_SS_B    | BD4
| 8          | SRST/--/--   | CRESET         | BD5
| 9          | !?           | CDONE          | BD6

JTAG !? pin and the UART RX pin are shorted in case you need to flash an iCE40 but don't want to solder your Tigard.

### AVR ISP
The common 6-pin ICSP header found on many AVR boards requires the following hookup:

| Pin Number | Label        | ISP Signal | ICSP Pin | FT2232 Pin  |
|------------|--------------|------------|----------|-------------|
| 1          | VTGT         | VTGT       |     2    | ---
| 2          | GND          | GND        |     6    | ---
| 3          | TCK          | SCK        |     3    | BD0
| 4          | TDI/MOSI/SDA | MOSI       |     4    | BD1
| 5          | TDO/MISO/--  | MISO       |     1    | BD2
| 6          | TMS/SS/--    | ---        |    ---   | ---
| 7          | TRST/--/--   | ---        |    ---   | ---
| 8          | SRST/--/--   | RST        |     5    | BD5

# Stripes and Tails
In order to support different interfaces, custom wiring harnesses or add-on boards might be useful. The preferred names are "Tigard Stripes" for add-on boards and "Tigard Tails" for wiring harnesses.

* The `UART Tail` ships with Tigard and includes VTGT, GND, TX and RX
* The `JTAG Tail` ships with Tigard and includes labeled multipurpose JTAG, SPI, and I2C wires
* The `SOIC8 Tail` refers to inexpensive SOIC-8 clips with ribbon cables and 2x4 connectors compatible with the SPI/I2C header

# Dens
Tigard is supplied as a bare board, but there are several options for 3d printing mounts and cases - 'dens' - for your Tigard.:

* The first [TigarDen](https://github.com/hamid-elaosta/TigarDen) by hamid is a multi-piece top and bottom case for holding tigard
* A [bottom-only case](https://www.thingiverse.com/thing:5166570) by phaktor has neat tiger stripes on the bottom, or can be printed in a single color
* A [hinged case](https://www.thingiverse.com/thing:5169952) by professor_plum holds a tigard, bitmagic, and several probes
* A [simplified 2-piece case](https://cad.onshape.com/documents/832906f0b96360664fedfe62/w/f1b917fd93e5b559c0927cf4/e/eb5666e6e328afd821567f83) by jwise for milling or printing
* [tigard-tray](https://github.com/tigard-tools/tigard-tray) is a 3d printed tray that holds tigard, bitmagic, and 9 probe clips while protecting the pins, so that it can slide in and out of a mesh pocket in a zippered case.

# Serial Numbers

Tigard follows a convention for allocating serial numbers. If you decide to make and sell your own Tigards, please change the first two characters "TG" to something else.

Format: `TGMmxxxx`

* `TG` - constant indicates "tigard"
* `M` - major version decimal number
* `m` - minor version decimal number
* `xxxx` - 0 padded lowercase hex serial starting at 0

## Allocated

* First batch of 28 (Jul 5, 2020): TG010000 - TG01001b
* Second batch of 42 (Aug 5, 2020): TG01001c - TG010046
* Third batch of 496 (October 2020): TG100000 - TG1001f0
* Fourth batch of 693 (January 2021): TG110000 - TG1102b5
* Fourth batch of 697 (September 2021): TG110400 - TG1106b9<br>Note: This batch's through hole components were not soldered with a selective solder wave machine. Some needed rework because the "hack" jumper was accidentally bridged with solder.
* Fifth batch of 1400 (Nov 2022): circa TG110700 - TG110c7f
* Sixth batch of 1400 (Jan 2024): circa TG110c80 - TG 1111ff <br>Note: This batch had the jtag header assembled in the wrong position. It's supposed to be an 8 pin header in a 9 hole footprint. Pin 9 of the footprint should be vacant - it's a special purpose pin only there for flashing ice40 fpgas like icebreaker. However, pin 1 was mistakenly left vacant instead. I added a header in pin 1 and trimmed pin 9 them so that the 8-pin jtag wiring harness can't be put on wrong and so the 9-pin uart header is not accidentally put there.
* Seventh batch of 2100 (Jan 2025): circa TG111200-TG111834

# Getting Tigard
## Where to Buy
Tigard can be purchased directly from [Crowd Supply](https://www.crowdsupply.com/securinghw/tigard), [Mouser](https://www.mouser.com/ProductDetail/Securing-Hardware/TIGARD-V1?qs=aP1CjGhiNiFnjSEE%2FnXyEw%3D%3D), or [1BitSquared](https://1bitsquared.com/collections/embedded-hardware/products/tigard)

[HackerWarehouse](https://hackerwarehouse.com/product/tigard/) also has a [laser-cut acrylic case](https://hackerwarehouse.com/product/tigard-case/) designed by SynapticRewrite

Tigard can be aquired in Europe via [1BitSquared's German store](https://1bitsquared.de/products/tigard)

10+ boards can be ordered at a discount direct from [SecuringHardware.com](https://learn.securinghardware.com/product/tigard/). Contact tigards - at - securinghw.com for wholesale orders.

## DIY Tigards
Tigard is published with the [CC-BY-SA 4.0 License](https://github.com/tigard-tools/tigard/blob/master/LICENSE.txt). This means you have permission and are encouraged to make your own devices, or make modifications to the Tigard design, as long as you attribute the project and share your changes.

If you choose to manufacture your own tigard boards, we recommend/request the following:
* Physically differentiate your boards somehow - choosing a different color is likely easiest - so they are not mistaken for boards from SecuringHardware.com
* You are welcome to call them "Tigard" and program that into the usb product descriptor in the EEPROM
* Program accurate manufacturer information in usb manufacturer descriptor the EEPROM
* Use serial numbers that do not conflict with the ones [noted above](#serial-numbers) - do not use the 'TG' prefix

SecuringHardware.com tests, verifies, and supports every Tigard they manufacture and sell. Please let us know about any [design issues](https://github.com/tigard-tools/tigard/issues) while making your own devices, but if you're having hardware issues with a third-party Tigard that you bought, you will need to resolve that with the seller or manufacturer.
