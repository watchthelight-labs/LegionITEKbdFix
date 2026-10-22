# LegionITEKbdFix

A codeless kext (one `Info.plist`, no binary) for Lenovo Legion laptops whose internal keyboard is an ITE 8258 USB controller (`048D:C195`). The device has two HID interfaces: interface 0 is the keyboard, interface 1 is the keyboard lighting (HID usage page 0x59, LampArray). macOS's HID driver opens both and polls the lighting interface, every read fails, and after ten failures the driver resets the whole USB device, every half second or so. The keyboard enumerates but never types. This kext claims interface 1 with a plain `IOService` so Apple's HID driver never touches it, and the keyboard on interface 0 works.

## Status

| What | Tested on | Result |
|---|---|---|
| Internal keyboard (ITE 8258, `048D:C195`) | Legion 5 15IAX10, macOS 15.4.1 Recovery, 15.8.1 and 26.7.1 (25G241), OpenCore 1.0.8 | typing works since 2026-10-15; without the kext the keyboard enumerated and never typed |

## What it can't do

- Keyboard lighting cannot be controlled from macOS: the lighting interface is claimed and left alone. The backlight stays in whatever state the firmware boots it to.
- It is specific to `048D:C195` interface 1. Other ITE keyboard controllers (the Legion Pro 5's `048D:C995`, for example) need their own product ID in the plist.
- The reset loop itself was not captured in a log on this laptop; the symptom and the fix match the Legion Pro 5 16IRX9 case this was modelled on, and the kext fixed it here on the first try.

## Installation

1. Copy `LegionITEKbdFix.kext` to `EFI/OC/Kexts/` and add it to `config.plist`, `Kernel > Add`:

   ```
   BundlePath      LegionITEKbdFix.kext
   ExecutablePath  (empty)
   PlistPath       Contents/Info.plist
   MinKernel       (empty)
   MaxKernel       (empty)
   Enabled         true
   ```

2. Make sure the USB port the keyboard sits on is enabled in your USB map. On the Legion 5 15IAX10 it is `HS05`, which Lenovo's firmware marks as not connectable; the `_UPC` patch for it is in [Legion-5-15IAX10-OpenCore](https://github.com/watchthelight-labs/Legion-5-15IAX10-OpenCore).
3. Reboot and check:

   ```
   ioreg -p IOUSB -l -w0 | grep -A3 -i 'ITE\|C195'
   ioreg -r -c IOUSBHostInterface -l -w0 | grep -B2 -A8 'bInterfaceNumber" = 1' | grep -E 'IOClass|LegionITE'
   ```

   Interface 1 should show `IOClass = IOService` from this kext; interface 0 keeps `IOUSBHostHIDDevice`.
4. Rollback: set `Enabled` to false. No code runs, so the worst case is the keyboard not typing again.

## Properties

| Key | Value |
|---|---|
| `IOProviderClass` | `IOUSBHostInterface` |
| `idVendor` / `idProduct` | 1165 / 49557 (`048D:C195`) |
| `bConfigurationValue` / `bInterfaceNumber` | 1 / 1 |
| `IOClass` | `IOService` |
| `IOProbeScore` | 100000 (above Apple's HID driver) |

## How it works

IOKit picks the driver with the highest probe score for each interface nub. A personality that names `IOService` as its class with a high score wins the lighting interface and does nothing with it, so no HID driver opens the endpoint that fails. See [docs/HOW-IT-WORKS.md](docs/HOW-IT-WORKS.md).

## Credits

- [legion-pro5-hackintosh](https://github.com/Ammarrrrrrr/legion-pro5-hackintosh) by Ammarrrrrrr: the `LegionRGBUSBFix.kext` pattern for the ITE 8295 (`048D:C995`), which this copies for the 8258. Only the matching keys were reused.

## Licence

ISC, see `LICENSE`.
