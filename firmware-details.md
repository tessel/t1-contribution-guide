# Firmware Details

You can find the firmware repo [here](https://github.com/tessel/firmware) and the source building instructions [here](./build-from-source.md#firmware).

When Tessel boots, firmware gets loaded into flash by way of the [bootloader](#bootloader).

The firmware layer is responsible for:
- [Receiving commands from the Tessel API](#tessel-api)
- [Transceiving data from peripherals over the SPI, UART, and I2C communication buses](#communication-buses).
- [Reading and Writing Pin States](#reading-and-writing-pin-states)
- [Communicating with the host computer over USB](#usb-communication)
- [Managing and updating the Wi-Fi chip](#cc-3k-wifi-chip)
- [Managing hardware interrupts](#hardware-interrupts)
- [Communicating with the Lua VM](#integrating-with-runtime)
- [Putting the processor to sleep when it's not being used](#power-reduction-modes)

## Bootloader

When Tessel is first programmed at the factory, the OTP binary sets the OTP bits to (0x0Y) where Y is the hardware version of Tessel (Y was 4 for the final production run). The OTP binary then loads the bootloader into flash and sets the OTP bits such that the microcontroller knows to always bootup from flash.

The OTP script can be found in [`/otp/main.c`](https://github.com/tessel/firmware/tree/master/otp/main.c) and the bootloader can be found in [`/boot/main.c`](https://github.com/tessel/firmware/blob/master/boot/main.c).

## Tessel API

The Tessel API is the built-in JavaScript file that defines the high-level interfaces to the Tessel board. It includes functions for sending data over the [communication buses](#communication-buses), writing and reading [GPIO pins](http://en.wikipedia.org/wiki/General-purpose_input/output), waiting for hardware interrupts, and much more.

The API documentation can be found [on our docs](https://github.com/tessel/docs) and the implementation can be found at [`builtin/tessel.js`](https://github.com/tessel/firmware/blob/master/builtin/tessel.js). You'll notice that it binds to two processes exposed by the runtime at the top of the file:

```.js
var tm = process.binding('tm');
var hw = process.binding('hw');
```

The `hw` process exposes Lua bindings to the hardware interfaces and the `tm` process exposes Lua bindings to the Technical Machine runtime specific interfaces.

We then have a functionality that binds those Lua functions to the underlying C function which can access the hardware.

For example, when a function like `tessel.port['A'].digital[0].write(0)` is
called, the following happens:

1. The Tessel object's properties are read from [`builtin.js`](https://github.com/tessel/firmware/blob/master/builtin/tessel.js):

  ```.js
  this.ports =  {
      A: new Port('A', [new Pin(hw.PIN_A_G1), new Pin(hw.PIN_A_G2), new Pin(hw.PIN_A_G3)], [], [],
        hw.I2C_1,
        hw.UART_3
      ),
      ...
  ```
2. At the top of `builtin.js` is `var hw = process.binding('hw');`. This process binding is what allows us to [call into the exposed Lua hardware functions](https://github.com/tessel/firmware/blob/master/src/hw/l_hw.c#L709):
  ```.js
  LUALIB_API int luaopen_hw(lua_State* L)
  {
    luaL_reg regs[] = {
      ...
      // gpio / leds
      { "digital_output", l_hw_digital_output },
      { "digital_input", l_hw_digital_input },
      { "digital_write", l_hw_digital_write },
      { "digital_read", l_hw_digital_read },
      { "digital_get_mode", l_hw_digital_get_mode},
      ...
    }
  ```

3. In the case of this example (digital.write), we are calling the [`l_hw_digital_write`](https://github.com/tessel/firmware/blob/master/src/hw/l_hw.c#L384) function:

  ```.c
  static int l_hw_digital_write(lua_State* L)
  {
    uint32_t pin = (uint32_t)lua_tonumber(L, ARG1);
    uint32_t level = (uint32_t)lua_tonumber(L, ARG1 + 1);

    hw_digital_write(pin, level);

    return 0;
  }

  ```
Which in turn calls the [`hw_digital_write`](https://github.com/tessel/firmware/blob/master/src/hw/hw_digital.c#L85) function:
  ```.c
  void hw_digital_write (size_t ulPin, uint8_t ulVal)
  {
    if (ulVal != HW_LOW) {
      GPIO_SetValue(g_APinDescription[ulPin].portNum, 1 << g_APinDescription[ulPin].bitNum);
    } else {
      GPIO_ClearValue(g_APinDescription[ulPin].portNum, 1 << (g_APinDescription[ulPin].bitNum));
    }
  }
  ```
4. The `hw-digital_write` function then writes the appropriate value to the hardware registers to change the physical state of the pin.

For returning values from a Lua function, you must use Lua's C API to push and/or pop values onto the stack and return the number of elements that should be returned to JS (in an array). As an example, check out how [SPI returns an error code](https://github.com/tessel/firmware/blob/master/src/hw/l_hw.c#L146) (it doesn't need to return the receive buffer because that's allocated prior to the transfer and received bytes are simply placed in the buffer).

For a guide on how to make changes in firmware read the [C to JS guide](https://github.com/tessel/docs/blob/master/tutorials/c-to-js.md).

## Reading and Writing Pin States

Tessel has three GPIO pins on each module port and six GPIO pins on the GPIO bank. Additionally, The GPIO bank has six pins for reading analog values.

### Writing to the GPIO pins

The interface for setting a pin as output and pulling 'high', 'low', setting a pull up or pulldown, or leaving in a 'tri-state' is defined in [`src/hw/hw_digital.c`](https://github.com/tessel/firmware/blob/master/src/hw/hw_digital.c).

Digital pins G4-G6 on the GPIO bank can also be used as [PWM](http://en.wikipedia.org/wiki/Pulse-width_modulation) pins which is handy for moving servos, lighting LEDs, etc. The implementation of PWM can be found in [`src/hw/hw_pwm.c`](https://github.com/tessel/firmware/blob/master/src/hw/hw_pwm.c) and takes advantage of the built-in hardware [State Configurable Timer (SCT)](http://www.lpcware.com/content/project/SCTimer-PWM).

The driver generates a PWM signal with two events: [one for the end of a PWM period](https://github.com/tessel/firmware/blob/master/src/hw/hw_pwm.c#L31), and one for [the end of a duty cycle](https://github.com/tessel/firmware/blob/master/src/hw/hw_pwm.c#L61). The end of period event sets the output high and the end of duty cycle event sets the output low.

**Hardware Limitations**: All of the PWM pins share the same frequency but can have different duty cycles.

### Reading from the GPIO pins

The code to set a pin as input and read a digital state is found in [`src/hw/hw_digital.c`](https://github.com/tessel/firmware/blob/master/src/hw/hw_digital.c).

The Analog Pins (A1-A5 on the GPIO bank) can read analog values and convert them to a digital value. The implementation can be found in [`src/hw/hw_analog.c`](https://github.com/tessel/firmware/blob/master/src/hw/hw_analog.c).

## Communication Buses

There are three hardware communication buses exposed on Tessel: [SPI](http://en.wikipedia.org/wiki/Serial_Peripheral_Interface_Bus), [I2C](http://en.wikipedia.org/wiki/I%C2%B2C), and [UART](http://en.wikipedia.org/wiki/Uart). Each have their own pros and cons as far as speed, number of wires, complexity, etc. which is why different integrated circuits use different protocols. These bus protocols can be seen as the analogue of HTTP in the embedded domain (with a different physical layer).

The drivers for these buses can be found in the [`/src/hw/`](https://github.com/tessel/firmware/tree/master/src/hw) folder in the firmware repo.

### SPI

There are two SPI drivers used for SPI: [`hw_spi.c`](https://github.com/tessel/firmware/blob/master/src/hw/hw_spi.c) and [`hw_spi_async.c`](https://github.com/tessel/firmware/blob/master/src/hw/hw_spi_async.c). We use the synchronous SPI driver for communication with the CC3k Wi-Fi chip and the asynchronous SPI driver for our exposed hardware API.

The synchronous SPI driver uses the interface defined by the [NXP provided SPI drivers](https://github.com/tessel/firmware/tree/master/lpc18xx) to start a polling transfer of bytes.

The asynchronous SPI driver is more complex in that it uses [DMA (Direct Memory Access)](http://en.wikipedia.org/wiki/Direct_memory_access). DMA allows us to reduce the CPU processing resources significantly for a SPI transfer by physically arranging "pipes" for data to be sent. Essentially, we tell the MCU [which interfaces to send data between](https://github.com/tessel/firmware/blob/master/src/hw/hw_spi_async.c#L193-L199), then [create a linked list for the transmit and receive buffers](https://github.com/tessel/firmware/blob/master/src/hw/hw_spi_async.c#L234) (because DMA can only deal with 4k bytes at a time), [and tell it to go](https://github.com/tessel/firmware/blob/master/src/hw/hw_spi_async.c#L254).

### I2C

[I2C](http://en.wikipedia.org/wiki/I%C2%B2C) transmission is relatively simple and can be found at [`src/hw/hw_i2c.c`](https://github.com/tessel/firmware/blob/master/src/hw/hw_i2c.c). We use the NXP provided I2C driver by creating a struct of transmission data, and [sending it off](https://github.com/tessel/firmware/blob/master/src/hw/hw_i2c.c#L84-L95).

**Hardware Limitations**: The I2C protocol demands that each device has a unique address which a master I2C device uses to send data to it. You cannot have more than one device with a specific address on an I2C bus. Tessel has two I2C buses routed to each side of the board (Ports A, B and C, D) which means you cannot plug in more than two of any module with a specific I2C address.

### UART

The [UART](http://en.wikipedia.org/wiki/Uart) driver can be found at [`/src/hw/hw_uart.c`](https://github.com/tessel/firmware/blob/master/src/hw/hw_uart.c) and is designed around a [ring buffer](http://en.wikipedia.org/wiki/Circular_buffer) architecture. UART is not a transaction-based protocol like SPI or I2C so transmissions can come or go at any time. A ring buffer lets us set a maximim amount of memory we're willing to allocate for UART data with the downside of possibly overwriting unread data if it comes in too quickly.

Data transmission and reception is achieved with the [NXP provided UART driver](https://github.com/tessel/firmware/blob/master/lpc18xx/Drivers/include/lpc18xx_uart.h). When data is received, it is placed at the next available position in the ring buffer and the number of bytes available to be read is incremented accordingly. We the trigger an event in the [event queue](#event-queue) so that the ring buffer will be read at the next available chance.

**Hardware Limitations**: There were not enough UART buses to route to every port so port C and the GPIO bank do not yet have a UART interface. We are planning on adding software UART support on those ports in the near future (or you can contribute!).

## USB Communication

Tessel communicates over USB micro cables to a host computer. Technical Machine was granted a USB PID/VID from [OpenMoco](http://wiki.openmoko.org/wiki/USB_Product_IDs) because we create open source hardware. The code that defines that the USB behavior can be found in [`/src/usb/`](https://github.com/tessel/firmware/tree/master/src/usb).

- [`device.c`](https://github.com/tessel/firmware/blob/master/src/usb/device.c) defines the implementation for USB vendor requests.
- [`log_interface.c`](https://github.com/tessel/firmware/blob/master/src/usb/log_interface.c) defines the implementation for human readable messages such as those created with `console.log`.
- [`msg_interface.c`](https://github.com/tessel/firmware/blob/master/src/usb/msg_interface.c) defines the implementation for machine-readable commands like those to manage Wi-Fi.
- [`usb.c`](https://github.com/tessel/firmware/blob/master/src/usb/usb.c) defines the implementation for arbitrary message passing.

## CC3k Wi-Fi Chip

We use TI's [CC3000](http://processors.wiki.ti.com/index.php/CC3000) as Tessel's Wi-Fi chip.

The CC3000 has TI's custom firmware loaded on to it. The version can be checked with a `tessel version --board`.

Tessel talks to the CC3000 through a dedicated [SPI bus](http://en.wikipedia.org/wiki/Serial_Peripheral_Interface_Bus). One SPI bus is used for all external ports (Ports A-D & GPIO), and the other SPI bus is use exclusively for Wi-Fi.

We use TI's [utility functions](https://github.com/tessel/firmware/tree/master/src/cc3000/utility) for communication and have ported over the [host drivers](https://github.com/tessel/firmware/blob/master/src/cc3000/host_spi.c) to work with Tessel.

The CC3000 functions are wrapped at [`src/tm/tm_net.c`](https://github.com/tessel/firmware/blob/master/src/tm/tm_net.c). These are the functions that our Lua runtime binds to for the `net` and `http` modules as well.

When Tessel first boots up, it does the following

1. `tessel_wifi_init` is called in [`src/main.c`](https://github.com/tessel/firmware/blob/master/src/main.c).
2. This enables the CC3000 by toggling a software enable pin and hooks a Tessel pin to listen for IRQs in [`src/tessel_wifi.c`](https://github.com/tessel/firmware/blob/master/src/tessel_wifi.c)
3. The Wi-Fi connection LED starts blinking
4. During bootup the CC3000 checks the last profile it used, and because we use [Fast Connect](http://processors.wiki.ti.com/index.php/CC3000_Host_Programming_Guide#Using_WLAN_policy_and_profiles) it will try to automatically reconnect.
5. After the connection, the CC3000 will trigger the `CC3000_UsynchCallback` callback at [`src/cc3000/host_spi.c`](https://github.com/tessel/firmware/blob/master/src/cc3000/host_spi.c).
6. `CC3000_UsynchCallback` will then trigger either `_cc3000_cb_wifi_connect` or `_cc3000_cb_wifi_disconnect` in [`src/tessel_wifi.c`](https://github.com/tessel/firmware/blob/master/src/tessel_wifi.c), which will in turn emit a command up to Tessel's CLI.

Trying to connect to Wi-Fi through a CLI `tessel wifi -n -p` does the following:

1. In CLI [`bin/tessel-wifi.js`](https://github.com/tessel/cli/blob/master/bin/tessel-wifi.js) the `wifi` command is parsed and calls the `configureWifi` command at [`src/commands.js`](https://github.com/tessel/cli/blob/master/src/commands.js).
2. `configureWifi` sends a USB command packet along with the Wi-Fi credentials.
3. The command is processed at `tessel_cmd_process` in [`src/main.c`](https://github.com/tessel/firmware/blob/master/src/main.c)
4. This then goes through the same steps as first bootup to connect.
5. CLI reads the emitted events from `_cc3000_cb_wifi_connect` or `_cc3000_cb_wifi_disconnect` and exposes that back to the user.


## Hardware Interrupts

GPIO pins can wait for their state to change in order to fire an interrupt. The pin change can be fired on a signal `rise`, `fall`, `high` (like `rise` but can fire immediately), or `low` (like `fall` but can fire immediately). The LPC1830 microcontroller allows up to 8 simultaneous active interrupts, three of which are used by internal components (like the Wi-Fi chip).

The code handling interrupts can be found in [`/src/hw/hw_interrupt.c`](https://github.com/tessel/firmware/blob/master/src/hw/hw_interrupt.c). When an interrupt fires, [an event is queued](https://github.com/tessel/firmware/blob/master/src/hw/hw_interrupt.c#L258) in the event queue. At the next available moment, the interrupt is fired into Lua.

A tutorial for using hardware interrupts can be found [in our tutorials](https://github.com/tessel/docs/blob/master/tutorials/gpio-interrupts.md).


## Integration with Runtime

Integration with the runtime allows the firmware to access several hardware agnostic resources such as the event queue and the Lua VM. The integration also allows the runtime to utilize the dedicated hardware for specific critical tasks such as timing, net access, and entropy generation. The separation of the runtime from the firmware will allow us to build out a portable version of the runtime that is platform agnostic (amongst ARM M3+ chips).

Integration points in the firmware can be found in [`/src/tm`](https://github.com/tessel/firmware/tree/master/src/tm).

## Power Reduction Modes

Tessel currently does very little in the way of reducing power when the event queue is empty. This is an area for much improvement.
