---
title: "MikroTik RouterOS on tagged M1 Fibre"
date: 2016-06-08
draft: false
---

If you have a recently-connected M1 fibre connection, you should be using the black Huawei ONT, with optional voice port for digital voice, and an untagged internet port on port 1. This guide is not for you.

If you're on an older M1 fibre connection on the white Nucleus Connect ONT, with a tagged port, typically issued with a white Huawei Residential Gateway or something like the Asus RT-N56U, and you want to connect a MikroTik RouterOS device to the WAN – this guide is for you.

<hr>

On the tagged ports, internet access is on vlan id 1103 prio 1. Prio here is the CoS from 802.1p.

Most of the guidance online only discusses configuring VLANs, but not the "prio" component required. Some of the guidance will discuss using "/ip firewall mangle" to set the priority. I tried that half a dozen ways but never got it to work – all the DHCP-containing frames coming out of the port were stubbornly stuck at CoS=0.

Instead, you need to use "/interface bridge filter" to make it work.

Here are extracts from a config for RouterOS 6.33.5 on a RB941. We assume that the WAN port will be ether1.

```
# the following mac address can be set to your current
# router's mac address. or you can just switch off the
# ONT for 15 min and you should be able to connect with
# a new one.
/interface bridge
add admin-mac=1c:B7:2C:AB:CD:EF auto-mac=no name=br-wan
/interface bridge port
add bridge=br-wan interface=ether1-vlan1103
/interface vlan
add interface=ether1 name=ether1-vlan1103 vlan-id=1103
/interface bridge filter
add action=set-priority chain=output new-priority=1 \
    out-bridge=br-wan out-interface=ether1-vlan1103 \
    passthrough=no
add action=set-priority chain=forward new-priority=1 \
    out-bridge=br-wan out-interface=ether1-vlan1103 \
    passthrough=no
/interface ethernet switch vlan
add ports=switch1-cpu,ether1 switch=switch1 vlan-id=1103
/ip dhcp-client
add default-route-distance=0 disabled=no interface=br-wan \
    use-peer-ntp=no
/ip firewall nat
add action=masquerade chain=srcnat out-interface=br-wan
```

Of course, you should at the very least add your firewall rules to this before putting it online.
