# ::Goodbye macOS Server, You Served Me Well::

## Step 3: Setting up MIT Kerberos

To build a secure Open Directory replacement, we start with MIT's Kerberos v5 service. During the install of openSUSE, I've already installed the required packages:

```
 - krb5
 - krb5-client
 - krb5-server
 - sssd-krb5
 - sssd-krb5-common
```

### Configuring the Krb5 General settings

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

### Create the Kerberos Database and Adjusting the `kdc.conf` File

After setting up the /etc/krb5.conf, run the following command and follow the prompts:

```shell
kdb5_util create -s
```

After running this command, verify `/var/lib/kerberos/krb5kdc/kdc.conf`, in particular, the `key_stash_file` setting. This setting should look like this, reflecting your DNS realm:

```
key_stash_file = /var/lib/kerberos/krb5kdc/.k5.TOLHARADYS.NET
```

The other settings in the file should be fine as the defaults in the file. If however, you want to tailor the Krb5 services on your machine as you need, take a look at the (man page for kdc.conf)[http://web.mit.edu/kerberos/krb5-1.12/doc/admin/conf_files/kdc_conf.html].

### Setting Up the Administrative Accesses

Now, edit the `/var/lib/kerberos/krb5kdc/kadm5.acl` file to add the appropriate administrative ACL on the kerberos sevice:

```
*/admin@TOLHARADYS.NET  *
```

This setting configures the service to allow any principle that contains /admin at the end of it's name before the realm section to have full rights to administer it.

With this out of the way, lets create an admin principle:

```shell
kadmin.local -q "addprinc admin/admin"
```

This creates a default administrative principle with full rights to add, modify, and remove kerberos principles and policies. Additionally, we need to create a service principle for working with the administrative service by doing the following, modifying accordingly for your Kerberos system's fully qualified hostname:

```shell
kadmin.local -q "addprinc -randkey kadmin/wotan.tolharadys.net"
```

Once this is created, restart the `kadmind` service to ensure it uses this principle for it's own authorization.

```shell
systemctl reload kadmind.service
```

### Configuring the Kerberos Authorization for LDAP

After restarting the `kadmind` service, we can work on creating the starting user accounts needed for the LDAP environment. To start, we should add a host principle to ensure that the host can request TGTs for users from the KDC. To do so, run the following command, modifying the FQDN in the principle with your KDC's hostname:

```shell
kadmin.local -q "addprinc -randkey host/wotan.tolharadys.net"
```

Now, this principle needs added to the host's `/etc/krb5.keytab`, replacing the FQDN, like above:

```shell
kadmin.local -q "ktadd -k /etc/krb5.keytab host/wotan.tolharadys.net"
```

At this point, we can create a service principle and keytab that the OpenLDAP slapd daemon will use for authorizing kerberized connections. To do so, run the following commands:

```shell
kadmin.local -q "addprinc -randkey ldap/wotan.tolharadys.net"
kadmin.local -q "ktadd -k /etc/openldap/krb5.keytab ldap/wotan.tolharadys.net"
chgrp ldap /etc/openldap/krb5.keytab
chmod g+r /etc/openldap/krb5.keytab
```

To validate the Kerberos portion of the configuration, run the following:

```shell
systemctl restart kadmind.service
kinit admin/admin@TOLHARADYS.NET
klist
```

This should complete without error, and display something like the following on your terminal:

```
Ticket cache: DIR::/run/user/0/krb5cc/tkt
Default principal: admin/admin@TOLHARADYS.NET

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

---

## Navigation

* [Home](https://greeneg.github.io)
* [Previous Step => Setting up NTP for Your Network](setup_ntp.md)
* [Next Step => Setting up OpenLDAP](setup_openldap.md)
