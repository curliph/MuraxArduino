# MuraxArduino

[MuraxArduino](https://github.com/lawrie/VexRiscv/blob/master/src/main/scala/vexriscv/demo/MuraxArduino.scala) is an implementation of an Arduino board for open source FPGAs. It uses a [RISC-V](https://en.wikipedia.org/wiki/RISC-V) CPU. It currently only supports the [mystorm BlackIce II ice40 board](https://github.com/mystorm-org/BlackIce-II/wiki/Getting-Started). 

It allows the FPGA to be programmed using the [Arduino IDE](https://www.arduino.cc/en/Main/Software), the [Arduino API](https://www.arduino.cc/reference/en/) and standard [Arduino libraries](https://www.arduino.cc/en/Reference/Libraries), but it makes use of the FPGA to provide accelerated hardware peripherals.

It is a version of [f32c/arduino](https://github.com/f32c/arduino) that works with the [SpinalHDL](https://github.com/SpinalHDL/SpinalHDL)  [Vexriscv](https://github.com/SpinalHDL/VexRiscv) [Murax SoC](https://github.com/SpinalHDL/VexRiscv/blob/master/src/main/scala/vexriscv/demo/Murax.scala).

## BlackIce II implementation

There is Arduino Board Manager support.

In File->Preferences->Additional Boards Manager URLs enter:

    https://raw.githubusercontent.com/lawrie/MuraxArduino/master/package_murax_core_index.json

Select pull down menu Tools->Board->Board Manager and install Murax Arduino.

Then select, Board: Blackice Murax FPGA board, CPU Speed: 50 Mhz, RAM Size: 512KB SRAM external, port /dev/tyyUSB0

The SpinalHDL scripts including the Makefile, pcf file and top level Verilog files are at https://github.com/lawrie/VexRiscv/tree/master/scripts/Murax/BlackIce, 
but there is a copy of the BlackIce II [binary](https://github.com/lawrie/MuraxArduino/blob/master/fpga/Blackice/bin/toplevel.bin) and the [pcf file](https://github.com/lawrie/MuraxArduino/blob/master/fpga/Blackice/toplevel.pcf), so you can try MuraxArduino out without installing SpinalHDL, by doing

```
git clone https://github.com/lawrie/MuraxArduino
cd MuraxArduino/fpga/BlackIce
stty -F /dev/ttyACM0
cat bin/toplevel.bin /dev/ttyACM0
```

You need both USB connections to the Blackice II board, as /dev/ttyUSB0 is used for uploading programs from the Arduino IDE and for the Arduino console.

### CPU

The CPU is the Vexriscv Risc-V 32-bit SpinalHDL implementation running at 50Mhz. See https://github.com/lawrie/VexRiscv/blob/master/src/main/scala/vexriscv/demo/MuraxArduino.scala for the configuration used for the CPU.

The CPU speed is set by the pll in toplevel_pll.v.

### Memory

If you select the 8Kb BRAM internal option, you get an 8kb device with a bootloader of just over 2kb, leaving just under 6kb for the Arduino sketch. The BRAM is mapped onto address 0x80000000. The BRAM could be increased to 12kb, but urrently the top 4kb is just used for the stack and function pointers for interrupts. 

The SRAM implementation is https://github.com/lawrie/VexRiscv/blob/master/src/main/scala/vexriscv/demo/MuraxSram.scala.

If you select the 512KB SRAM external option (the default) , you get the Blackice II external SRAM. The bootloader still runs in BRAM and the stack is still in BRAM, but the code and static data is in external SRAM which is mapped to address 0x90000000. The heap size is currently fixed as 64kb. 

It is recommended that SRAM is used as the BRAM implementation is not very robust or well tested.

### Peripherals

#### GPIO

There is currently a single 32-bit GPIO peripheral with Arduino digital pins mapped as follows:

```
Pins 0-3  : Output : Blackice LEDs
Pins 4-7  : Output : Pin 4 is used as the trigger (Blackice pin 33) for a ping sensor. Pin 5 is multiplexed with the shiftIn clk 
                     pin (see below) and pins 6 and 7 are multiplexed with the shiftOut clk and data pins.
pins 8- 9 : Input  : 8 and 9 are buttons 1 and 2.
pins 10-31: GPIO   : These correspond to the 4 switches (which double as SD card SPI pins) on pins 10 -13 , 
                     Blackice Pmod 2 pins 87 and 89 on Arduino pins 14 and 15, and the whole of  Pmods 3, 4, 5 and 6 on
                     Arduino Pins 16 - 31. Arduino pins 16 - 19  and 24 - 27 are multiplexed with the second 7-segment 
                     display (see below). Pins 16 and 17 can also be multiplexed with a Quadrature peripheral (see below).
```

The GPIO pins are accessed using the Arduino pinMode, digitalRead and digitalWrite methods. The INPUT_PULLUP mode is not implemented and is treated as INPUT.

The GPIO peripheral is implemented by the spinal.lib TristateBuffer. The SB_IO for the inout pins (10 - 31) is in toplevel.v.

#### UART

There is currently a single UART peripheral that maps to the Blackice USB 2 connector on pins 85 and 88. It is accesssed using the Arduino Serial class. There is no flow control.

The UART peripheral is implemented by the spinal.lib UartCtrl.

#### MachineTimer

A 32-bit microsecond machine timer is used for the implementation of the millis, micros, delay and delayMicroseconds methods.

MachineTimer is implemented by https://github.com/lawrie/VexRiscv/blob/master/src/main/scala/vexriscv/demo/MachineTimer.scala.

#### Mux

There is a Mux peripheral which has 32 pins controlled by a 32-bit register. The mux pins are implemebted in toplevel.v.
It allows FPGA pins to be multiplexed between different peripherals, such as  between GPIO and another peripheral.

4 muxes are currently used as follows:

```
Mux 0 : shiftIn clk if set, else GPIO pin 5.
Mux 1 : shiftOut clk and data if set, else GPIO pins 6 and 7.
Mux 2 : 7-segment display 1 if set, else GPIO pins 10-13 and 18-21
Mux 3 : Quadrature on pins Pmod 5 if set, else see Mux 2.
```

The Mux is implemented by https://github.com/lawrie/VexRiscv/blob/master/src/main/scala/vexriscv/demo/Mux.scala.

#### I2C 

There is a single I2C peripheral on Blackice pins 95 (SDA) and 93 (SCL). 

It is accessed using the Arduino Wire class.

Both master and slave are supported, but only master tested.

The I2c peripheral uses the spinal.lib I2cCtrl implementation.

#### SPI

There is a single SPI master peripheral on Pmod 10. SPI slave is not currently supported.

The SPI peripheral uses the spinal.lib SPICtrl implementation.

#### PulseIn

There is a single PulseIn peripheral on BlackIce pin 34. It can be used in combination with the trigger pin (Arduino pin 4)
to drive a ping sensor.

It is accessed using the Arduino pulseIn and pulseInLong methods.

The PulseIn peripheral is implemented by https://github.com/lawrie/VexRiscv/blob/master/src/main/scala/vexriscv/demo/PulseIn.scala.

#### PWM

There are currently 3 PWM pins available on BlackIce pin 94 and on thes DBG1 and DONE LED pins.

The  PWM duty cycle is set using the Arduino analogWrite method. The pin number is the PWM channel.

Any number (up to 64) of PWM pins can be configured, but currently only 3 are used.

The PWM is peripheral is implemented by https://github.com/lawrie/VexRiscv/blob/master/src/main/scala/vexriscv/demo/PWM.scala.

#### Tone

There is a single Tone peripheral available on Blackice pin 26.

Corresponds to the Arduino tone() and notTone() methods. The pin number on these methods is ignored.

The Tone is peripheral is implemented by https://github.com/lawrie/VexRiscv/blob/master/src/main/scala/vexriscv/demo/Tone.scala.

#### 7-segment LED display

There are two 7-segment periphewral designed to be used with Digilent Pmods PMODSSD.

The first (channel 0) is accessed by the SevenSegment class and is on Blackice Pmods 7 and 9.

The second (channel 1) is multiplexed with GPIO and maps to Pmods 3 and 5.

These are supported by the SevenSegment library. Channel 1 is accessed by the SevenSegment1 class.

The 7-segment peripheral is implemented by https://github.com/lawrie/VexRiscv/blob/master/src/main/scala/vexriscv/demo/SevenSegment.scala.

#### shiftIn

There is a single ShiftIn peripheral on BlackIce Pmod 11, pins 31 and 32.

It is accessed using the Arduino shiftIn function.

As the ShiftIn clk pin is muxed with GPIO pin 5, mux 0 is set by the shiftIn method and can only be unset by direct use of the Mux peripheral.

The ShiftIn peripheral  is implemented by https://github.com/lawrie/VexRiscv/blob/master/src/main/scala/vexriscv/demo/ShiftIn.scala.

#### shiftOut

There is a single ShiftOut peripheral on BlackIce Pmod 12, pins 21 and 22.

It is accessed using the Arduino shiftOut function.

As the ShiftOut clk and data pins are muxed with GPIO pins 6 and 7, mux 1 is set by the shiftOut method and can only be unset by direct use of the Mux peripheral.

The ShiftOut peripheral is implememted by https://github.com/lawrie/VexRiscv/blob/master/src/main/scala/vexriscv/demo/ShiftOut.scala.

#### Interrupts

Timer interrupts are implemented usinfg the MsTimer2 library and the spinal.lib Timer implementation.

The Arduino atachInterrupt and detachInterrupt functions are supported on up to 32 pins. Currently only the two Blackice buttons are support on interrupts 0 and 1.

The PinInterrupt peripheral is implemented by https://github.com/lawrie/VexRiscv/blob/master/src/main/scala/vexriscv/demo/PinInterrupt.scala.

Interrupts currently go to an interrupt function in the bootloader, which determines the type of interrupt and calls the appropriate interrupt function.

#### Quadrature

There is a single quadrature peripheral on Pmod 5 that supports encoders and encoder motors.

The Quadrature peripheral is multiplexed with GPIO pins 16 and 17 and with the channel 1 7-segment display.

The encoder position is read using the read method of the Quadrature library. Using this method sets mux 3 and it can only be unset by direct use of the Mux peripheral.

The Quadrature peripheral is implemented by https://github.com/lawrie/VexRiscv/blob/master/src/main/scala/vexriscv/demo/Quadrature.scala.

#### Servo

There is currently just one instance of the Servo peripheral on Blackice Pmod 12, pin 25.

Up to 32 servos can be configured. 

The Servo peripheral is is implemented by https://github.com/lawrie/VexRiscv/blob/master/src/main/scala/vexriscv/demo/Servo.scala.

#### Jtag

A Jtag debugger for the Risc-V processor is supported on Blackice Pmod 8. Most USB Jtag devices are supported. It has been tested wiith an Anlogic Lychee Tang device.

#### QSPI Analog

The 6 10-bit analog channels connected to the Blackice STM32 co-processor are supported via a QspiAnalog peripheral.

The analog pins are on the Blackice Arduino header. 

An [Arduino version of the Blackice iceboot firmware](https://github.com/lawrie/MuraxArduino/tree/master/firmware/QSPIanalog) is required, which sends the analog values to the ice40, continuously.

The QspiAnalog peripheral is implemented by https://github.com/lawrie/VexRiscv/blob/master/src/main/scala/vexriscv/demo/QspiAnalog.scala.

### Libraries

SPI and Wire Arduino libraries are implemented, which use the spinal.lib SPI and I2C peripherals.

The SD library is supported using the Blackice SD card, whicg shares pins with the switches.

There is an implementation of the Servo library, which uses a Servo peripheral, currently just on one pin.

There is an implementation of the MsTimer2 third-party timer library, which can be used for millisecond resolution timer interrupts.

There is a simple SevenSegment library to display a hex character on a Digilent PMOD SSD.

There is a simple Quadrature library.

There is a version of the Firmata library, but it is not yet tested.

### Third-party libraries

The Adafruit BME280 and SSD1306 libraries have been tested in i2c, SPI hardware and SPI software modes.

The LiquidCrystal library has been tested.

### Not implemented

The analogReadResolution, analogWriteResolution and analogReference methods are no ops.

The String toFloat method is not supported.

USB classes are not supported.

Lots of other features have not been tested.

### Bugs

The bootloader currently echoes the SREC data to the uart. 

The Arduino console is not synchronised correctly with rebooting the CPU, so if the console is open when an Arduino program is uploaaded, the program is started and you see some console output, which is then cut short, and the CPU is rebooted and you see the full output.

