# ./My Personal Site...

This site is my personal collection of howtos and ruminations. Anything I post here is either my opinion, or based off experiences building out opsn source tooling for personal infrastructure projects.

## ./Goodbye macOS Server, You Served Me Well?

Over the last several years, I've used a Mac Mini running OS X and OS X Server to manage the identity information for my personal network. Unfortunately, Apple has decided not to allow upgrades for the model of Mini that I'm using. To keep my server from having security and other nasty issues, I've decided that while it was easy to maintain, it's time for the machine to no longer run OS X, and by extension OS X Server.

This left me in a bit of a cunnundrum, since my home network has a few Macs and a Windows box, plus several Linux systems, all of which need identity data, and other services that that machine ran. After thinking it through, I decided to move that Mini to use openSUSE Leap 15, and use the array of open source services on the machine to fill the gap that used to be managed by OS X Server.

### Step 0: Prep the Mac Mini for Linux

**NOTE:** The steps below will _remove_ your local macOS installation. If there was ANYTHING you wanted to keep, back it up somewhere else!

OK, first, Apple doesn't really intend you to install other operating systems on their hardware. So, to facilitate a smooth install, the following should be done:

1. Install rEFInd Bootloader in the ESP. You can find the installer for it off the developer's website [here](http://www.rodsbooks.com/refind/).
2. Boot into the openSUSE DVD and select the Rescue option from the boot menu
3. When in the rescue shell, mount the ESP at /mnt
4. Traverse into /mnt/EFI/BOOT and rename the BOOTX64.efi to OLD-BOOTX64.efi
5. Change to /mnt/EFI and then rename the BOOT directory OLD
6. Using ```parted```, remove the macOS HFS+ partition

Now that these steps have been completed, reboot into the normal installation environment.

### Step 1: Install openSUSE

The install of openSUSE Leap 15 normally is pretty pedestrian. Unfortunately in my case, I didn't have a copy of 15 around, only a DVD of 42.1. Normally, this wouldn't be an issue, _except_ that 42.1 is EOL, and no longer mirrored. So off to hunting down a mirror that still has this older version available.

The site ```mirrors.vtti.vt.edu```, in Virginia, USA still seems to have a few of the older openSUSE releases present, and so after pointing my net install disk at it, the Mac Mini was able to install that release of the OS, using the configuration [here](configuration/autoinst.xml).