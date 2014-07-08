# Firmware Details

When Tessel boots, firmware gets loaded into flash by way of the [bootloader](#bootloader).

The firmware layer is responsible for:

- [Transcieving data from peripherals over the SPI, UART, and I2C communication Buses](#communication-buses). 
- [Communinicating with the host computer over USB](#usb-communication)
- [Managing and updating the WiFi chip](#cc3k-wifi-chip)
- [Queueing and firing events from the event queue](#event-queue)
- [Managing hardware interrupts](#hardware-interrupts)
- [Receiving commands from the Tessel API](#tessel-api)
- [Communicating with the Lua VM](#integrating-with-runtime)
- [Putting the processor to sleep when it's not being used](#power-reduction-modes)

## Bootloader

## Communication Buses 

## USB Communication

## CC3k WiFi Chip

## Event Queue

## Hardware Interrupts

## Tessel API

## Integration with Runtime

## Power Reduction Modes

