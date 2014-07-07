#Tessel System Overview

This is an overview of how all the components of Tessel work. It is intended to be a reference for those wishing to contribute to the internals of Tessel.

Tessel comprises of the following:

* firmware - C code that is the interface to all the hardware components (Wifi, RAM, Flash, SPI/UART/I2C busses). It also runs the event queue and handles interrupts.
* runtime - the Lua VM running Lua code. It also includes the compatability layer for core JS functions (String, Number, etc), as well as the core Node functions (fs, buffers, etc)
* Colony - JS -> Lua compiler
* CLI - the command line interface for communicating to a Tessel over a USB bus. 

##Running Code
When code is pushed to the Tessel (via a `tessel run` or a `tessel push`), the following happens:

1. The CLI checks for a `package.json` or a `node_modules` folder in the current directory. If it finds either, it assumes it will bundle the entire directory.
2. The JS files are cross compiled to Lua using Colony.
3. This entire directory is sent over USB to Tessel through a [USB control transfer](http://www.beyondlogic.org/usbnutshell/usb4.shtml#Control).
5. Tessel handles the incoming event transfer and on the next iteration of the event loop rewrites the Flash/RAM with the contents of the transfer.
6. On the next iteration of the event loop, the new Flash/RAM code is run.

##Binding JS functions to hardware
[JS functions that are dependent on hardware](https://github.com/tessel/docs/blob/master/hardware-api.md) is exposed through the [firmware](https://github.com/tessel/firmware).

When a function like `tessel.port['A'].digital[0].write(0)` is 
called, the following happens:

1. The Tessel object's properties are read from [builtin.js](https://github.com/tessel/firmware/blob/master/builtin/tessel.js).
2. At the top of `builtin.js` is `var tm = process.binding('tm');`. This process binding is what allows us to [call into the Lua functions](https://github.com/tessel/firmware/blob/master/src/hw/l_hw.c#L709).
3. In the case of this example (digital.write), we are calling the `l_hw_digital_write` function which in turn calls the `hw_digital_write` function.

For more details and a guide on how to make changes in firmware read the [firmware contribution walkthrough]().


##Node/JS compatability
The Node & core JS function compatability layer is handled by the [runtime](https://github.com/tessel/runtime).

The Node modules live underneath [src/colony/modules](https://github.com/tessel/runtime/tree/master/src/colony/modules).

Additionally some modules (require, buffer, EventEmitter) are written in Lua. Those modules are located at [src/colony/lua/colony-node.lua](https://github.com/tessel/runtime/blob/master/src/colony/lua/colony-node.lua). 

Core JS functions (String, Number, Array, Boolean, Date, JSON, etc) are located at [src/colony/lua/colony-js.lua](https://github.com/tessel/runtime/blob/master/src/colony/lua/colony-js.lua).

The runtime can both be run from a computer and on Tessel. For more details and a guide on how to make changes in runtime read the [runtime contribution walkthrough]().

##Wifi
We use TI's [CC3000](http://processors.wiki.ti.com/index.php/CC3000) as Tessel's wifi chip. 

The CC300 has TI's custom firmware loaded on to it. The version can be checked with a `tessel version --board`.

Tessel talks to the CC3000 through a dedicated [SPI bus](http://en.wikipedia.org/wiki/Serial_Peripheral_Interface_Bus). One SPI bus is used for all external ports (Ports A-D & GPIO), and the other SPI bus is use exclusively for Wifi. 

We use TI's [utility functions](https://github.com/tessel/firmware/tree/master/src/cc3000/utility) for communication and have ported over the [host drivers](https://github.com/tessel/firmware/blob/master/src/cc3000/host_spi.c) to work with Tessel.

The CC3000 functions are wrapped at [src/tm/tm_net.c](https://github.com/tessel/firmware/blob/master/src/tm/tm_net.c). These are the functions that our Lua runtime binds to for the `net` and `http` modules as well.

When Tessel first boots up, it does the following

1. `tessel_wifi_init` is called in [src/main.c](https://github.com/tessel/firmware/blob/master/src/main.c).
2. This enables the CC3000 by toggling a software enable pin and hooks a Tessel pin to listen for IRQs in [src/tessel_wifi.c](https://github.com/tessel/firmware/blob/master/src/tessel_wifi.c)
3. The wifi connection LED starts blinking
4. During bootup the CC3000 checks the last profile it used, and because we use [Fast Connect](http://processors.wiki.ti.com/index.php/CC3000_Host_Programming_Guide#Using_WLAN_policy_and_profiles) it will try to automatically reconnect.
5. After the connection, the CC3000 will trigger the `CC3000_UsynchCallback` callback at [src/cc3000/host_spi.c](https://github.com/tessel/firmware/blob/master/src/cc3000/host_spi.c). 
6. `CC3000_UsynchCallback` will then trigger either `_cc3000_cb_wifi_connect` or `_cc3000_cb_wifi_disconnect` in [src/tessel_wifi.c](https://github.com/tessel/firmware/blob/master/src/tessel_wifi.c), which will in turn emit a command up to Tessel's CLI.

Trying to connect to Wifi through a CLI `tessel wifi -n -p` does the following:

1. In CLI [bin/tessel-wifi.js](https://github.com/tessel/cli/blob/master/bin/tessel-wifi.js) the wifi command is parsed and calls the `configureWifi` command at [src/commands.js](https://github.com/tessel/cli/blob/master/src/commands.js).
2. `configureWifi` sends a USB command packet along with the wifi credentials.
3. The command is processed at `tessel_cmd_process` in [src/main.c](https://github.com/tessel/firmware/blob/master/src/main.c)
4. This then goes through the same steps as first bootup to connect.
5. CLI reads the emitted events from `_cc3000_cb_wifi_connect` or `_cc3000_cb_wifi_disconnect` and exposes that back to the user.
