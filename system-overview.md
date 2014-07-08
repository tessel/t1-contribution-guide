#Tessel System Overview

This is an overview of how all the components of Tessel work. It is intended to be a reference for those wishing to contribute to the internals of Tessel.

Tessel comprises of the following:
 
* **Firmware** - C code that is the interface to all the hardware components (Wifi, RAM, Flash, SPI/UART/I2C busses). It also runs the event queue and handles interrupts.
* **runtime** - the Lua VM running Lua code. It also includes the compatability layer for core JS functions (String, Number, etc), as well as the core Node functions (fs, buffers, etc)
* **Colony** - JS -> Lua compiler
* **CLI** - the command line interface for communicating to a Tessel over a USB bus. 

##Running Code
When code is pushed to the Tessel (via a `tessel run` or a `tessel push`), the following happens:

1. The CLI checks for a `package.json` or a `node_modules` folder in the current directory. If it finds either, it assumes it will bundle the entire directory.
2. The JS files are cross compiled to Lua using Colony.
3. This entire directory is sent over USB to Tessel through a [USB control transfer](http://www.beyondlogic.org/usbnutshell/usb4.shtml#Control).
5. Tessel handles the incoming event transfer and on the next iteration of the event loop rewrites the Flash/RAM with the contents of the transfer.
6. On the next iteration of the event loop, the new Flash/RAM code is run.

##Node/JS compatability
The Node & core JS function compatability layer is handled by the [runtime](https://github.com/tessel/runtime).

The Node modules live underneath [src/colony/modules](https://github.com/tessel/runtime/tree/master/src/colony/modules).

Additionally some modules (require, buffer, EventEmitter) are written in Lua. Those modules are located at [src/colony/lua/colony-node.lua](https://github.com/tessel/runtime/blob/master/src/colony/lua/colony-node.lua). 

Core JS functions (String, Number, Array, Boolean, Date, JSON, etc) are located at [src/colony/lua/colony-js.lua](https://github.com/tessel/runtime/blob/master/src/colony/lua/colony-js.lua).

The runtime can both be run from a computer and on Tessel. For more details and a guide on how to make changes in runtime read the [runtime contribution walkthrough]().
