# ::Goodbye macOS Server, You Served Me Well::

## Step 2: Setting up an NTP Service for Your Network

Network authentication, especially with Kerberos, is dependent on having reasonably accurate clocks to prevent replay attacks and other methods of stealing credentials. To keep the time synced on the machines in your network you'll need an NTP (Network Time Protocol) service running.

### Installing an NTP Implementation

In previous distribution releases, openSUSE used the `ntpd` implementation from ntp.org. Unfortunately, that implementation is known for having frequently discovered security vulnerabilities. Because of this, openSUSE has moved to using the newer Chrony NTP tools. This service uses only one package:

```
 - chrony
```

### Configuring Chrony - Server
 
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

Once configured, set it to start at boot, and then start it up for systems on your network to use it:

```shell
systemctl enable chrony.service
systemctl start chrony.service
```

### Configuring Chrony - Client

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

Once configured on the client, enable it to start at boot, and then start it up:

```shell
systemctl enable chrony.service
systemctl start chrony.service
```

### Further Information

For further information about NTP, look [here](http://www.ntp.org), and for info on Chrony, see these man pages:
- [chronyc](https://linux.die.net/man/1/chronyc)
- [chronyd](https://linux.die.net/man/8/chronyd)
- [chrony.conf](https://linux.die.net/man/5/chrony.conf)


---

## Navigation:

* [Home](https://greeneg.github.io)
* [Previous Step => Install openSUSE Leap 15](install_opensuse_leap_15.md)
* [Next Step => Setting up MIT Kerberos](setup_mit_krb5.md)
