# :My Personal Site:

This site is my personal collection of howtos and ruminations. Anything I post here is either my opinion, or based off experiences building out opsn source tooling for personal infrastructure projects.

## ::Goodbye macOS Server, You Served Me Well::

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

#### Configuring Chrony - Server

On a machine that will act as an NTP server that has had Chrony installed, we'll need to configure the installation to allow syncing from an upstream public server, and allowing local network hosts to sync against it. In my case, I chose to use my network's network management server that runs DNS, DHCP, and acts as the Shorewall firewall for my network to host this service. To configure Chrony, I've modified the `/etc/chrony.conf` file to fit my network's needs:

```
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
pool 0.us.pool.ntp.org iburst
pool 1.us.pool.ntp.org iburst
pool 2.us.pool.ntp.org iburst
pool 3.us.pool.ntp.org iburst

# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift

# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second.
makestep 1.0 3

# Enable kernel synchronization of the real-time clock (RTC).
rtcsync

# Allow NTP client access from local network.
allow 192.168.8.0/24

# Specify directory for log files.
logdir /var/log/chrony

# Also include any directives found in configuration files in /etc/chrony.d
include /etc/chrony.d/*.conf
```

#### Configuring Chrony - Client

To configure Chrony, either we can use the `yast2 ntp-client` ncurses configuration tool, or modify the `/etc/chrony.conf` file directly, like so:

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

# Specify directory for log files.
logdir /var/log/chrony

# Also include any directives found in configuration files in /etc/chrony.d
include /etc/chrony.d/*.conf
```

For further information about NTP, look [here](http://www.ntp.org), and for info on Chrony, see these man pages:
- [chronyc](https://linux.die.net/man/1/chronyc)
- [chronyd](https://linux.die.net/man/8/chronyd)
- [chrony.conf](https://linux.die.net/man/5/chrony.conf)

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

### Step 4: Setting up OpenLDAP

#### Configuring the Kerberos Authorization for LDAP

After restarting the `kadmind` service, we can work on creating the starting user accounts needed for the LDAP environment. To start, we should add a host principle to ensure that the host can request TGTs for users from the KDC. To do so, run the following command, modifying the FQDN in the principle with your KDC's hostname:

```
kadmin.local -q "addprinc -randkey host/wotan.tolharadys.net"
```

Now, this principle needs added to the host's `/etc/krb5.keytab`, replacing the FQDN, like above:

```
kadmin.local -q "ktadd -k /etc/krb5.keytab host/wotan.tolharadys.net"
```

At this point, we can create a service principle and keytab that the OpenLDAP slapd daemon will use for authorizing kerberized connections. To do so, run the following commands:

```
kadmin.local -q "addprinc -randkey ldap/wotan.tolharadys.net"
kadmin.local -q "ktadd -k /etc/openldap/krb5.keytab ldap/wotan.tolharadys.net"
chgrp ldap /etc/openldap/krb5.keytab
chmod g+r /etc/openldap/krb5.keytab
```

To validate the Kerberos portion of the configuration, run the following:

```
systemctl restart kadmind.service
kinit diradmin/admin@TOLHARADYS.NET
klist
```

This should complete without error, and display something like the following on your terminal:

```
Ticket cache: DIR::/run/user/0/krb5cc/tkt
Default principal: diradmin/admin@TOLHARADYS.NET

Valid starting       Expires              Service principal
07/19/2018 19:00:24  07/20/2018 05:00:24  krbtgt/TOLHARADYS.NET@TOLHARADYS.NET
    renew until 07/26/2018 19:00:24
```

Further information on Kerberos v5 and the MIT implementation, specifically can be found on the links below:
- [kadmin](https://linux.die.net/man/1/kadmin)
- [kadmind](https://linux.die.net/man/8/kadmind)
- [kdb5_util](https://linux.die.net/man/8/kdb5_util)
- [klist](https://linux.die.net/man/1/klist)
- [kinit](https://linux.die.net/man/1/kinit)
- [krb5.conf](https://linux.die.net/man/5/krb5.conf)
- [krb5kdc](https://linux.die.net/man/8/krb5kdc)
- [MIT Kerberos v5 Documentation](http://web.mit.edu/kerberos/krb5-current/doc/)

#### Installing OpenLDAP

The next step in replacing your Open Directory server is to install the Cyrus SASL authorization suite, the OpenLDAP suite of tools, and related tooling:

```
 - cyrus-sasl
 - cyrus-sasl-gs2
 - cyrus-sasl-gssapi
 - cyrus-sasl-ldap-auxprop
 - cyrus-sasl-ntlm
 - cyrus-sasl-otp
 - cyrus-sasl-plain
 - cyrus-sasl-saslauthd
 - cyrus-sasl-scram
 - ldapvi
 - openldap2
 - openldap2-client
 - openldap2-contrib
 - openldap2-doc
 - openldap2-ppolicy-check-password
 - sssd-ldap
 - sssd-proxy
 - sssd-tools
```

#### Configuring OpenLDAP

There are a number of files to modify for `slapd`, the LDAP daemon portion of OpenLDAP to work. Additionally, some of these files also govern the configuration of the local LDAP client tools we installed above.

First, let's modify the `/etc/openldap/ldap.conf` so the tools know your LDAP directory name (modify appropriate for your environment):

```
BASE    dc=tolharadys,dc=net
URI     ldap://localhost
```

We'll return to this file later to set up default GSSAPI and, if desired, TLS settings as we progress. But before then, let's get our LDAP schema in order.

#### Obtain, Modify, and Configure Your OpenLDAP to Use Needed Schema

Under the hood, most directory services are a mix of LDAP and Kerberos, with a mix of extra tooling to add features to differentiate one from the other, like group policy, or key management. In most of these cases, vendors of directory service solutions have created or modified the base LDAP schemas to meet their needs. In Apple's case, they chose to augment the base [RFC2307bis schema](https://tools.ietf.org/html/draft-howard-rfc2307bis-02) to add support for fields that once NetInfo provided, and some compatibility elements from Microsoft's Active Directory schema.

In addition, I've chosen to add support for other Linux related services to the directory service we're building. The list of schema, at this time, we'll be working with follows:

```
 - core
 - cosine
 - inetorgperson
 - openldap
 - rfc2307bis
 - apple_auxillary
 - samba3
 - apple
 - sudo
 - openssh-lpk
 - yast
```

The schema files from Apple are unfortunately not marked directly with a license. However, they _ARE_ available off their open source site online, so I'm going to assume (at least until I get a cease and desist) that their schemas are free to redistribute.

The files above are in the tar.bzip2 linked from the site [here](https://github.com/greeneg/website/blob/master/configuration/ldap_schema.tar.bz2).