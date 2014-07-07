#Building From Source

Firmware and runtime have their own compilation tools but many of the same dependencies. We have instructions for OSX and Linux. Windows users should use Linux in a Virtual Machine. The CLI and colony-compiler are simply npm repositories.

##Runtime
The Lua VM and bindings to the firmware (includes Node.js library).

###Installation
To install the runtime compilation tools, check out [these instructions on the runtime README](https://github.com/tessel/runtime#tessel-runtime). The runtime can  be tested on your computer without a Tessel by running javascript files with `colony` instead of `tessel run` or `tessel push` (eg. `colony index.js`). 

##Firmware
The low-level operating system and bindings to the hardware

###Installation
To install the firmware compilation tools, check out [these instructions on the firmware README](https://github.com/tessel/firmware#compiling). 

If you'd like to compile with a local version of runtime, you should create a symlink in your deps folder to your cloned version of runtime.

For example (within cloned firmware, on OSX)
```
firmware >> cd /deps
firmware/deps >> rm -rf runtime
firmware/deps >> ln -s <PATH TO RUNTIME> runtime
firmware/deps >> ls

runtime usb
firmware/deps >> cd ../
firmware >> make arm-deploy
```

Replace <PATH TO RUNTIME> with the actual path to your cloned Runtime repo.

### Debugging
Optionally, you can set up a hardware debugger that will allow you to place breakpoints, pause execution, and read memory with GDB. We recommend using the open source [Bus Blaster by Dangerous Prototypes](http://www.seeedstudio.com/depot/Bus-Blaster-v3-p-1415.html) and we have [a tutorial in our docs](https://github.com/tessel/docs/blob/master/tutorials/debug-using-busblaster.md) to help you get started. 


##Command Line Interface

The interface used to send data between a host computer and the Tessel.

##Installation
To install your own version of CLI, [clone our CLI repo], then run `npm link --local` within the CLI repo. Note that if you `npm install tessel -g`, you will override this local link.


##Colony

The transpiler that converts JavaScript to Lua Byte Code

###Installation
To install our JS-> Lua compiler, simply clone [the colony-compiler repo](https://github.com/tessel/colony-compiler).
