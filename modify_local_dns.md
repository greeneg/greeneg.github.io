# ::Goodbye MacOS Server, You Served Me Well::

## Step 6: Add DNS Records for Auto-discovery

The next step in the build of a Linux-based Open Directory is to populate your local DNS services with the required records to allow clients to automatically discover your Kerberos and LDAP services. Note, this requires you to have a LOCAL DNS server, and not rely on the one that your ISP might afford you access to. This is needed to ensure that information about your domain is not discoverable outside your network.

### Why DNS?

Adding a couple of `SRV` records to DNS allows client tooling to automatically discover your Kerberos and LDAP installation, without having to pass command-line flags each time you run a command. While this can be added locally, this allows for quicker bootstrapping of new systems within your network.

### Preparing your DNS Server

In my network, I run a local ISC BIND 9 installation. Unfortunately, BIND 9 by default does not like the format of `SRV` records, and will not load "invalid" zones with them in them. To work around this, add the following line to your /etc/named.conf in the zone definition for your domain:

```
check-names: ignore;
```

For my domain, I have the following internal zone definition stanza:

```
zone "tolharadys.net" {
    type master;
    file "/etc/named.d/tolharadys.net";
    check-names ignore;
    allow-query { internals; };
    allow-update { key DHCP_UPDATER; };
};
```

### The Kerberos TXT Record

The `TXT` record for Kerberos automatic realm discovery looks like this:

```zone
_kerberos IN TXT "TOLHARADYS.NET"
```

### Service Records: Kerberos and LDAP Server Automatic Discovery

Service Resource Records, or `SRV` records, have a very specific syntax to allow client tools to query for them and easily parse the responses. The syntax looks like so:

```
  [_SERVICE] [TTL] [CLASS] SRV [PRIORITY] [WEIGHT] [PORT] [TARGET]

```

For example, lets look at Kerberos and LDAP section for my network's zone file:

```zone
_kerberos._tcp                          IN SRV  0  100 88   wotan
_kerberos._udp                          IN SRV  0  100 88   wotan
_kerberos-adm._tcp                      IN SRV  0  100 749  wotan
_kerberos-adm._udp                      IN SRV  0  100 749  wotan
_kerberos-master._tcp                   IN SRV  0  100 88   wotan
_kerberos-master._udp                   IN SRV  0  100 88   wotan
_kpasswd._tcp                           IN SRV  0  100 464  wotan
_kpasswd._udp                           IN SRV  0  100 464  wotan
_ldap._tcp                              IN SRV  0  100 389  wotan
_ldap._udp                              IN SRV  0  100 389  wotan
```

In my network, the primary hostname of my Kerberos and LDAP master is wotan.tolharadys.net, which is only accessible from inside. The records above tell applications using kerberos libraries to look for the TCP and UDP ports for the Kerberos service on `wotan`, on port 88. Similarly, the kadmin service on `wotan` is on TCP and UDP port 749, and the KDC master is also on `wotan` for tools that understand the difference between slave and master Kerberos topology.

Similarly, the KPasswd service, which is used for users to change their Kerberos password with the `kpasswd` utility is accessible via TCP and UDP at port 464 on `wotan`.

And, finally, LDAP without TLS encryption is available at port 389 on `wotan`.

All of these are listed with a 0 priority, as they are all hosted on the same host. If there were replicas of each service, this would have multiple entries with different priority levels. Additionally, each of these records have a weight of 100. As there is only one host running these services, the `weight` value is not really important, however, again if the services were replicated on multiple hosts, this number would allow breaking ties for which instance should be looked at first when querying for the service.


* [Home](https://greeneg.github.io)
* [Previous Step => Populating Your Directory](populating_your_ldap.md)
