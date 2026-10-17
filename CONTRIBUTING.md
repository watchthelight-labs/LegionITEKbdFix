# Contributing

If your Legion has a different ITE keyboard controller, open an issue with the USB vendor and product IDs, the interface layout (`ioreg -p IOUSB -l -w0` around the device, and `lsusb -v` or USB Tree View output from Windows or Linux if you have it), and whether the keyboard typed before and after. A log showing the reset loop (`log show --predicate 'subsystem == "com.apple.iokit.IOUSBHostFamily"'`) would also settle the part this README calls unverified.

Pull requests: small, with the machine you tested on. AI-assisted PRs are fine as long as you can explain the change, which is the same bar I hold my own code to.
