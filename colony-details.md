# Colony Details

Colony is a node module (colony-compiler) responsible for:

- [Compiling JavaScript to Lua](#compiling-js-to-lua)
- [Maintaining a forked version of Lua](./runtime-details#lua-fork)

## Compiling JS to Lua

The Colony compiler will take a JS file and transpile it to a version of Lua that is ideal for our fork of the Lua 5.1 compiler.
