---
layout: post
title: Partial VPN routing on a Raspberry Pi with Policy Routing 
date: 2019-01-20 13:32:20 +0300
description: Policy Routing with OpenVPN and the local internet connection based on the geo-location of targets.
img: vpn-policy-routing/banner.png # Add image post (optional)
fig-caption: # Add figcaption (optional)
excerpt_separator: <!--more-->
tags: [Policy Routing, Raspberry Pi, OpenVPN, IPTables]
---

Media content such as news or streaming is often restricted to certain geolocations. This can be avoided by setting up a VPN in the permitted geolocations, but then all traffic will be routed via the VPN interface. For local, domestic websites, this can either bring speed disadvantages or, in the most annoying case, new access restrictions.

This article describes a solution to this problem, whereby a Raspberry Pi is set up as a partial VPN router in the home network. Depending on the destination country of outgoing IP packets, this defines either the VPN as the next hop router or, as usual, the local internet router. The solution is based on OpenVPN, IPTables with the geoip add-on and Linux Policy Routing. The article assumes that you have access to a remote OpenVPN server for example NordVPN and that you have a local Raspberry Pi.
<!--more-->

## Setting up OpenVPN

Todo describe how to install OpenVPN

## Installing Required Packages

Todo describe how to install geoip and xtables

## Automatic Setup of Policy Routing

When OpenVPN starts up the connection there is the possibility to run a setup script via a hook. The same applies to terminating connections. We will use this possibility to adapt the routing of the Raspberry Pi accordingly. Roughly speaking, the solution will take the following steps:

- Packets received from the home network and to be routed are marked with a `marker` 2 or 3 depending on the destination country
- A separate routing table is created for both markers, using either the tunnel or the local home router

These configurations are carried out in a script `up.sh`, which is called automatically by OpenVPN with the relevant parameters when the connection is established. They are made undone in a script `down.sh` as soon as OpenVPN disconnects.

### Marking IP Packages with Linux Netfilter and IPTables

To set the markers described, two Linux mechanisms are used, which are briefly described below for better understanding, Netfilter and IPTables.

The IPv4 protocol stack in the Linux kernel is implemented as a directed acyclic graph (DAG) which is traversed by IP packets. Netfilter is a mechanism in the Linux kernel that allows other kernel modules to register their own functions as callbacks/hooks on nodes of that DAG. These callbacks are then executed as soon as an IP packet passes the registered node in this graph. The callback function has, among other things, the ability to return a tampered version of the packet to Netfilter or to instruct Netfilter to accept or delete the packet.

We will use the ability to manipulate IP packets to set the marker that determines whether the IP packet should reach a domestic or a foreign destination. This marker is a logical marker within the Linux protocol stack, i.e. it will no longer be visible on the outgoing IP packet.

![The simplified IPv4 traversal diagram with Netfilter hooks in blue boxes [1].](assets/img/vpn-policy-router/ip-stack-dag.png)

IPTables is a program for IPv4 and IPv6 packet filtering and NAT that runs in the userspace. It registers for Netfilter hooks by utilizing its associated kernel modules and apply its internally stored rules on transferred IP packets. The ultimate goal of these rules is, as described above, to instruct Netfilter whether the packet should be accepted, deleted, or to return a modified version of the packet to Netfilter for the further course in the DAG.

Depending on which hook is triggered by Netfilter, IPTables sequentially processes a list of chains. Chains contain the actual rules. Chains are grouped logically in tables in IPTables, whereby roughly speaking tables correspond to dedicated certain task areas, for example, NAT or packet filtering. This has the advantage that there is always exactly one suitable table for every type of rule. The `man iptables` describes a total of 5 tables:

- `filter`: Table for filtering of IP Pakets.
- `nat`: This table is consulted when a packet  that  creates  a  new connection  is  encountered. 
- ***`mangle`***: This table is used for specialized packet alteration. This is the table where we will have to put the rule to add the marker to the IP Pakets.
- `raw`: This  table  is  used mainly for configuring exemptions from connection tracking in combination with the NOTRACK  target.
- `security`: This  table  is used for Mandatory Access Control (MAC) net‐working rules, such as those enabled  by  the SECMARK and CONNSECMARK  targets.

Transferred to the above DAG, the processing logic of IPtables can then be visualized as follows:

![IPTables registers a sequence of chains for each Netfilter callback.](assets/img/vpn-policy-router/iptables-chains.png)

For the rule for marking the IP packets in this project we will need the table `mangle`, and have to work with the` PREROUTING` chain, because we have to start the hook `NF_IP_PRE_ROUTING` before the routing.

For the rules write the `man iptables`:

>    A firewall rule specifies criteria for a packet and a target. If  the
   	  packet  does  not  match, the next rule in the chain is examined; if it
   	  does match, then the next rule is specified by the value of the target,
    which  can  be the name of a user-defined chain, one of the targets de‐
   	  scribed in iptables-extensions(8), or one of the special values ACCEPT,
	 DROP or RETURN. 

In order to recognize packets with a foreign destination IP we need a rule with a `match`, see` man iptables`:

>    	-m, --match match
			Specifies a match to use, that is, an extension module that 
			tests for a specific property. The set of matches make up the
			condition under which a target is invoked.  Matches  are 
			evaluated  first to last as specified on the command line
			and work in short-circuit fashion, i.e. if one extension
			yields false, evaluation will stop.

The match must reference the extension module `geoip`. In addition, the rule must define a so-called target, i.e. the command to mark the package with marker 2. This is done using a so-called builtin target. According to `man iptables`:

>      -j, --jump target
			This specifies the target of the rule; i.e., what to do
			if the packet matches it. The target can be a user-defined
			chain (other than the one this rule is in), one of the 
			special builtin targets which decide the fate of the packet
			immediately, or an extension (see EXTENSIONS below). If this
			option is omitted in a rule (and -g is not used), then matching
			the rule will have no effect on the packet's fate, but the counters
			on the rule will be incremented.

We use the builtin target `MARK`. The final command is then:

~~~shell
sudo iptables -A PREROUTING -t mangle -m geoip ! --dst-cc CN -j MARK --set-mark 2
~~~


[^1]: [https://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO-3.html](https://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO-3.html)

[^2]: [https://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO-4.html#ss4.1] (https://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO-4.html#ss4.1)



 --- Seperator ---


[^5]: https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture





[^2]: https://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO-3.html

The iptables firewall uses tables to organize its rules. These tables classify rules according to the type of decisions they are used to make. Within each iptables table, rules are further organized within separate "chains"
Chains map to netfilter hooks. Rules are placed within a specific chain of a specific table. A target is the action that are triggered when a packet meets the matching criteria of a rule. [^1].

-j MARK: Only valid in mangle table. Note that the mark value is not set within the actual package, but is a value that is associated within the kernel with the packet. In other words does not make it out of the machine [^1].

target jumps: http://www.faqs.org/docs/iptables/targets.html

[^1]: https://gist.github.com/mcastelino/c38e71eb0809d1427a6650d843c42ac2

```shell
# mark packages to rest of world with marker 2
sudo iptables -A PREROUTING -t mangle -m geoip ! --dst-cc CN -j MARK --set-mark 2

# setting up policy routing table 2 (rest of world)
sudo ip rule add fwmark 2 table 2
sudo ip route add 0.0.0.0/1 table 2 via $route_vpn_gateway dev $dev
sudo ip route add default table 2 via $route_net_gateway dev eth0 src 192.168.0.223 metric 202
sudo ip route add 10.8.8.0/24 table 2 dev $dev proto kernel scope link src $ifconfig_local
sudo ip route add $ifconfig_pool_remote_ip table 2 via $route_net_gateway  dev eth0
sudo ip route add 128.0.0.0/1 table 2 via $route_vpn_gateway dev $dev
sudo ip route add 192.168.0.0/24 table 2 dev eth0 proto kernel scope link src 192.168.0.223 metric 202

# mark packages to China with marker 3
sudo iptables -A PREROUTING -t mangle -m geoip --dst-cc CN -j MARK --set-mark 3

# setting up policy routing table 2 (rest of world)
sudo ip rule add fwmark 3 table 3
sudo ip route add default table 3 via $route_net_gateway dev eth0 src 192.168.0.223 metric 202
sudo ip route add $ifconfig_pool_remote_ip table 3 via $route_net_gateway dev eth0
sudo ip route add 192.168.0.0/24 table 3 dev eth0 proto kernel scope link src 192.168.0.223 metric 202
```

[^3]: Manpage of IPTables