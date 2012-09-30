---
layout: post
title: "Managing dynamic ip sets in firewall rules with iptables and ipset"
date: 2012-09-24 15:23
comments: true
categories: [Linux, iptables]
---

There was a need to configure particular server to apply a set of iptables rules to dynamically comprised set of ip addresses. Trying to solve this problem using only plain old iptables rules can quickly become a mess, especially considering that large number of rules drastically diminishes performance of machine in general, set aside becoming a pain adding/removing entries from the list. Fortunately, there is a tool out there allowing us to solve this problem in a neat and tidy way, and it is called [ipset](http://ipset.netfilter.org/).
<!--more-->

## Problem statement
We need server to have __reboot persistent__ configuration of ipset and iptables rules, allowing us to easily manage list of whitelisted IPs eligible to access certain services (i.e. TCP connections on particular ports). Particularly, we should tune iptables `nf_conntrack_ftp` module to facilitate passive FTP connections on non-standart port (e.g. 2121).

In short, allow TCP connections on ports 22, 80, 139, 445, 2121 and UDP packets on port 53.

## Installation and initial setup
First, make sure ipset is installed, otherwise install it using
```bash
$ sudo apt-get install ipset
```

We are going to use `hash:ip` set type to store list of IP addresses for whitelisting. The advantage of this set type is that the amount of memory required to store entries dynamically expands and shrinks whether entries are added to or removed from list. `bitmap:ip` set type in contrary allocates memory for `maxelem` entries(which cannot exceed 65536, by the way) specified upon creation of set and thus we cannot add any entry if that amount is exceeded. For some use-cases this may just not be an issue, but you should be aware of this caveat.

```bash creating ip set
$ sudo ipset create whitelisted hash:ip family inet hashsize 64 maxelem 4294967295
```

Afterwards we can manage contents of created set from anywhere we find fit:
```bash
#adding ip to set
$ sudo ipset add whitelisted 10.0.0.1
#removing ip from set
$ sudo ipset del whitelisted 10.0.0.1
#or even deleting the whole set at once
$ sudo ipset destroy whitelisted
```

Configuring iptables around our set and according to initial requirements would look like this:
```bash
$ sudo iptables-restore <<-EOF
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
:whitelisted - [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -m set --match-set whitelist src -j whitelisted
-A whitelisted -p tcp -m multiport --dports 22,80,139,445,2121 -j ACCEPT
-A whitelisted -p udp --dport 53 -j ACCEPT
COMMIT
EOF
```
Here we set default policies for INPUT and FORWARD chains to DROP, then some basic setup for accepting related packets for established connections and loopback interface. Then packets with source addresses matching those in __whitelisted__ ip set go into whitelisted chain, where we've defined required rules. As simple as that.

In order to allow passive ftp data connections through firewall, we need to load nf_conntrack_ftp kernel module on boot. By default, it monitors control ftp connections on port 21 detecting advertisements of data connection ports and dynamically allowing them in. Given we are using custom port 2121, we should add following line to `/etc/modules`
```
nf_conntrack_ftp ports=21,2121
```

That concludes initial configuration, let's proceed with persisting it.

## Reboot persistence

In Ubuntu there is a great evented-based init service geared up for our cause called [Upstart](http://upstart.ubuntu.com/). All we need is to create a few simple tasks to run on certain events on system.

First, we need some place safe to store iptables and ipset configurations. I prefer `/etc/iptables` folder, but it's up to you to choose, really. So, we create this folder with `sudo mkdir /etc/iptables` and make sure it's only writable for root user. Then we create task for saving configurations there.
```bash /etc/init/iptables_save.conf
description     "Save iptables rules on shutdown"
start on runlevel [016]
task
script
  ipset save > /etc/iptables/ipset.save
  iptables-save > /etc/iptables/iptables.save
end script
```

Here we say this task is to run whether server is rebooting or shutting down and it's simply exporting settings to corresponding files.

The following tasks are designed to restore this settings back up on startup of a server.
```bash /etc/init/iptables_restore.conf
description     "Apply iptables rules on startup"
start on startup and starting networking
task
script
  iptables-restore < /etc/iptables/iptables.save
end script
```

```bash /etc/init/ipset_restore.conf
description    "Apply ipset rules un startup"
start on startup and starting iptables_restore
task
script
  ipset restore < /etc/iptables/ipset.save
end script
```

Important caveat here is that we start `ipset_restore`  on `starting iptables_restore` meaning _right before_ it. Restoring iptables earlier would fail because of referencing rule with ipset set not yet created/imported.

Finally, we should run `sudo initctl reload-configuration` for upstart to detect our changes and configure itself.
