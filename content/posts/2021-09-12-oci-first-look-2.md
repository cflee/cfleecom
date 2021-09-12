+++
title = "Oracle Cloud: first impressions (II)"
+++

This continues my first impression snippets as I poke around my "Always Free" OCI Ampere A1 instances.

## Setting up compute instances

I was originally going to review the Oracle Linux Cloud Developer 8 image, but all the RHEL/CentOS/Oracle Linux differences from Ubuntu/Debian-based OS would distract from the OCI aspect, so I spun up another instance using Ubuntu Linux 20.04.

Note: In order to use Ubuntu Linux with Ampere A1 instances (aarch64), you need to select the non-minimal image when creating the instance.

### Instance firewall gotchas

To open up inbound traffic to a specific port as part of bringing up a service on your host, you need to sort out the VCN-level stuff (subnet's security list and/or network security group's ingress rules), as well as the OS firewall rules.

**Why do we need to specifically configure the OS firewall?**
Unlike a default Ubuntu setup where traffic is unrestricted by default, OCI configures a bunch of rules on iptables, including an default deny on inbound traffic. 
It's done directly on iptables without going through ufw (ufw remains disabled by default if you look at `ufw status`).
You can see the inbound rules with `iptables -L INPUT` or by looking at `/etc/iptables/rules.v4`:

```
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -p udp --sport 123 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited
```

(On Oracle Linux they nominally configure these rules through firewalld, but `/etc/firewalld/direct.xml` is sorta kinda passing through raw iptables commands anyway...)

This doesn't look too unusual, perhaps just rather prudent to also restrict inbound traffic.

**Why aren't these rules configured through UFW?**
In fact, you're specifically advised not to enable or use UFW, because OCI docs describe a [known issue](https://docs.oracle.com/en-us/iaas/Content/knownissues.htm#ufw) that enabling UFW will prevent the instance from booting, and the stated workaround is to... not use UFW?!

If we keep looking at the configured iptables rules, we see that OCI also adds outbound rules to restrict access to `169.254.0.0/16`, which is fully reserved by OCI:

```
-A OUTPUT -d 169.254.0.0/16 -j InstanceServices
-A InstanceServices -d 169.254.0.2/32 -p tcp -m owner --uid-owner 0 -m tcp --dport 3260 -m comment --comment "See the Oracle-Provided Images section in the Oracle Cloud Infrastructure documentation for security impact of modifying or removing this rule" -j ACCEPT
-A InstanceServices -d 169.254.2.0/24 -p tcp -m owner --uid-owner 0 -m tcp --dport 3260 -m comment --comment "See the Oracle-Provided Images section in the Oracle Cloud Infrastructure documentation for security impact of modifying or removing this rule" -j ACCEPT
-A InstanceServices -d 169.254.4.0/24 -p tcp -m owner --uid-owner 0 -m tcp --dport 3260 -m comment --comment "See the Oracle-Provided Images section in the Oracle Cloud Infrastructure documentation for security impact of modifying or removing this rule" -j ACCEPT
-A InstanceServices -d 169.254.5.0/24 -p tcp -m owner --uid-owner 0 -m tcp --dport 3260 -m comment --comment "See the Oracle-Provided Images section in the Oracle Cloud Infrastructure documentation for security impact of modifying or removing this rule" -j ACCEPT
-A InstanceServices -d 169.254.0.2/32 -p tcp -m tcp --dport 80 -m comment --comment "See the Oracle-Provided Images section in the Oracle Cloud Infrastructure documentation for security impact of modifying or removing this rule" -j ACCEPT
-A InstanceServices -d 169.254.169.254/32 -p udp -m udp --dport 53 -m comment --comment "See the Oracle-Provided Images section in the Oracle Cloud Infrastructure documentation for security impact of modifying or removing this rule" -j ACCEPT
-A InstanceServices -d 169.254.169.254/32 -p tcp -m tcp --dport 53 -m comment --comment "See the Oracle-Provided Images section in the Oracle Cloud Infrastructure documentation for security impact of modifying or removing this rule" -j ACCEPT
-A InstanceServices -d 169.254.0.3/32 -p tcp -m owner --uid-owner 0 -m tcp --dport 80 -m comment --comment "See the Oracle-Provided Images section in the Oracle Cloud Infrastructure documentation for security impact of modifying or removing this rule" -j ACCEPT
-A InstanceServices -d 169.254.0.4/32 -p tcp -m tcp --dport 80 -m comment --comment "See the Oracle-Provided Images section in the Oracle Cloud Infrastructure documentation for security impact of modifying or removing this rule" -j ACCEPT
-A InstanceServices -d 169.254.169.254/32 -p tcp -m tcp --dport 80 -m comment --comment "See the Oracle-Provided Images section in the Oracle Cloud Infrastructure documentation for security impact of modifying or removing this rule" -j ACCEPT
-A InstanceServices -d 169.254.169.254/32 -p udp -m udp --dport 67 -m comment --comment "See the Oracle-Provided Images section in the Oracle Cloud Infrastructure documentation for security impact of modifying or removing this rule" -j ACCEPT
-A InstanceServices -d 169.254.169.254/32 -p udp -m udp --dport 69 -m comment --comment "See the Oracle-Provided Images section in the Oracle Cloud Infrastructure documentation for security impact of modifying or removing this rule" -j ACCEPT
-A InstanceServices -d 169.254.169.254/32 -p udp --dport 123 -m comment --comment "See the Oracle-Provided Images section in the Oracle Cloud Infrastructure documentation for security impact of modifying or removing this rule" -j ACCEPT
-A InstanceServices -d 169.254.0.0/16 -p tcp -m tcp -m comment --comment "See the Oracle-Provided Images section in the Oracle Cloud Infrastructure documentation for security impact of modifying or removing this rule" -j REJECT --reject-with tcp-reset
-A InstanceServices -d 169.254.0.0/16 -p udp -m udp -m comment --comment "See the Oracle-Provided Images section in the Oracle Cloud Infrastructure documentation for security impact of modifying or removing this rule" -j REJECT --reject-with icmp-port-unreachable
```

We could summarise it as such, with an implicit deny for all other outbound access:

| Destination network | Destination proto/port | Restrictions | Usage |
|---|---|---|---|
| `169.254.0.2/32`<br>`169.254.2.0/24`<br>`169.254.4.0/24`<br>`169.254.5.0/24` | tcp/3260 | uid-owner 0 | iSCSI |
| `169.254.0.2/32` | tcp/80 | | ? |
| `169.254.0.3/32` | tcp/80 | uid-owner 0 | ? |
| `169.254.0.4/32` | tcp/80 | | ? |
| `169.254.169.254/32` | udp/53, tcp/53 | | DNS |
| `169.254.169.254/32` | udp/67 | | DHCP |
| `169.254.169.254/32` | udp/69 | | TFTP |
| `169.254.169.254/32` | tcp/80 | | [Instance metadata](https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/gettingmetadata.htm) |
| `169.254.169.254/32` | udp/123 | | NTP |

`169.254.169.254` is visibly actively in use by the instance's DNS, DHCP, NTP clients and instance metadata clients (e.g. `oracle-cloud-agent` running in snap).
`169.254.0.3` and `169.254.0.4` respond to a HTTP GET request with a 404, while connection is refused by `169.254.0.2`.

What stands out here are the rules with the `uid-owner 0` restriction, that only allow "a process with the given effective user id" to contact those endpoints.
Currently the [essential firewall rules](https://docs.oracle.com/en-us/iaas/Content/Compute/References/bestpracticescompute.htm#Essentia) docs only mention `169.254.0.2/32` and `169.254.2.0/24`, but there are four CIDRs listed with destination port tcp/3260 for iSCSI, and another endpoint that has the http port restricted as well.

**Why is iSCSI access required?**
The docs state that it's used to serve the boot and block volumes, but... really?
That is rather unexpected.
Looking at `mount` and `iscsiadm -m session`, the boot volume looks like it's "just" presented as `/dev/sda`, nothing fancy, and no iSCSI involved.

```
$ lsblk /dev/sda
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda       8:0    0 46.6G  0 disk
├─sda1    8:1    0 46.5G  0 part /
└─sda15   8:15   0   99M  0 part /boot/efi
```

I wondered for a while whether this was some deliberate choice to provide more performance for some specific Oracle software product that prefers guest OS iSCSI instead of going through a virtual device interface to the hypervisor.
In that case, it would make sense that perhaps regular-looking block devices had come first, and then the direct iSCSI path had been retrofitted on later to provide higher performance.

It turns out that Block Volumes have [two attachment types](https://docs.oracle.com/en-us/iaas/Content/Block/Concepts/overview.htm#attachtype), iSCSI or paravirtualized, and iSCSI attachments came first and were the only choice prior to Mar 2018 when OCI [announced paravirtualized attachments](https://blogs.oracle.com/cloud-infrastructure/post/press-the-easy-button-paravirtualized-block-volume-attachments-for-vms).
(OCI in its current incarnation launched in Oct 2016 with a single region offering "Bare Metal Cloud Services".)

Given that iSCSI attachments were the default, it's probably difficult to remove iSCSI support from OS images moving forward, especially since the iSCSI attachments are not being deprecated nor discouraged. 
Therefore, it looks like the presence of iSCSI volume attachments and the need for these uid restrictions on outbound traffic to the iSCSI network endpoints will continue to persist.
If a specific instance is known to not require iSCSI volume attachments, it would be possible to just block outbound traffic to those CIDRs on entirely, while preserving the critical traffic to `169.254.169.254`.

Since the need to accommodate iSCSI volume attachments is unavoidable in general, perhaps it's worth looking into whether those rules can be safely fitted into UFW, to allow users to `ufw enable` without breaking the iSCSI connections.

**What did I want to do in the first place before all this yak-shaving?**
Oh, open up some udp ports for [mosh](https://mosh.org/):

```
# insert into /etc/iptables/rules.v4 before the -A INPUT -j REJECT line
-A INPUT -p udp --dport 60000:60100 -j ACCEPT

# overwrite all current rules
# sudo iptables-restore < /etc/iptables/rules.v4
```

There, now the first three things that I install on a new VM are up: Tailscale, mosh and tmux.

### Software repos

apt is configured to use `http://ap-melbourne-1-ad-1.clouds.ports.ubuntu.com/ubuntu-ports/`, which by the name is something within the region and specific Availability Domain (~ Availability Zone in AWS).

Nope - running a traceroute shows that it exits into Melbourne, then goes to Sydney to San Jose to NYC to Boston, and in the end `banjo.canonical.com` is 230ms ping.
Unfortunately it seems like arm64 architecture (being part of [`ubuntu-ports`](http://ports.ubuntu.com/ubuntu-ports/dists/focal/)) is simply not mirrored much, at least the larger Ubuntu mirrors in Australia all don't have it, so there may not be all that much that we can do about this.
