# Firmware Details

When Tessel boots, firmware gets loaded into flash by way of the [bootloader](#bootloader).

The firmware layer is responsible for:

- [Transcieving data from peripherals over the SPI, UART, and I2C communication Buses](. 
- Communinicating with the host computer over USB
- Managing and updating the WiFi chip (which includes communicating over the SPI bus)
- Queueing and firing events from the event queue
- Managing hardware interrupts
- Receiving commands from the Lua VM
- Putting the processor to sleep when it's not being used

## Bootloader

## Communication Buses 

## USB Communication

## Managing the CC3k WiFi Chip

## The Event Queue

## Hardware Interrupts

## Interacting with the Lua VM

## Power Reduction Modes

