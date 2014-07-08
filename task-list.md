# The Task List

This list contains bug fixes or features that would substantially increase the value and usability of the Tessel ecosystem. We'd love help getting these additions complete and are happy to help advise if you're interested in picking one up! See the [contribution process](./contribution-process.md) for more details on how to contribute.

- [Allow Lua bytecode to be sent to the device directly](https://github.com/tessel/cli/issues/111). Currently, Tessel compiles JavaScript to Lua before sending a tarball of lua bytecode to the device to run on our forked version of the Lua VM. Many Lua users have been requesting to be able to write in Lua directly.

- [Allow coffee script to be used out of the box](https://github.com/tessel/cli/issues/99).Preprocess coffee script before executing colony transpilation with a command line flag. 