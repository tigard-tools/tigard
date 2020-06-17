<img alt="Tigard PCB Render" width="320" align="right" src="https://raw.githubusercontent.com/tigard-tools/tigard/master/tigard-render.png">

# Tigard

An FTDI FT2232H-based multi-protocol tool for hardware hacking.

# Objectives

There are plenty of -232H series breakout boards, but they are generally designed to be an easy way to adapt it to a specific use, and not designed for regularly plugging in to all different target systems.

The two exceptions are the Exodus Intellegence Hardware Interface Board which is not open hardware or commercially available, and [TIMEP](https://github.com/Matir/timep) which is the origin and heritage of this project.

## Software

In general, Tigard was designed to work as-is with several tools and lbraries that already support the x232H family of chips. This includes:

* USB-Serial drivers for UART access
* OpenOCD and URJTAG for JTAG
* Flashrom, libmpsse, pyftdi and other tools for SPI interfaces
* libmpsse and pyftdi for I2C interfaces

## Hardware Features

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

## I2C (on SPI header)

#### Hookup:

The SPI/I2C header is laid out to be the same orientation as the pins on most 8-pin I2C chips, making it easy to attach clips or sockets.

* Select I2C on the mode selection switch
* Connect your clip or socket to the header. Pay attention to pin 1, which is usually marked on sockets or has a red/highlighted wire on ribbon cables
* Connect your clip to the target or insert your chip into the socket
* If you're doing this in-circuit, you *must* take extra precautions to make sure no other device is communicating with the SPI device.
* If you're using a clip in-circuit and the target is powered on, choose the VTGT option on the voltage slider
* If you're using a zif socket, or clip with a powered-off target, choose the voltage on the voltage slider. 

#### Quirks:

The FT2232H has a very limited I2C implementation. I2C depends on shared I/O lines using common emitter instead of push-pull-tristate I/O, but the FT2232H doesn't support common emitter. Therefore:

* Only Main operation is supported, not Subordinate
* Clock stretching is not supported
* The I2C switch ties the DI and DO lines together so that it can do bidirectional communication
* The pullup resistors are not included since they are usually located on the target, and the weak pullups on the level shifters are sufficient

#### Hacks:

To accomodate both I2C and SWD, the DI and DO lines are combined through resistor R16. This is required for SWD and acceptable for I2C. For the best I2C performace **with the tradeoff of breaking SWD functionality**, bridge the HACK solder jumper on the bottom. This will bypass resistor R16, shorting DI and DO when the MODE switch is set to SWD/I2C.

Many I2C targets already have pullup resistors. In addition, all of Tigard's I/O pins have a weak 100K ohm pullup. In testing, this has been sufficient for both in-circuit and external use of most I2C devices. If you need stronger pullups on I2C, you can temporarily add them by pulling up MOSI and SCK on the SPI header, or TCK and TDI on the JTAG header.

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

## JTAG (on JTAG or CORTEX header)

#### Hookup:

The JTAG header is laid out with pins in the same order as the FTDI I/O pins are labeled, in order to be consistent with many other x232H breakout boards.

In additon, the CORTEX header is also wired as a standard ARM 10-pin JTAG header.

Be sure to select JTAG on the mode selection switch. This makes sure that TDI and TDO are separated, and ensures that TMS is wired properly to the CORTEX header. Otherwise, the standard hookup sequence applies.

#### Software:

OpenOCD is a powerful tool for On-Chip Debugging of ARM, MIPS, and some other architectures.

The appropriate configuration file (make this a link to the file) should look like:

```
interface ftdi
ftdi_vid_pid 0x0403 0x6010
ftdi_channel 1
adapter_khz 2000
ftdi_layout_init 0x0078 0x017b
ftdi_layout_signal nTRST -ndata 0x0010 -noe 0x0040
ftdi_layout_signal nSRST -ndata 0x0020 -noe 0x0040
transport select jtag
```

To use it with openocd:

```bash
openocd -f tigard-jtag.cfg
```

## SWD (on CORTEX header)

#### Hookup:

The SWD header a standard 10-pin header found on many SWD target boards. A short 'SWD' cable with the same header on both ends is the ideal way to hook up to most targets.

Be sure to select SWD on the mode selection switch. This connects the DI and DO pins with resistor R16 to make the bidirectional SWDIO pin, and connect it to pin 2 of the CORTEX header. Otherwise, the standard hookup sequence applies.

#### Software:

OpenOCD is a powerful tool for On-Chip Debugging of ARM, MIPS, and some other architectures.

The appropriate configuration file (make this a link to the file) should look like:

```
interface ftdi
ftdi_vid_pid 0x0403 0x6010
ftdi_channel 1
adapter_khz 2000
ftdi_layout_init 0x0078 0x017b
ftdi_layout_signal nTRST -ndata 0x0010 -noe 0x0040
ftdi_layout_signal nSRST -ndata 0x0020 -noe 0x0040
transport select swd
```

To use it with openocd:

```bash
openocd -f tigard-swd.cfg
```

## Debugging

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

#### ALL LEDS ON:

When all LEDS are on, then Tigard is probably working as intended. If you are still having trouble, there are a few possibilities. They are listed in order of likelihood, though it makes sense to test the easier cases first:

* It's a *wiring* issure. Make sure your wires are connected well. Confirm with a multimeter.
* It's a *target* issue. Try a different target to make sure your software is working properly
* It's a *software* issue. Try a different software tool or mode to make sure your board works properly
* It's a *protocol* issue. Observe your signals with a logic analyzer or oscilloscope to make sure they're correct
* It's a *hardware* issue. Try a different Tigard board to see if it persists
* It's a *tigard* issue. Try a different x232H board to see if it persists

## Pinouts

There are way too many choices of 'standard' pinouts for all of these interfaces. Pinouts were chosen for ease of use, specificially:

* If there's a well established and used standard, use it (SWD)
* If there's a predominant primary usage that would be easier, use it (SPI/I2C)
* If there are other common boards with pinouts, use them (UART and JTAG)

### UART

This pinout prioritizes putting the FT2232H pins in sequential order - similar to many x232H breakout boards

The coloring of the wiring harness is intended to match most common usb-serial cables

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

This pinout prioritizes putting the FT2232H pins in sequential order - similar to many x232H breakout boards.

The coloring of the wiring harness is what SecuringHardware.com used for their Adafruit FT232H wiring harness for several years. The colors were chosen because frequently black-brown-red-orange are used with logic analyzers in class, so unique colors were chosen for this wiring harness.

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

This header is designed specifically to match the pinout of most 8-pin SPI and I2C chips, so that a jumper to a chip clip or a socket would align properly.

In order to accomodate both SPI and I2C, the mode switch:

1. Combines DI and DO to create SDA for I2C mode
2. Disconnects pin 2 in I2C mode for devices that use it as an address signal

| Chip Pin Number | Header Pin Number | SPI signal | FT2232H Pin |
| --------------- | ----------------- | ---------- | ----------- |
| 1               | 1                 | CS         | BD3         |
| 2               | 3                 | MISO       | BD2         |
| 3               | 5                 | WP         | pullup      |
| 4               | 7                 | GND        | --          |
| 5               | 8                 | MOSI       | BD1         |
| 6               | 6                 | SCK        | BD0         |
| 7               | 4                 | EN         | pullup      |
| 8               | 2                 | VTGT       | --          |

### LA

Often it is necessary to debug a protocol with a logic analyzer. This header is designed to connect directly to the Bitmagic logic analyzer, or you could use the Bitmagic logic analyzer wiring harness to connect to any other pinout.

The 8 most interesting signals are connected - 6 from the JTAG/SWD/SPI/I2C port, and 2 from the UART port.

| Pin Number | Bitmagic Signal | FT2232 Pin | SPI signal | JTAG signal | SWD Signal | I2C Signal | UART Signal |
| ---------- | --------------- | ---------- | ---------- | ----------- | ---------- | ---------- | ----------- |
| 1          | xPB0            | BD0        | TCK        | SCK         | SWCLK      | SCL        |
| 2          | xPB1            | BD1        | TDI        | MOSI        | SWDIO      | SDA        |
| 3          | xPB2            | BD2        | TDO        | MISO        | SWDIO      | SDA        |
| 4          | xPB3            | BD3        | TMS        | CS          |            |
| 5          | xPB4            | BD4        | TRST       |
| 6          | xPB5            | BD5        | SRST       |
| 7          | xPB6            | AD0        |            |             |            |            | TX          |
| 8          | xPB7            | AD1        |            |             |            |            | RX          |
| 9          | GND             |
| 10         | GND             |
| 11         | xTRIG           |
| 12         | xCLK            |
| 13         | xTRIG2          |
| 14         | xIFCLK          |
