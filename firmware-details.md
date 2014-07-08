# Firmware Details

You can find the firmware repo [here](github.com/firmware) and the source building instructions [here](./build-from-source.md)

When Tessel boots, firmware gets loaded into flash by way of the [bootloader](#bootloader).

The firmware layer is responsible for:
- [Receiving commands from the Tessel API](#tessel-api)
- [Transcieving data from peripherals over the SPI, UART, and I2C communication Buses](#communication-buses). 
- [Changing Pin States](#changing-pin-states)
- [Communinicating with the host computer over USB](#usb-communication)
- [Managing and updating the WiFi chip](#cc3k-wifi-chip)
- [Queueing and firing events from the event queue](#event-queue)
- [Managing hardware interrupts](#hardware-interrupts)
- [Communicating with the Lua VM](#integrating-with-runtime)
- [Putting the processor to sleep when it's not being used](#power-reduction-modes)

## Bootloader

When Tessel is first programmed at the factory, two things happen. First...

## Tessel API

The Tessel API is the built-in JavaScript file that defines the high-level interfaces to the Tessel board. It includes functions for sending data over the [communication buses](#communication-buses), writing and reading [GPIO pins](http://en.wikipedia.org/wiki/General-purpose_input/output), waiting for hardware interrupts, and much more.

The API documentation can be found [on our docs](https://github.com/tessel/docs) and the implementation can be found at [`built-in/tessel.js`](https://github.com/tessel/firmware/blob/master/builtin/tessel.js). You'll notice that it binds to two processes exposed by the runtime at the top of the file:

```.js
var tm = process.binding('tm');
var hw = process.binding('hw');
```

The `hw` process exposes Lua bindings to the hardware interfaces and the `tm` process exposes Lua bindings to Technical Machine specific interfaces (like time keeping).

We then have a functionality that binds those Lua functions to the underlying C function which can access the hardware.

For example, when a function like `tessel.port['A'].digital[0].write(0)` is 
called, the following happens:

1. The Tessel object's properties are read from [builtin.js](https://github.com/tessel/firmware/blob/master/builtin/tessel.js).
2. At the top of `builtin.js` is `var hw = process.binding('hw');`. This process binding is what allows us to [call into the Lua functions](https://github.com/tessel/firmware/blob/master/src/hw/l_hw.c#L709).
3. In the case of this example (digital.write), we are calling the [`l_hw_digital_write` function](https://github.com/tessel/firmware/blob/master/src/hw/l_hw.c#L384) which in turn calls the [`hw_digital_write`](https://github.com/tessel/firmware/blob/master/src/hw/hw_digital.c#L85) function.
4. The `hw-digital_write` function then writes the appropriate value to the hardware registers to change the physical state of the pin.

For returning values from a Lua function, you must use Lua's C API to push and/or pop values onto the stack and return the number of elements that should be returned to JS (in an array). As an example, check out how [SPI returns an error code](https://github.com/tessel/firmware/blob/master/src/hw/l_hw.c#L146) (it doesn't need to return the receive buffer because that's allocated prior to the transfer and received bytes are simply placed in the buffer).

For a guide on how to make changes in firmware read the [C to JS guide](https://github.com/tessel/docs/blob/master/tutorials/c-to-js.md).

## Communication Buses 

There are three hardware communication buses exposed on Tessel: [SPI](http://en.wikipedia.org/wiki/Serial_Peripheral_Interface_Bus), [I2C](http://en.wikipedia.org/wiki/I%C2%B2C), and [UART](http://en.wikipedia.org/wiki/Uart). Each have their own pros and cons as far as speed, number of wires, complexity, etc. which is why different integrated circuits use different protocols. These bus protocols can be seen as the analogue of HTTP in the embedded domain (with a different physical layer).

The drivers for these buses can be found in the [`/src/hw/`](https://github.com/tessel/firmware/tree/master/src/hw) folder in the firmware repo. 

### SPI

There are two SPI drivers used for SPI: [`hw_spi.c`](https://github.com/tessel/firmware/blob/master/src/hw/hw_spi.c) and [hw_spi_async.](https://github.com/tessel/firmware/blob/master/src/hw/hw_spi_async.c). We use the synchronous SPI driver for communication with the CC3k WiFi chip and the asynchronous SPI driver for our exposed hardware API. 

The synchronous SPI driver uses the interface defined by the [NXP provided example drivers](https://github.com/tessel/firmware/tree/master/lpc18xx) to start a polling transfer of bytes.

The asynchronous SPI driver is more complex in that it uses [DMA (Direct Memory Access)[http://en.wikipedia.org/wiki/Direct_memory_access]. DMA allows us to reduce the CPU processing resources significantly for a SPI transfer by physically arranging "pipes" for data to be sent. Essentially, we tell the MCU [which interfaces to send data between](https://github.com/tessel/firmware/blob/master/src/hw/hw_spi_async.c#L193-L199), then [create a linked list for the transmit and receive buffers](https://github.com/tessel/firmware/blob/master/src/hw/hw_spi_async.c#L234) (because DMA can only deal with 4k bytes at a time), [and tell it to go](https://github.com/tessel/firmware/blob/master/src/hw/hw_spi_async.c#L254).

### I2C

[I2C](http://en.wikipedia.org/wiki/I%C2%B2C) transmission is relatively simple and can be found at [`src/hw/hw_i2c.c`](https://github.com/tessel/firmware/blob/master/src/hw/hw_i2c.c). We use the NXP provided I2C driver by creating a struct of transmission data, and [sending it off](https://github.com/tessel/firmware/blob/master/src/hw/hw_i2c.c#L84-L95).

### UART

The [UART](http://en.wikipedia.org/wiki/Uart) driver can be found at [`/src/hw/hw_uart.c`](https://github.com/tessel/firmware/blob/master/src/hw/hw_uart.c) and is designed around a [ring buffer](http://en.wikipedia.org/wiki/Circular_buffer) architecture. UART is not a transaction-based protocol like SPI or I2C so transmissions can come or go at any time. A ring buffer lets us set a maximim amount of memory we're willing to allocate for UART data with the downside of possibly overwriting unread data if it comes in too quickly.

Data transmission and reception is achieved with the [NXP provided UART driver](https://github.com/tessel/firmware/blob/master/lpc18xx/Drivers/include/lpc18xx_uart.h). When data is received, it is placed at the next available position in the ring buffer and the number of bytes available to be read is incremented accordingly. We the trigger an event in the [event queue](#event-queue) so that the ring buffer will be read at the next available chance.

## USB Communication

Tessel communicates over USB micro cables to a host computer. Technical Machine was granted a USB PID/VID from [OpenMoco](http://wiki.openmoko.org/wiki/USB_Product_IDs) because we create open source hardware. The code that defines that the USB behavior can be found in [`/src/usb/`]https://github.com/tessel/firmware/tree/master/src/usb). 

[`device.c`](https://github.com/tessel/firmware/blob/master/src/usb/device.c) defines the implementation for USB vendor requests.
['log_interface.c'](https://github.com/tessel/firmware/blob/master/src/usb/log_interface.c) defines the implementation for human readable messages such as those created with `console.log`.
[`msg_interface.c`](https://github.com/tessel/firmware/blob/master/src/usb/msg_interface.c) defines the implementation for machine-readable commands like those to manage WiFi.
[`usb.c`](https://github.com/tessel/firmware/blob/master/src/usb/usb.c) defines the implementation for arbitrary message passing.

## CC3k WiFi Chip


## Event Queue

## Hardware Interrupts

## Integration with Runtime

## Power Reduction Modes

