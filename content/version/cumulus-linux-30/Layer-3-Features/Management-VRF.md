---
title: Management VRF
author: Cumulus Networks
weight: 151
aliases:
 - /display/CL30/Management+VRF
 - /pages/viewpage.action?pageId=5118405
pageID: 5118405
product: Cumulus Linux
version: 3.0.1
imgData: cumulus-linux-30
siteSlug: cumulus-linux-30
---
*Management VRF* - a subset of
[VRF](/version/cumulus-linux-30/Layer-3-Features/Virtual-Routing-and-Forwarding-VRF)
(virtual routing tables and forwarding) - provides a separation between
the out-of-band management network and the in-band data plane network.
For all VRFs, the *main* routing table is the default table for all of
the data plane switch ports. With management VRF, a second table,
*mgmt*, is used for routing through eth0.

Cumulus Linux only supports eth0 as the management interface. VLAN
subinterfaces, bonds, bridges and the front panel switch ports are not
supported as management interfaces.

When management VRF is enabled, logins to the switch are set into the
management VRF context. IPv4 and IPv6 networking applications run by an
administrator communicate out the management network by default. This
default context does not impact services run through `systemd` and the
`systemctl` command, and does not impact commands examining the state of
the switch; for example, using the `ip` command to list links, neighbors
or routes.

## <span id="src-5118405_ManagementVRF-enablevrf" class="confluence-anchor-link"></span>Enabling Management VRF</span>

To enable management VRF on eth0, complete the following steps:

1.  Open `/etc/network/interfaces` in a text editor.

2.  Configure the following, and save the file. Remember you must name
    the VRF *mgmt* to distinguish the mgmt VRF from a data plane VRF.
    
        auto mgmt
        iface mgmt
            address 127.0.0.1/8
            vrf-table auto
        
        auto eth0
        iface eth0 inet dhcp
            vrf mgmt

3.  Reboot the switch to activate the mgmt VRF:
    
        cumulus@switch:~$ sudo reboot

### Enabling NTP</span>

To enable NTP to run in the mgmt VRF:

1.  Configure the *mgmt* VRF in `/etc/networking/interfaces`. In this
    example, eth0 is assigned an IP address and default route via DHCP.
    
        auto mgmt
        iface mgmt
            address 127.0.0.1/8
            vrf-table auto
          
        auto eth0
        iface eth0 inet dhcp
            vrf mgmt

2.  Reboot the switch to activate the mgmt VRF.
    
        cumulus@switch:~$ sudo reboot

3.  By default, NTP is running in the default VRF. Stop NTP if it is
    currently running.
    
        cumulus@switch:~$ sudo systemctl stop ntp.service

4.  By default, NTP is configured to automatically start in the default
    VRF when the system boots. Disable NTP from automatically starting
    in the default VRF.
    
        cumulus@switch:~$ sudo systemctl disable ntp.service

5.  Start NTP in the mgmt VRF.
    
        cumulus@switch:~$ sudo systemctl start ntp@mgmt

6.  Verify that NTP peers are active.
    
    ``` 
    cumulus@switch:~$ ntpq -pn
         remote           refid      st t when poll reach   delay   offset  jitter
    ==============================================================================
    *38.229.71.1     204.9.54.119     2 u   42   64  377   31.275   -0.625   3.105
    -104.131.53.252  209.51.161.238   2 u   47   64  377   16.381   -5.251   0.681
    +45.79.10.228    200.98.196.212   2 u   44   64  377   42.998    0.115   0.585
    +74.207.240.206  127.67.113.92    2 u   43   64  377   73.240   -1.623   0.320    
    ```

7.  Enable ntp@mgmt so it starts when the switch boots:
    
        cumulus@switch:~$ sudo systemctl enable ntp@mgmt

### Enabling snmpd</span>

To enable `snmpd` to run in the mgmt VRF:

1.  Configure the mgmt VRF in `/etc/networking/interfaces`. In this
    example, eth0 is assigned an IP address and default route via DHCP.
    
        auto mgmt
        iface mgmt
            address 127.0.0.1/8
            vrf-table auto
        auto eth0
        iface eth0 inet dhcp
            vrf mgmt

2.  Reboot the switch to activate the mgmt VRF.
    
        cumulus@switch:~$ sudo reboot

3.  Stop `snmpd` is it is running.
    
        cumulus@switch:~$ sudo systemctl stop snmpd.service

4.  Make sure `snmpd` does not try to start in the default VRF if the
    system is rebooted.
    
        cumulus@switch:~$ sudo systemctl disable snmpd.service

5.  Start `snmpd` in the the mgmt VRF.
    
        cumulus@switch:~$ sudo systemctl start snmpd@mgmt

6.  Enable snmpd@mgmt so it starts when the switch boots:
    
        cumulus@switch:~$ sudo systemctl enable snmpd@mgmt

### Using ping or traceroute</span>

By default, issuing a `ping` or `traceroute` assumes the packet should
be sent to the dataplane network (the main routing table). If you wish
to use `ping` or `traceroute` on the management network, use the `-I`
flag for ping and `-i` for traceroute.

    cumulus@switch:~$ ping -I mgmt

or

    cumulus@switch:~$ sudo traceroute -i mgmt

## OSPF and BGP</span>

In general, no changes are required for either BGP or OSPF. Quagga was
updated in Cumulus Linux 3.0 to be VRF-aware and automatically sends
packets based on the switch port routing table. This includes BGP
peering via loopback interfaces. BGP does routing lookups in the default
table. However, one modification you may consider has to do with how
your routes get redistributed.

### Redistributing Routes in Management VRF</span>

Management VRF uses the mgmt table, including local routes. It does not
affect how the routes are redistributed when using routing protocols
such as OSPF and BGP.

To redistribute the routes in your network, use the `redistribute
connected` command under BGP or OSPF. This enables the directly
connected network out of eth0 to be advertised to its neighbor.

{{%notice note%}}

This also creates a route on the neighbor device to the management
network through the data plane, which may not be desired.

{{%/notice%}}

Cumulus Networks recommends you always use route maps to control the
advertised networks redistributed by the `redistribute connected`
command. For example, you can specify a route map to redistribute routes
in this way (for both BGP and OSPF):

    <routing protocol> 
    redistribute connected route-map redistribute-connected
    
    route-map redistribute-connected deny 100
     match interface eth0
    !
    route-map redistribute-connected permit 1000

## SNMP Traps Use eth0 Only</span>

SNMP cannot use a switch port to send data. For any SNMP traps, this
traffic gets sent out to eth0. Cumulus Networks plans to support switch
ports in the future.

{{%notice note%}}

For SNMP, this restriction only applies to traps. SNMP polling is not
affected.

{{%/notice%}}

## Using SSH within a Management VRF Context</span>

If you SSH to the switch through a switch port, it works as expected. If
you need to SSH from the device out a switch port, use `vrf exec default
ssh <ip_address_of_swp_port>`. For example:

    cumulus@switch:~$ sudo ip addr show swp17
    19: swp17: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 500
        link/ether ec:f4:bb:fc:19:23 brd ff:ff:ff:ff:ff:ff
        inet 10.23.23.2/24 scope global swp17
        inet6 fe80::eef4:bbff:fefc:1923/64 scope link
           valid_lft forever preferred_lft forever
    
    cumulus@switch:~$ sudo vrf exec default ssh 10.23.23.2 10.3.3.3

## Viewing the Routing Tables</span>

When you look at the routing table with `ip route show`, you are looking
at the switch port (*main*) table. You can also see the dataplane
routing table with `ip route show table main`.

To look at information about eth0 (the management routing table), use
`ip route show table mgmt`.

    cumulus@switch:~$ ip route show table mgmt
    default via 192.168.0.1 dev eth0
    
    cumulus@switch:~$ ip route show
    default via 10.23.23.3 dev swp17  proto zebra  metric 20
    10.3.3.3 via 10.23.23.3 dev swp17
    10.23.23.0/24 dev swp17  proto kernel  scope link  src 10.23.23.2
    192.168.0.0/24 dev eth0  proto kernel  scope link  src 192.168.0.11

### Viewing a Single Route</span>

Note that if you use `ip route get` to return information about a single
route, the command resolves over the *mgmt* table by default. To get
information about the route in the switching silicon, use:

    cumulus@switch:~$ ip route get <addr>

Or:

    cumulus@switch:~$ sudo cl-rctl ip route show <addr> 

To get the route for any VRF, use `ip route get <addr> oif <vrf name>`.
So to get the route for the mgmt VRF, run:

    cumulus@switch:~$ ip route get <addr> oif mgmt

## Using the mgmt Interface Class</span>

In `ifupdown2` [interface classes](/version/cumulus-linux-30/Configuring-and-Managing-Network-Interfaces/#ifupdown2-interface-classes)
are used to create a user-defined grouping for interfaces. The special
class *mgmt* is available to separate the switch's management interfaces
from the data interfaces. This allows you to manage the data interfaces
by default using `ifupdown2` commands. Performing operations on the
*mgmt* interfaces requires specifying the `--allow-mgmt` option, which
prevents inadvertent outages on the management interfaces. Cumulus Linux
by default brings up all interfaces in both the *auto* (default) class
and the *mgmt* interface class when the switch boots.

You configure the management interface in `/etc/network/interfaces`. In
the example below, the management interface, eth0, and the mgmt VRF
stanzas are added to the *mgmt* interface class:

    auto lo
    iface lo inet loopback 
      
    allow-mgmt eth0
    iface eth0 inet dhcp
        vrf mgmt
      
    allow-mgmt mgmt
    iface mgmt
        address 127.0.0.1/8
        vrf-table auto 

When you run `ifupdown2` commands against the interfaces in the mgmt
class, include --allow=mgmt with the commands. For example, to see which
interfaces are in the mgmt interface class, run:

    cumulus@switch:~$ ifquery l --allow=mgmt
    eth0
    mgmt 

To reload the configurations for interfaces in the mgmt class, run:

    cumulus@switch:~$ sudo ifreload --allow=mgmt

However, you can still bring the management interface up and down using
`ifup eth0` and `ifdown eth0`.

## Management VRF and DNS</span>

Cumulus Linux supports both DHCP and static DNS entries over management
VRF through IP FIB rules. These rules are added to direct lookups to the
DNS addresses out of the management VRF. However, nameservers configured
through DHCP are automatically updated, while statically configured
nameservers (configured in `/etc/resolv.conf`) only get updated when you
run `ifreload -a`.

Because DNS lookups are forced out of the management interface using FIB
rules, this could affect data plane ports if there are overlapping
addresses.

## Incompatibility with cl-ns-mgmt</span>

{{%notice warning%}}

Management VRF has replaced the management namespace functionality in
Cumulus Linux. The management namespace feature (via the `cl-ns-mgmt`
utility) has been deprecated, and the `cl-ns-mgmt` command has been
removed.

{{%/notice%}}

