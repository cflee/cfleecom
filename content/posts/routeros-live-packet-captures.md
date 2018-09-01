---
title: "Live packet captures using MikroTik RouterOS and Wireshark"
date: 2015-07-26
draft: false
aliases:
  - /post/live-packet-captures-using-mikrotik-routeros-and-wireshark
---

This is a quick post to let Google pick it up. You know how the MikroTik wiki and forums are, plus how Stack Overflow is.. very confusing.

This was tested on RouterOS v6.27 (mipsbe) and v6.28 (smips), but it should work mostly the same everywhere. I was using Windows 7 (64-bit) and Wireshark 1.12.6, on a Thinkpad X220 using the onboard gigabit ethernet port and Intel 6205 802.11n card.

## How do I get it working?
**First,** launch Wireshark, and start a capture on the interface that's connected to the MikroTik box. This may be a wired or wireless interface. (Promiscuous/monitor mode is not necessary, everything works fine even on Windows 7. I've also tested this setup over wifi to an AP that is then connected to the MikroTik box.)

You should set a capture filter of 'udp port 37008' to only capture the sniffer traffic, excluding traffic directed or originating from your monitoring machine. You should also be able to use 'udp dst port 37008' instead, if you like. Firewalls should also be configured appropriately to allow this traffic.

According to one site (Bravi.org, link below), you should also disable WCCP protocol interpretation in Wireshark. I'm not sure if this makes a difference, I have yet to test re-enabling it.

**Next,** configure the sniffer tool on the MikroTik box.

```
/tool sniffer
set streaming-enabled=yes
set streaming-server=192.168.88.2
set filter-stream=yes
set filter-interface=bridge1
The filter parts are optional, but I find it very helpful to only observe one of the interfaces. Replace the IP address as appropriate, of course.
```

**Finally,** start the sniffer.

```
/tool sniffer start
```

That's it! You should start to see frames streaming in on the Wireshark display. Don't forget to stop the sniffer later, otherwise it will keep running until rebooted.

## Why these particular instructions?
It turns out that the MikroTik router sends UDP datagrams to port 37008 on the streaming-server IP address, containing a TZSP packet. Each packet encapsulates a single Ethernet frame. If all goes well, Wireshark transparently unwraps it and displays it as whatever's inside, i.e. the Ethernet frame, IP packet, UDP datagram, and DNS query. You can see the TZSP encapsulation in the packet details.

## Tips and tricks
The 'tzsp' display filter is useful if you are getting incomplete IP packet errors.

If you want to exclude an IP address (ip.addr) or MAC address (eth.addr), 'ip.addr != 192.168.88.100' will yield unexpected results (i.e. no effect). This is because, for instance, ip.addr refers to source or destination address. Therefore, that expression is translated into 'ip.src != 192.168.88.100 or ip.dst != 192.168.88.100', which is probably not what you want. You can fix this by NOT-ing the whole thing, or going '!(ip.addr == 192.168.88.100)' instead.

## References
Bravi.org ["Stream Mikrotik RouterOS Sniffer TZSP directly to a remote WireShark host"](https://blog.bravi.org/?p=768) ([Mirror by WebCite](http://www.webcitation.org/6aI16Xs59))

MikroTik Wiki ["Manual:Tools/Packet Sniffer"](http://wiki.mikrotik.com/index.php?title=Manual:Tools/Packet_Sniffer&oldid=23105) ([Mirror by WebCite](http://www.webcitation.org/6aI1Vkzih))
