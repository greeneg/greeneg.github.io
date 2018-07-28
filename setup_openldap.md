# ::Goodbye macOS Server, You Served Me Well::

## Step 4: Setting up OpenLDAP

### Installing OpenLDAP

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

### Configuring OpenLDAP CLI Tools

There are a number of files to modify for `slapd`, the LDAP daemon portion of OpenLDAP to work. Additionally, some of these files also govern the configuration of the local LDAP client tools we installed above.

First, let's modify the `/etc/openldap/ldap.conf` so the tools know your LDAP directory name (modify appropriate for your environment):

```
BASE    dc=tolharadys,dc=net
URI     ldap://localhost
```

We'll return to this file later to set up default GSSAPI and, if desired, TLS settings as we progress. But before then, let's get our LDAP schema in order.

### Download the Needed LDAP Schema, and Register Them

Under the hood, most directory services are a mix of LDAP and Kerberos, with a mix of extra tooling to add features to differentiate one from the other, like group policy, or key management. In most of these cases, vendors of directory service solutions have created or modified the base LDAP schemas to meet their needs. In Apple's case, they chose to augment the base [RFC2307bis schema](https://tools.ietf.org/html/draft-howard-rfc2307bis-02) to add support for fields that once NetInfo provided, and some compatibility elements from Microsoft's Active Directory schema.

In addition, I've chosen to add support for other Linux related services to the directory service we're building. The list of schema, at this time, we'll be working with follows:

```
 - core
 - cosine
 - nis
 - inetorgperson
 - fmserver
 - openldap
 - apple_auxillary
 - samba3
 - apple
 - sudo
 - openssh-lpk
 - yast
```

The schema files from Apple are unfortunately not marked directly with a license. However, they _ARE_ available off their open source site online, so I'm going to assume (at least until I get a cease and desist) that their schemas are free to redistribute.

The files above are in the tar.bzip2 linked from the site [here](https://github.com/greeneg/greeneg.github.io/blob/master/configuration/ldap_schema.tar.bz2?raw=true). Just for clarity, there are more schema in that tar.bz2 ball than what we'll be dealing with. In addition, there are a few changes to the Apple supplied schema to deal with parsing errors and correctness to allow slaptest generate LDIF output for the in-database configuration system.

To better understand the changes I've done, lets look at a diff from the Apple version of the apple.schema file and the one I've put in the tar bz2 ball, let's look at a diff:

```diff
--- /etc/openldap/schema/apple.schema 2018-06-09 03:29:58.000000000 -0400
+++ /etc/openldap/schema/apple.schema 2018-07-18 22:45:46.500811045 -0400
@@ -6,12 +6,12 @@
 #
 # Container structural object class.
 #
-#objectclass (
-#1.2.840.113556.1.3.23
-#NAME 'container'
-#SUP top
-#STRUCTURAL
-#MUST ( cn ) )
+objectclass (
+1.2.840.113556.1.3.23
+NAME 'container'
+SUP top
+STRUCTURAL
+MUST ( cn ) )

 #
 # Time to live
@@ -31,6 +31,25 @@
 MAY ( ttl ) )
 
 #
+# Authentication authority attribute 1.3.6.1.4.1.63.1000.1.1.2.16.1
+# 
+attributetype (
+1.3.6.1.4.1.63.1000.1.1.2.16.1
+NAME 'authAuthority'
+DESC 'password server authentication authority'
+EQUALITY caseExactIA5Match
+SUBSTR caseExactIA5SubstringsMatch
+SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
+
+#attributetype (
+#1.3.6.1.4.1.63.1000.1.1.2.16.2
+#NAME ( 'authAuthority' 'authAuthority2' )
+#DESC 'password server authentication authority'
+#EQUALITY caseExactMatch
+#SUBSTR caseExactSubstringsMatch
+#SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 )
+
+#
 # User attributes 1.3.6.1.4.1.63.1000.1.1.1.1
 #
 attributetype (
@@ -80,6 +99,7 @@
 #EQUALITY caseExactMatch
 #SUBSTR caseExactSubstringsMatch
 #SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 SINGLE-VALUE )
+
 attributetype (
 1.3.6.1.4.1.63.1000.1.1.1.1.16
 NAME ( 'apple-mcxsettings' 'apple-mcxsettings2' )
@@ -312,10 +332,9 @@

 attributetype (
 1.3.6.1.4.1.63.1000.1.1.1.1.41
-  NAME 'lastFailedLoginTime'
-  EQUALITY generalizedTimeMatch
-  SYNTAX '1.3.6.1.4.1.1466.115.121.1.24'
-  SINGLE-VALUE )
+NAME 'lastFailedLoginTime'
+EQUALITY generalizedTimeMatch
+SYNTAX '1.3.6.1.4.1.1466.115.121.1.24' SINGLE-VALUE )

 attributetype (
 1.3.6.1.4.1.63.1000.1.1.1.1.42
@@ -1142,25 +1161,6 @@
       apple-serviceslocator ) )
       
 #     
-# Authentication authority attribute 1.3.6.1.4.1.63.1000.1.1.2.16.1
-# 
-#attributetype (
-#1.3.6.1.4.1.63.1000.1.1.2.16.1
-#NAME 'authAuthority'
-#DESC 'password server authentication authority'
-#EQUALITY caseExactIA5Match
-#SUBSTR caseExactIA5SubstringsMatch
-#SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
-
-#attributetype (
-#1.3.6.1.4.1.63.1000.1.1.2.16.2
-#NAME ( 'authAuthority' 'authAuthority2' )
-#DESC 'password server authentication authority'
-#EQUALITY caseExactMatch
-#SUBSTR caseExactSubstringsMatch
-#SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 )
-
-#
 # Authentication authority object 1.3.6.1.4.1.63.1000.1.1.2.16
 #
 objectclass (
@@ -1404,44 +1404,44 @@
         STRUCTURAL
         MUST ( cn ) )
 
-attributetype (
-1.3.6.1.1.1.1.31
-NAME 'automountMapName'
-            DESC 'automount Map Name'
-            EQUALITY caseExactMatch
-            SYNTAX 1.3.6.1.4.1.1466.115.121.1.15
-            SINGLE-VALUE )
-            
-attributetype (
-1.3.6.1.1.1.1.32
-NAME 'automountKey'
-            DESC 'Automount Key value'
-            EQUALITY caseExactMatch
-            SYNTAX 1.3.6.1.4.1.1466.115.121.1.15
-            SINGLE-VALUE )
-            
-attributetype (
-1.3.6.1.1.1.1.33
-NAME 'automountInformation'
-            DESC 'Automount information'
-            EQUALITY caseExactMatch
-            SYNTAX 1.3.6.1.4.1.1466.115.121.1.15
-            SINGLE-VALUE )
-            
-objectclass (
-1.3.6.1.1.1.2.16
-NAME 'automountMap'
-SUP top STRUCTURAL
-            MUST ( automountMapName )
-            MAY description )
-            
-objectclass (
-1.3.6.1.1.1.2.17
-NAME 'automount'
-SUP top STRUCTURAL
-            DESC 'Automount'
-            MUST ( automountKey $ automountInformation )
-            MAY description )
+# attributetype (
+#1.3.6.1.1.1.1.31
+#NAME 'automountMapName'
+#            DESC 'automount Map Name'
+#            EQUALITY caseExactMatch
+#            SYNTAX 1.3.6.1.4.1.1466.115.121.1.15
+#            SINGLE-VALUE )
+
+# attributetype (
+#1.3.6.1.1.1.1.32
+#NAME 'automountKey'
+#            DESC 'Automount Key value'
+#            EQUALITY caseExactMatch
+#            SYNTAX 1.3.6.1.4.1.1466.115.121.1.15
+#            SINGLE-VALUE )
+
+# attributetype (
+#1.3.6.1.1.1.1.33
+#NAME 'automountInformation'
+#            DESC 'Automount information'
+#            EQUALITY caseExactMatch
+#            SYNTAX 1.3.6.1.4.1.1466.115.121.1.15
+#            SINGLE-VALUE )
+
+# objectclass (
+#1.3.6.1.1.1.2.16
+#NAME 'automountMap'
+#SUP top STRUCTURAL
+#            MUST ( automountMapName )
+#            MAY description )
+
+# objectclass (
+#1.3.6.1.1.1.2.17
+#NAME 'automount'
+#SUP top STRUCTURAL
+#            DESC 'Automount'
+#            MUST ( automountKey $ automountInformation )
+#            MAY description )

 #
 # Apple User Info object 1.3.6.1.4.1.63.1000.1.1.2.27
```

As can be seen, many of the changes are to comment out the AutoFS entries, which are provided by the rfc2307bis schema; uncomment the container objectclass, as this is required for conformance to how Open Directory is structured for the Workgroup Manager or other tools to work correctly; and move the `authAuthority` attribute definitions near the top of the schema.

Next we modified the samba3 schema to bring back a couple of legacy entries that the Apple schema required:

```diff
--- /etc/openldap/schema/samba3.schema 2018-05-14 19:05:11.000000000 -0400
+++ /etc/openldap/schema/samba3.schema 2018-07-18 22:39:18.751534299 -0400
@@ -71,33 +71,33 @@
 ##
 ## Account flags in string format ([UWDX     ])
 ##
-#attributetype ( 1.3.6.1.4.1.7165.2.1.4 NAME 'acctFlags'
-#DESC 'Account Flags'
-#EQUALITY caseIgnoreIA5Match
-#SYNTAX 1.3.6.1.4.1.1466.115.121.1.26{16} SINGLE-VALUE )
+attributetype ( 1.3.6.1.4.1.7165.2.1.4 NAME 'acctFlags'
+DESC 'Account Flags'
+EQUALITY caseIgnoreIA5Match
+SYNTAX 1.3.6.1.4.1.1466.115.121.1.26{16} SINGLE-VALUE )

 ##
 ## Password timestamps & policies
 ##
-#attributetype ( 1.3.6.1.4.1.7165.2.1.3 NAME 'pwdLastSet'
-#DESC 'NT pwdLastSet'
-#EQUALITY integerMatch
-#SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 SINGLE-VALUE )
-
-#attributetype ( 1.3.6.1.4.1.7165.2.1.5 NAME 'logonTime'
-#DESC 'NT logonTime'
-#EQUALITY integerMatch
-#SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 SINGLE-VALUE )
-
-#attributetype ( 1.3.6.1.4.1.7165.2.1.6 NAME 'logoffTime'
-#DESC 'NT logoffTime'
-#EQUALITY integerMatch
-#SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 SINGLE-VALUE )
-
-#attributetype ( 1.3.6.1.4.1.7165.2.1.7 NAME 'kickoffTime'
-#DESC 'NT kickoffTime'
-#EQUALITY integerMatch
-#SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 SINGLE-VALUE )
+attributetype ( 1.3.6.1.4.1.7165.2.1.3 NAME 'pwdLastSet'
+DESC 'NT pwdLastSet'
+EQUALITY integerMatch
+SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 SINGLE-VALUE )
+
+attributetype ( 1.3.6.1.4.1.7165.2.1.5 NAME 'logonTime'
+DESC 'NT logonTime'
+EQUALITY integerMatch
+SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 SINGLE-VALUE )
+
+attributetype ( 1.3.6.1.4.1.7165.2.1.6 NAME 'logoffTime'
+DESC 'NT logoffTime'
+EQUALITY integerMatch
+SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 SINGLE-VALUE )
+
+attributetype ( 1.3.6.1.4.1.7165.2.1.7 NAME 'kickoffTime'
+DESC 'NT kickoffTime'
+EQUALITY integerMatch
+SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 SINGLE-VALUE )

 #attributetype ( 1.3.6.1.4.1.7165.2.1.8 NAME 'pwdCanChange'
 #DESC 'NT pwdCanChange'
@@ -112,30 +112,30 @@
 ##
 ## string settings
 ##
-#attributetype ( 1.3.6.1.4.1.7165.2.1.10 NAME 'homeDrive'
-#DESC 'NT homeDrive'
-#EQUALITY caseIgnoreIA5Match
-#SYNTAX 1.3.6.1.4.1.1466.115.121.1.26{4} SINGLE-VALUE )
-
-#attributetype ( 1.3.6.1.4.1.7165.2.1.11 NAME 'scriptPath'
-#DESC 'NT scriptPath'
-#EQUALITY caseIgnoreIA5Match
-#SYNTAX 1.3.6.1.4.1.1466.115.121.1.26{255} SINGLE-VALUE )
-
-#attributetype ( 1.3.6.1.4.1.7165.2.1.12 NAME 'profilePath'
-#DESC 'NT profilePath'
-#EQUALITY caseIgnoreIA5Match
-#SYNTAX 1.3.6.1.4.1.1466.115.121.1.26{255} SINGLE-VALUE )
-
-#attributetype ( 1.3.6.1.4.1.7165.2.1.13 NAME 'userWorkstations'
-#DESC 'userWorkstations'
-#EQUALITY caseIgnoreIA5Match
-#SYNTAX 1.3.6.1.4.1.1466.115.121.1.26{255} SINGLE-VALUE )
-
-#attributetype ( 1.3.6.1.4.1.7165.2.1.17 NAME 'smbHome'
-#DESC 'smbHome'
-#EQUALITY caseIgnoreIA5Match
-#SYNTAX 1.3.6.1.4.1.1466.115.121.1.26{128} )
+attributetype ( 1.3.6.1.4.1.7165.2.1.10 NAME 'homeDrive'
+DESC 'NT homeDrive'
+EQUALITY caseIgnoreIA5Match
+SYNTAX 1.3.6.1.4.1.1466.115.121.1.26{4} SINGLE-VALUE )
+
+attributetype ( 1.3.6.1.4.1.7165.2.1.11 NAME 'scriptPath'
+DESC 'NT scriptPath'
+EQUALITY caseIgnoreIA5Match
+SYNTAX 1.3.6.1.4.1.1466.115.121.1.26{255} SINGLE-VALUE )
+
+attributetype ( 1.3.6.1.4.1.7165.2.1.12 NAME 'profilePath'
+DESC 'NT profilePath'
+EQUALITY caseIgnoreIA5Match
+SYNTAX 1.3.6.1.4.1.1466.115.121.1.26{255} SINGLE-VALUE )
+
+attributetype ( 1.3.6.1.4.1.7165.2.1.13 NAME 'userWorkstations'
+DESC 'userWorkstations'
+EQUALITY caseIgnoreIA5Match
+SYNTAX 1.3.6.1.4.1.1466.115.121.1.26{255} SINGLE-VALUE )
+
+attributetype ( 1.3.6.1.4.1.7165.2.1.17 NAME 'smbHome'
+DESC 'smbHome'
+EQUALITY caseIgnoreIA5Match
+SYNTAX 1.3.6.1.4.1.1466.115.121.1.26{128} )

 #attributetype ( 1.3.6.1.4.1.7165.2.1.18 NAME 'domain'
 #DESC 'Windows NT domain to which the user belongs'
@@ -145,15 +145,15 @@
 ##
 ## user and group RID
 ##
-#attributetype ( 1.3.6.1.4.1.7165.2.1.14 NAME 'rid'
-#DESC 'NT rid'
-#EQUALITY integerMatch
-#SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 SINGLE-VALUE )
-
-#attributetype ( 1.3.6.1.4.1.7165.2.1.15 NAME 'primaryGroupID'
-#DESC 'NT Group RID'
-#EQUALITY integerMatch
-#SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 SINGLE-VALUE )
+attributetype ( 1.3.6.1.4.1.7165.2.1.14 NAME 'rid'
+DESC 'NT rid'
+EQUALITY integerMatch
+SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 SINGLE-VALUE )
+
+attributetype ( 1.3.6.1.4.1.7165.2.1.15 NAME 'primaryGroupID'
+DESC 'NT Group RID'
+EQUALITY integerMatch
+SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 SINGLE-VALUE )

 ##
 ## The smbPasswordEntry objectclass has been depreciated in favor of the
```

These changes were to reenable several attributes that had been available in earlier Samba versions, and are currently viewed as historical. These attributes are required by the Apple schema's objectclasses.

After downloading the collection of schema to your Linux host, move it to `/etc/openldap/`. After this, remove the existing schema directory on your machine, then unpack them from the tar.bz2 file, like so:

```shell
cd /etc/openldap
rm -rf schema
tar xvf ldap_schema.tar.bz2
```

Now that the schema are in place, let's start updating our `/etc/openldap/slapd.conf` to allow them to be used. Open the file with your favourite editor and register the schema files in it by modifying the schema include section, like so:

```
include /etc/openldap/schema/core.schema
include /etc/openldap/schema/cosine.schema
include /etc/openldap/schema/nis.schema
include /etc/openldap/schema/inetorgperson.schema
include /etc/openldap/schema/fmserver.schema
include /etc/openldap/schema/openldap.schema
include /etc/openldap/schema/apple_auxillary.schema
include /etc/openldap/schema/samba3.schema
include /etc/openldap/schema/apple.schema
include /etc/openldap/schema/sudo.schema
include /etc/openldap/schema/openssh-lpk.schema
include /etc/openldap/schema/yast.schema
```

**NOTE:** The order of schema is VERY important, as some are dependent on earlier ones, so please be mindful of the entries that you add!

### Setting the DB Type

OpenLDAP has a number of storage backends, from flat files to Berkeley DB to shell or Perl interpreter backends. These allow it to be very flexible. In our case, we're using the MDB backend, which is the current recommended default by the OpenLDAP upstream developers. To enable it, in the module path section, load it's backend:

```
# Load backend modules such as databas engines
modulepath /usr/lib64/openldap
moduleload back_mdb.la
```

### SASL Mappings

By default, OpenLDAP uses the Distinguished Name of a person to find them in the database, however, when using SASL mechanisms, such as Kerberos via GSSAPI, the request does not come in the same way that OpenLDAP expects it. To map this, we use the `authz-regexp` directive to map users and computers to the appropriate place in the LDAP tree. To make this work, add the following lines to your slapd.conf:

```
# authz replace to allow mapping from SASL mechs to LDAP users
authz-regexp uid=host/([^,]*),cn=.*,cn=gssapi,cn=auth "uid=$1,cn=computers,dc=tolharadys,dc=net"
authz-regexp uid=([^/]*)(/[^,]*|),cn=.*,cn=.*,cn=auth "uid=$1,cn=users,dc=tolharadys,dc=net"
authz-regexp uid=([^/]*)(/[^,]*|),cn=.*,cn=auth "uid=$1,cn=users,dc=tolharadys,dc=net"
```

### Access Controls

The next section is critical in securing your LDAP installation. The access controls to the data in your LDAP should be carefully considered, as certain attributes and Distinguished Names should be controlled both for read and write. The following Access Controls match up with the ones used in macOS Server, which should give reasonable security over the data.

In later updates, we may review these entries and expand them to protect new fields as we expand the LDAP to incorporate more services in our network.

To control access, we need to add `access` directives in `/etc/openldap/slapd.conf` with the appropriate rights associated with whom can do specific actions. To protect your Linux Open Directory data, add the following lines to slapd.conf:

```
# Very important: define ACL to authorise client access
access to attrs=userPassword,shadowLastChange,userPKCS12
        by dn="cn=admin,dc=tolharadys,dc=net" write
        by dn="uid=diradmin,cn=users,dc=tolharadys,dc=net" write
        by anonymous auth
        by self write
        by * none
        
access to dn.base=""
        by dn="cn=admin,dc=tolharadys,dc=net" write
        by dn="uid=diradmin,cn=users,dc=tolharadys,dc=net" write
        by * read
        
access to *
        by dn="cn=admin,dc=tolharadys,dc=net" write
        by dn="uid=diradmin,cn=users,dc=tolharadys,dc=net" write
        by * read
        
access to dn.base="cn=Subschema"
        by * read
```

These default Access Controls allow the rootdn (cn=admin) and the Directory Admin (uid=diradmin) write to the userPassword, shadowLastChange, and userPCKS12 attributes, plus write to all sections of the LDAP tree, except the cn=Subschema section of the tree. Additionally, any user may modify the userPassword, shadowLastChange, and userPCKS12 attributes of their own record. Anyone else only has read-only access.

Eventually, we might put stronger controls around the attributes for SSH keys, user certificates (other than userPCKS12), and allow other attributes to be modified by their respective owners.

#### Defining the LDAP Database

The next part in the `/etc/openldap/slapd.conf` file we'll be looking at for now is defining the actual LDAP database that will hold the data that will be presented when an LDAP query is sent to the server.

As the OpenLDAP project is moving to use the MDB database type as the default, we'll set that as the database type. Next, we'll set the root suffix, which sets our base distinguished name for the LDAP, which is typically your DNS domain. After this, we'll define our LDAP's rootdn, which is the core administrative user to the LDAP tree. As the administrative distinguished name is sensitive, it needs a secure password defined, which should be encrypted while in the local configuration file on the disk, and inside the LDAP tree itself to protect against malicious users that somehow manage to breach the system from gaining accesses they shouldn't have. Additionally, we will set the directory where the database should reside. Finally, adding some indexes to the database to allow for rapid queries of the data.

To set these settings, add the following into your `/etc/openldap/slapd.conf`:

```
# Define a LDAP database
database     mdb
suffix       "dc=tolharadys,dc=net"
rootdn       "cn=admin,dc=tolharadys,dc=net"
# Please avoid using clear text for root password
# See slappasswd(8) for instructions on creating a salted+hashed password
#
# NOTE: the entry below is NOT valid. Please set this using the slapppasswd
# tool.
rootpw       {SSHA}xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
# The database directory must exist prior to the start of OpenLDAP daemon
# The directory should be owned by ldap user and permission 0700 is recommended
directory    /var/lib/ldap
# Indices to maintain
index        objectClass eq
```

### Allowing OpenLDAP to Use Krb5 for Authorisation

To allow slapd to be aware of the krb5.keytab we installed into `/etc/openldap`, we need to export an environment variable that lists where the keytab file is located during it's service startup. On openSUSE, this is set in the `/etc/sysconfig/openldap` file. To set this, change the OPENLDAP_KRB5_KEYTAB variable in the file like so:

```
OPENLDAP_KRB5_KEYTAB="FILE:/etc/openldap/krb5.keytab"
```

### Testing

To validate that OpenLDAP is configured to allow LDAP services to work, ensure that `slapd` is not running, and then run the following command:

```shell
slapd -d 1 -g openldap -u openldap -F /etc/ldap/slapd.d/
```

If no errors appear, it should start correctly and wait dilligently for queries to service. From another terminal, run`kinit diradmin/admin` to authenticate to the kerberos service and then run `ldapsearch` to finally test that queries are being serviced correctly. If you see errors like the following:

```
SASL/GSSAPI authentication started
ldap_sasl_interactive_bind_s: Other (e.g., implementation specific) error (80)
```

This can indicate problems with either read access to the LDAP krb5.keytab, or clock skew that should have been addressed by the NTP client configuration earlier. Once you've resolved any issues, stop the service running on the other terminal, then set `slapd` to start at boot, and start the service:

```shell
systemctl enable slapd.service
systemctl start slapd.service
systemctl status slapd.service
```

The output from the status should look something like this:

```shell
● slapd.service - OpenLDAP Server Daemon
   Loaded: loaded (/usr/lib/systemd/system/slapd.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2018-07-25 22:49:51 EDT; 28min ago
  Process: 16265 ExecStart=/usr/lib/openldap/start (code=exited, status=0/SUCCESS)
 Main PID: 16353 (slapd)
    Tasks: 4 (limit: 4915)
   CGroup: /system.slice/slapd.service
           └─16353 /usr/sbin/slapd -h ldap:///   -f /etc/openldap/slapd.conf -u ldap -g ldap -o slp=on
```

---

## Navigation:

* [Home](https://greeneg.github.io)
* [Previous Step => Setting up MIT Kerberos v5](setup_mit_krb5.md)
* [Next Step => Populating Your Directory](populate_ldap.md)
