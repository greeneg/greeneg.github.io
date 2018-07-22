# ./My Personal Site...

This site is my personal collection of howtos and ruminations. Anything I post here is either my opinion, or based off experiences building out opsn source tooling for personal infrastructure projects.

## ./Goodbye macOS Server, You Served Me Well?

**THIS ARTICLE IS A WORK IN PROGRESS! There are things that will change as I work out issues.**

Over the last several years, I've used a Mac Mini running OS X and OS X Server to manage the identity information for my personal network. Unfortunately, Apple has decided not to allow upgrades for the model of Mini that I'm using. To keep my server from having security and other nasty issues, I've decided that while it was easy to maintain, it's time for the machine to no longer run OS X, and by extension OS X Server.

This left me in a bit of a cunnundrum, since my home network has a few Macs and a Windows box, plus several Linux systems, all of which need identity data, and other services that that machine ran. After thinking it through, I decided to move that Mini to use openSUSE Leap 15, and use the array of open source services on the machine to fill the gap that used to be managed by OS X Server.

### Step 0: Prep the Mac Mini for Linux

**NOTE:** The steps below will **remove** your local macOS installation. If there is **ANYTHING** you wanted to keep, back it up somewhere else first!

OK, first, Apple doesn't really intend you to install other operating systems on their hardware. So, to facilitate a smooth install, the following should be done:

1. Install rEFInd Bootloader in the ESP. You can find the installer for it off the developer's website [here](http://www.rodsbooks.com/refind/).
2. Boot into the openSUSE DVD and select the Rescue option from the boot menu
3. When in the rescue shell, mount the ESP at /mnt
4. Traverse into /mnt/EFI/BOOT and rename the BOOTX64.efi to OLD-BOOTX64.efi
5. Change to /mnt/EFI and then rename the BOOT directory OLD
6. Using ```parted```, remove the macOS HFS+ partition

Now that these steps have been completed, reboot into the normal installation environment from the openSUSE DVD or Net-Inst CDROM.

### Step 1: Install openSUSE

The install of openSUSE Leap 15 normally is pretty pedestrian. Unfortunately in my case, I didn't have a copy of 15 around, only a DVD of 42.1. Normally, this wouldn't be an issue, _except_ that 42.1 is EOL, and no longer mirrored. So off to hunting down a mirror that still has this older version available.

The site ```mirrors.vtti.vt.edu```, in Virginia, USA still seems to have a few of the older openSUSE releases present, and so after pointing my net install disk at it, the Mac Mini was able to install that release of the OS, using the configuration [here](https://github.com/greeneg/website/blob/master/configuration/autoinst.xml).

**NOTE:** This autoinst.xml has NO password set for the root user, allowing login with just the root username. Once install is finished, <u>after boot, please run `passwd` after logging into the `root` user account</u>.

After this, I ran a series of distribution updates (42.1 -> 42.2 -> 42.3 -> 15.0) to get to having openSUSE Leap 15 running on the Mac Mini.

### Step 2: Setting up an NTP service for your network

Network authentication, especially with Kerberos, is dependent on having reasonably accurate clocks to prevent replay attacks and other methods of stealing credentials. To keep the time synced on the machines in your network you'll need an NTP (Network Time Protocol) service running.

#### Installing an NTP Implementation

In previous distribution releases, openSUSE used the `ntpd` implementation from ntp.org. Unfortunately, that implementation is known for having frequently discovered security vulnerabilities. Because of this, openSUSE has moved to using the newer Chrony NTP tools. This service uses only one package:

```
 - chrony
```

#### Configuring Chrony

To configure Chrony, either we can use the `yast2 ntp-client` ncurses configuration tool, or modify the `/etc/chrony.conf` file directly. As this howto will eventually be scripted, we'll look at modifying the file directly.

Open `/etc/chrony.conf` and ensure it's content looks like so, but change the host that acts as your NTP master in the pool section:

```
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
pool ns1.tolharadys.net iburst

# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift

# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second.
makestep 1.0 3

# Enable kernel synchronization of the real-time clock (RTC).
rtcsync

# Enable hardware timestamping on all interfaces that support it.
#hwtimestamp *

# Increase the minimum number of selectable sources required to adjust
# the system clock.
#minsources 2

# Allow NTP client access from local network.
#allow 192.168.0.0/16

# Serve time even if not synchronized to a time source.
#local stratum 10

# Specify file containing keys for NTP authentication.
#keyfile /etc/chrony.keys

# Get TAI-UTC offset and leap seconds from the system tz database.
#leapsectz right/UTC

# Specify directory for log files.
logdir /var/log/chrony

# Select which information is logged.
#log measurements statistics tracking

# Also include any directives found in configuration files in /etc/chrony.d
include /etc/chrony.d/*.conf
```

### Step 3: Setting up MIT Kerberos

To build a secure Open Directory replacement, we start with MIT's Kerberos v5 service. During the install of openSUSE, I've already installed the required packages:

```
 - krb5
 - krb5-client
 - krb5-server
 - sssd-krb5
 - sssd-krb5-common
```

#### Configuring the Krb5 General settings

With the required packages installed, the host's configuration needs set to create the key database that will store the kerberos principles and keys. The first file to modify is `/etc/krb5.conf`. It should look something like this, but modified to reflect your realm's DNS domain:

```
[libdefaults]
    default_realm = TOLHARADYS.NET
    clockskew = 300
    # "dns_canonicalize_hostname" and "rdns" are better set to false for improved security.
    # If set to true, the canonicalization mechanism performed by Kerberos client may
    # allow service impersonification, the consequence is similar to conducting TLS certificate
    # verification without checking host name.
    # If left unspecified, the two parameters will have default value true, which is less secure.
    dns_canonicalize_hostname = false
    rdns = false
    dns_lookup_kdc = true
    dns_lookup_realm = true
    forwardable = true
    proxiable = true

[realms]
    TOLHARADYS.NET = {
        kdc = wotan.tolharadys.net
        admin_server = wotan.tolharadys.net
    }

[domain_realm]
    .tolharadys.net = TOLHARADYS.NET
    tolharadys.net  = TOLHARADYS.NET

[logging]
    kdc = FILE:/var/log/krb5/krb5kdc.log
    admin_server = FILE:/var/log/krb5/kadmind.log
    default = SYSLOG:NOTICE:DAEMON

[plugins]
```

With these settings, the server will allow for the normal 5 minute clock leeway for checking ticket requests, allow TGTs to be forwarded or proxied, and look up both the realm and kdcs in said realm via DNS SRV styled records. While I do manually set the kdc and admin server, and set the domain to realm mapping, In the long run, I'll be moving to using DNS for this.

#### Create the Kerberos Database and Adjusting the `kdc.conf` File

After setting up the /etc/krb5.conf, run the following command and follow the prompts:

```
kdb5_util create -s
```

After running this command, verify `/var/lib/kerberos/krb5kdc/kdc.conf`, in particular, the `key_stash_file` setting. This setting should look like this, reflecting your DNS realm:

```
key_stash_file = /var/lib/kerberos/krb5kdc/.k5.TOLHARADYS.NET
```

The other settings in the file should be fine as the defaults in the file. If however, you want to tailor the Krb5 services on your machine as you need, take a look at the (man page for kdc.conf)[http://web.mit.edu/kerberos/krb5-1.12/doc/admin/conf_files/kdc_conf.html].

#### Setting Up the Administrative Accesses

Now, edit the `/var/lib/kerberos/krb5kdc/kadm5.acl` file to add the appropriate administrative ACL on the kerberos service:

```
*/admin@TOLHARADYS.NET  *
```

This setting configures the service to allow any principle that contains /admin at the end of it's name before the realm section to have full rights to administer it.

With this out of the way, lets create an admin principle:

```
kadmin.local -q "addprinc admin/admin"
```

This creates a default administrative principle with full rights to add, modify, and remove kerberos principles and policies. Additionally, we need to create a service principle for working with the administrative service by doing the following, modifying accordingly for your Kerberos system's fully qualified hostname:

```
kadmin.local -q "addprinc -randkey kadmin/wotan.tolharadys.net"
```

Once this is created, restart the `kadmind` service to ensure it uses this principle for it's own authorization.

```
systemctl reload kadmind.service
```
