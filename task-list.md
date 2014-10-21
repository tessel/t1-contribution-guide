# The Task List

This list contains improvements that would substantially increase the value and usability of the Tessel ecosystem. We'd love help getting these additions complete and are happy to help advise if you're interested in picking one up! See the [contribution process](./contribution-process.md) for more details on how to contribute.

You can chat with us at #tessel on IRC. 

## Finding an Approachable Task

For each of Tessel's repositories, we've added the 'help wanted' flag for issues that we feel require less hardware-specific knowledge to fix and are a great way to start contributing. Check out the [firmware](https://github.com/tessel/firmware/labels/help%20wanted), [runtime](https://github.com/tessel/runtime/labels/help%20wanted), and [CLI](https://github.com/tessel/cli/labels/help%20wanted) contribution tasks or, for each hardware module, check out the issue tags for under each module's repository. The CLI has the least hardware-specific tasks for developers who aren't quite ready to get their toes wet with the bare metal.

Below we've listed some of the most important issues that we'd appreciate help with:

### Runtime

- [JavaScript Typed Array Support](https://github.com/tessel/runtime/issues/254). These sized arrays would be really helpful for lower-level hardware.

- [Improving JavaScript Date compatibility](https://github.com/tessel/runtime/labels/Date-Incompatibility). There are a handful of reported issues where our Date implementation is not compatible with ES5.1 or they simply don't work. Our Date implementation code can be found in [`colony-js.lua`](https://github.com/tessel/runtime/blob/master/src/colony/lua/colony-js.lua#L1366).
]

### Firmware


- [Tessel Freezes with I2C Buffer of Length 0](https://github.com/tessel/firmware/issues/29). Trouble for obvious reasons. A very easy fix involves checking the length of the transmit buffer and returning if it's zero. 

- [Set a Static IP Address on Tessel](https://github.com/tessel/firmware/issues/35) using [TI's CC3k API](http://processors.wiki.ti.com/index.php/CC3000_Host_Programming_Guide#MAC_Address_update_process.). This will require more patience and/or experience with embedded firmware. 

- [Adding Pin Name As Pin Property](https://github.com/tessel/firmware/issues/30). Currently, calling `Port.pwm`, `Port.digital`, or `Port.analog` will return an array of pins. Unfortunately, those pins don't have a name property (like 'G1' or 'A3') so it's difficult to distinguish which you want to use.
