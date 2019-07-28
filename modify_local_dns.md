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

For example,
