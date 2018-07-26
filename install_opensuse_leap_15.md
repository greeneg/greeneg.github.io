# ::Goodbye macOS Server, You Served Me Well::

## Step 1: Install openSUSE

The install of openSUSE Leap 15 normally is pretty pedestrian. Unfortunately in my case, I didn't have a copy of 15 around, only a DVD of 42.1. Normally, this wouldn't be an issue, _except_ that 42.1 is EOL, and no longer mirrored. So off to hunting down a mirror that still has this older version available.

The site ```mirrors.vtti.vt.edu```, in Virginia, USA still seems to have a few of the older openSUSE releases present, and so after pointing my net install disk at it, the Mac Mini was able to install that release of the OS, using the configuration [here](https://github.com/greeneg/website/blob/master/configuration/autoinst.xml).

**NOTE:** This autoinst.xml has NO password set for the root user, allowing login with just the root username. Once the installation is finished, <u>after boot, please run `passwd` after logging into the `root` user account to close this security hole</u>.

After this, I ran a series of distribution updates (42.1 -> 42.2 -> 42.3 -> 15.0) to get to having openSUSE Leap 15 running on the Mac Mini.

* [openSUSE Website](http://www.opensuse.org)
* [openSUSE Release Notes](https://en.opensuse.org/Portal:15.0?pk_campaign=counter)
* [openSUSE Documentation](https://doc.opensuse.org)
* [openSUSE Support](https://en.opensuse.org/Portal:Support)

* [Next Step => Setting up an NTP Service for Your Network](./setup_ntp.md)