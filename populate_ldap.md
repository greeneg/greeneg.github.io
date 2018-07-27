# ::Goodbye MacOS Server, You Served Me Well::

## Step 5: Populating Your Directory

The next step in the build of a Linux-based Open Directory is to populate the directory tree with data.

### Creating the Directory Administrator Kerberos principle

Before we can continue, we need to create an administrative principle in the Kerberos datastore to administrate the directory. By default, Open Directory's normal administrative user is the Directory Administrator, diradmin. To create the kerberos principle for this account, run the following command:

```shell
kadmin.local -q "addprinc diradmin/admin"
```

### Base LDAP Structure

At this point, you have a functioning, albet empty, directory service. To make use of it, we need to first create a basic hierarchy for your computers, groups, users, and various other objects to be stored. To do this, create a file using your favourite text editor that contains the data that follows, with the slight difference that the `dc=tolharadys,dc=net` should be replaced with an appropriate stanza matching your DNS domain:

```ldif
dn: dc=tolharadys,dc=net
objectClass: top
objectClass: domain
dc: tolharadys
description: The tolharadys.net LDAP tree

dn: cn=users,dc=tolharadys,dc=net
cn: users
objectClass: container

dn: cn=groups,dc=tolharadys,dc=net
cn: groups
objectClass: container

dn: cn=mounts,dc=tolharadys,dc=net
cn: mounts
objectClass: container

dn: cn=accesscontrols,dc=tolharadys,dc=net
cn: accesscontrols
objectClass: container

dn: cn=certificateauthorities,dc=tolharadys,dc=net
cn: certificateauthorities
objectClass: container

dn: cn=computers,dc=tolharadys,dc=net
cn: computers
objectClass: container

dn: cn=computer_groups,dc=tolharadys,dc=net
cn: computer_groups
objectClass: container

dn: cn=computer_lists,dc=tolharadys,dc=net
cn: computer_lists
objectClass: container

dn: cn=config,dc=tolharadys,dc=net
cn: config
objectClass: container

dn: cn=locations,dc=tolharadys,dc=net
cn: locations
objectClass: container

dn: cn=machines,dc=tolharadys,dc=net
cn: machines
objectClass: container

dn: cn=neighborhoods,dc=tolharadys,dc=net
cn: neighborhoods
objectClass: container

dn: cn=people,dc=tolharadys,dc=net
cn: people
objectClass: container

dn: cn=presets_computer_lists,dc=tolharadys,dc=net
cn: presets_computer_lists
objectClass: container

dn: cn=presets_groups,dc=tolharadys,dc=net
cn: presets_groups
objectClass: container

dn: cn=presets_users,dc=tolharadys,dc=net
cn: presets_users
objectClass: container

dn: cn=printers,dc=tolharadys,dc=net
cn: printers
objectClass: container

dn: cn=augments,dc=tolharadys,dc=net
cn: augments
objectClass: container

dn: cn=autoserversetup,dc=tolharadys,dc=net
cn: autoserversetup
objectClass: container

dn: cn=filemakerservers,dc=tolharadys,dc=net
cn: filemakerservers
objectClass: container

dn: cn=resources,dc=tolharadys,dc=net
cn: resources
objectClass: container

dn: cn=places,dc=tolharadys,dc=net
cn: places
objectClass: container

dn: cn=maps,dc=tolharadys,dc=net
cn: maps
objectClass: container

dn: cn=presets_computers,dc=tolharadys,dc=net
cn: presets_computers
objectClass: container

dn: cn=presets_computer_groups,dc=tolharadys,dc=net
cn: presets_computer_groups
objectClass: container

dn: cn=automountMap,dc=tolharadys,dc=net
cn: automountMap
objectClass: container

dn: ou=macosxodconfig,cn=config,dc=tolharadys,dc=net
ou: macosxodconfig
objectClass: top
objectClass: organizationalUnit

dn: cn=CollabServices,cn=config,dc=tolharadys,dc=net
cn: CollabServices
objectClass: apple-configuration
objectClass: top
```

When you're happy with the file, save it as `ldap_structure.ldif` and securely transfer the file to the server. Then open a terminal on your server, login as root, then run the following commands to add these records to your domain:

```shell
kinit diradmin/admin
ldapvi --bind sasl --add --in ~/ldap_structure.ldif
```

To exit `ldapvi`, press the `:` character, then type `q`. Ldapvi will then request if it should save the changes, or close without importing the LDAP information.

---

## Navigation:

* [Home](https://greeneg.github.io)
* [Previous Step => Setting up OpenLDAP](setup_openldap.md)
