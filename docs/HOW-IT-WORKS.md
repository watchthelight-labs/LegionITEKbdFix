# How LegionITEKbdFix works

## The device

The Legion 5 15IAX10's internal keyboard is not on PS/2 for typing: the ITE 8258 keyboard controller is a USB device, `048D:C195`, on the PCH xHCI port `HS05`. It is a composite device with two HID interfaces:

- interface 0: keyboard, consumer control keys, radio controls and vendor collections;
- interface 1: HID usage page 0x59 (Lighting and Illumination), usage 1 (LampArray), the per-key RGB lighting.

Windows shows both and drives the lighting through Lenovo's software. (The DSDT also declares a PS/2 keyboard device, `IDEA0100` with `_CID PNP0303`; Windows binds its 8042 driver to it and shows no PS/2 mouse. Which path carries the letter keys was an open question until the USB fix made typing work.)

## The failure

macOS's HID driver (`IOUSBHostHIDDevice`, user-space `IOUserUSBHostHIDDevice` on recent systems) attaches to every HID interface and starts reading its interrupt IN endpoint. On the lighting interface every read fails (`0xe00002ed`, as logged on the Legion Pro 5 16IRX9 with the ITE 8295 `048D:C995`), the driver retries ten times, then resets the whole USB device. The reset takes the keyboard interface down with it, the device re-enumerates, and the cycle starts again about every 0.6 s. The keyboard appears in `ioreg` but never delivers a key.

## The fix

IOKit matching picks, for each `IOUSBHostInterface` nub, the personality with the highest probe score whose keys match. This kext's personality matches vendor 0x048D, product 0xC195, configuration 1, interface 1 with a probe score of 100000 and names `IOService` as its class. `IOService` attaches and does nothing: no endpoint is opened, nothing is polled, nothing fails, nothing is reset. Interface 0 is not matched and keeps Apple's HID driver.

The cost is that nothing in macOS can talk to the lighting interface, so the keyboard backlight cannot be changed from macOS. A real driver for the LampArray interface (or a VirtualSMC plugin exposing it as a keyboard backlight) would replace this kext.

## Checking

```
ioreg -p IOUSB -l -w0 | grep -A3 -i 'C195'
ioreg -r -c IOUSBHostInterface -l -w0 | grep -B2 -A8 'bInterfaceNumber" = 1'
```

Interface 1 should show `IOClass = IOService`. If the keyboard still does not type, check the port first: on this laptop `HS05` is marked not connectable by the firmware's USB table and needs a `_UPC` patch before macOS powers it at all.
