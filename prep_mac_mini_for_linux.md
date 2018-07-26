# ::Goodbye macOS Server, You Served Me Well::

## Step 0: Prep the Mac Mini for Linux

This is the preparation phase of the article about converting a Mac Mini to run Linux as a replacement for macOS Server that linked to this page.

**NOTE:** The steps below will **remove** your local macOS installation. If there is **ANYTHING** you wanted to keep, back it up somewhere else first!

OK, first, Apple doesn't really intend you to install other operating systems on their hardware. So, to facilitate a smooth install, the following should be done:

1. Install rEFInd Bootloader in the ESP. You can find the installer for it off the developer's website [here](http://www.rodsbooks.com/refind/).
2. Boot into the openSUSE DVD and select the Rescue option from the boot menu
3. When in the rescue shell, mount the ESP at /mnt
4. Traverse into /mnt/EFI/BOOT and rename the BOOTX64.efi to OLD-BOOTX64.efi
5. Change to /mnt/EFI and then rename the BOOT directory OLD
6. Using ```parted```, remove the macOS HFS+ partition

Now that these steps have been completed, reboot into the normal installation environment from the openSUSE DVD or Net-Inst CDROM.


---

## Navigation:

* [Home](https://greeneg.github.io/)
* [Next Step => Installing openSUSE Leap 15](install_opensuse_leap_15.md)
