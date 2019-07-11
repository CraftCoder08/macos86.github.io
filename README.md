## What is OpenCore?

OpenCore is an alternative to Clover. By design, OpenCore is versatile by being more modular and open as it aims to resolve the constraints and issues that Clover brings. It is not only for Hackintoshes as it can be used for other purposes that require an emulated EFI. Please remember we’re still in very early infancy so there will be issues. This specific guide will be omitting Vault.plist and Vault.sig as there's still quite a bit of development happening there. OpenCore should be considered in Alpha stage at this time. If you have a working, stable system you should not migrate unless you prefer "bleeding edge" development, want to contribute, and don't mind recovering your system should it fail to boot.

And please keep in mind that this guide is more of a jumping-off point, your hardware may have different requirements so please read the guide carefully 

## Current issues with OpenCore

* Order of kexts matter: Make sure to have kexts like Lilu and VoodooPS2Controller to be injected first before kexts that require them like WhateverGreen, VirtualSMC, keyboard/Mouse/Trackpad and etc
* Z97 based systems require pure UEFI mode for booting \(also known as Windows 8/10 mode\).
* NVMe issues if setup as a SATA device in BIOS.
* Sometimes can't access other partitions on the drive, solution is to "bless" the drive with Startup Disk
* ~~Currently minimal support for emulated NVRAM \(sorry z390 users, EmuVariableRuntimeDxe may work but won't save on reboot\).~~ Support for nvram.plist has been added to OpenCore but do keep in mind it's still quite finicky


# Hey why does this guide look quite similar to CorpNewt's Vanilla Guide?

Well, when I made the [original OpenCore guide](https://github.com/khronokernel/Getting-Started-With-OpenCore) I was quite disappointed in how the whole thing was set up. It felt clunky, confusing and quite easy to get lost in as hardware-specific changes were hard to see. So I took CorpNewt's [r/Hackintosh Vanilla Desktop Guide](https://hackintosh.gitbook.io/-r-hackintosh-vanilla-desktop-guide/) as a base and built it up from there. Full credit to him and also seriously a great guy, definitely check him out on [Github](https://github.com/corpnewt)(Don't worry I've talked to Corp about all this)

