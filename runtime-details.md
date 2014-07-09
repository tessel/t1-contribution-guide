# Runtime Details

You can find the runtime repo [here](github.com/runtime) and the source building instructions [here](./build-from-source.md#runtime). Note that you don't even need a Tessel to contribute to runtime – especially JS and Node.js compatibility. 

The runtime is a lua virtual machine and related C library that is responsible for:

- [Running Lua code, usually after being transpiled from JavaScript](#lua-vm)
- [Mainting compatibility with JavaScript](#javascript-compatibility)
- [Maintaining compatibility with Node.js](#node.js-compatibility)
- [Maintaining an Event Queue](#event-queue)
- [Supporting a file system](#vfs)
- [Testing the JS and Node compatibility](#tests)
- [Providing SSL bindings](#ssl-support)
- [Calling into firmware functions for hardware dependencies](./firmware-details.md#integration-with-runtime)

## Lua Virtual Machine

Tessel is running [a fork of the Lua 5.1 virtual machine](https://github.com/tessel/colony-lua) with a handful of changes to make it easier to create JS-like objects in Lua.

After code is transpiled from JavaScript to Lua, the Lua bytecode is run in the Lua VM.

## JavaScript Compatibility

The native JS functions (String, Number, Array, Boolean, Date, JSON, etc) are implemented in Lua at [`src/colony/lua/colony-js.lua`](https://github.com/tessel/runtime/blob/master/src/colony/lua/colony-js.lua). If you find a Native JS function is undefined, it will need to be added to this file.

## Node.js Compatibility

The implemented Node modules live underneath [`src/colony/modules`](https://github.com/tessel/runtime/tree/master/src/colony/modules).

Additionally some modules (Require, Buffer, EventEmitter) are written in Lua. Those modules are located at [`src/colony/lua/colony-node.lua`](https://github.com/tessel/runtime/blob/master/src/colony/lua/colony-node.lua). 

One question that comes up is, "How do I know if a node module should be written in Lua or JavaScript?" Generally, the answer depends on if the Node module or method needs to access the underlying hardware. If it does (like Buffer), it needs to be written in Lua, if not, it can be written in JS (like Streams).

## Event Queue

The Event Queue is a datastructure of events waiting to be triggered either from the runtime itself or (more likely) from the firmware. This is useful in the firmware where we have an interrupt request (IRQ) but we want to safely malloc data outside of the IRQ. We would trigger an event in the IRQ and handle it when the event queue fires the event.

The event queue implementation can be found in [`src/tm_event.c`](https://github.com/tessel/runtime/blob/master/src/tm_event.c). 

## VFS

The runtime implements a FAT filesystem which is used to read and write files to RAM or Flash. The implementation can be found in [the vfs directory](https://github.com/tessel/runtime/tree/master/src/vfs).

## Tests

The [`test/`](https://github.com/tessel/runtime/tree/master/test) directory includes several different kinds of tests. The two most relevant are [`/modules`](https://github.com/tessel/runtime/tree/master/test/modules) and ['/issues'](https://github.com/tessel/runtime/tree/master/test/issues). We keep tests for specific Node modules in the `modules` repo and tests for issues reported on Github in the `issues` repo. If you're working on a bug, please follow these conventions.

## SSL Support

The runtime provides SSL support through two primary files. [`src/tm_ssl.c`](https://github.com/tessel/runtime/blob/master/src/tm_ssl.c) provides the implementation around sessions and [`src/tm_random.c`](https://github.com/tessel/runtime/blob/master/src/tm_random.c) which provides the interface for key generation.

