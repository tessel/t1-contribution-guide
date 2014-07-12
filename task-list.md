# The Task List

This list contains improvements that would substantially increase the value and usability of the Tessel ecosystem. We'd love help getting these additions complete and are happy to help advise if you're interested in picking one up! See the [contribution process](./contribution-process.md) for more details on how to contribute.

Most urgently, we would also love assistance fixing [runtime](github.com/tessel/runtime/issues), [firmware](github.com/tessel/firmware/issues), and [colony](https://github.com/tessel/colony-compiler/issues) bugs. 

## Runtime

- [We have many Node.js methods that aren't supported yet](https://tessel.io/docs/compatibility#node). This is where we need most of the help. 

- [Add JavaScript Typed Array Support](https://github.com/tessel/runtime/issues/254). These sized arrays would be really helpful for lower-level hardware.

- [String slice doesn't work correctly with an end parameter of 0](https://github.com/tessel/runtime/issues/244).


## Firmware
- [CC3000 Sockets may not be recycled](https://github.com/tessel/firmware/issues/28) between code pushes. There are four available sockets on the TI CC3000 and we need to make sure they are sanitized and ready to go after every code push. This task is not for the faint of heart.

- [Add Duplex Stream Support to UART](https://github.com/tessel/firmware/issues/31). Currently, our UART API receives `data` events and has a `write` function. Ideally, you would be able to pipe data to and from UART. 

- [Tessel freezes with I2C Buffer of length 0](https://github.com/tessel/firmware/issues/29). Trouble for obvious reasons. 

- [Set a static IP Address on the Tessel](https://github.com/tessel/firmware/issues/35) using TI's CC3k API. 