# :My Personal Site:

This site is my personal collection of howtos and ruminations. Anything I post here is either my opinion, or based off experiences building out opsn source tooling for personal infrastructure projects.

## ::Goodbye macOS Server, You Served Me Well::

**THIS ARTICLE IS A WORK IN PROGRESS! There are things that will change as I work out issues.**

Over the last several years, I've used a Mac Mini running OS X and OS X Server to manage the identity information for my personal network. Unfortunately, Apple has decided not to allow upgrades for the model of Mini that I'm using. To keep my server from having security and other nasty issues, I've decided that while it was easy to maintain, it's time for the machine to no longer run OS X, and by extension OS X Server.

This left me in a bit of a cunnundrum, since my home network has a few Macs and a Windows box, plus several Linux systems, all of which need identity data, and other services that that machine ran. After thinking it through, I decided to move that Mini to use openSUSE Leap 15, and use the array of open source services on the machine to fill the gap that used to be managed by OS X Server.

As this process is a pretty lengthy process, I've opted to split it into a few different pages.

 * [Step 0: Prep the Mac Mini for Linux](prep_mac_mini_for_linux.md)
 * [Step 1: Install openSUSE Leap 15](install_opensuse_leap_15.md)
 * [Step 2: Setting up an NTP Service for Your Network](setup_ntp.md)
 * [Step 3: Setting up MIT Kerberos](setup_mit_krb5.md)
 * [Step 4: Setting up OpenLDAP](setup_openldap.md)
