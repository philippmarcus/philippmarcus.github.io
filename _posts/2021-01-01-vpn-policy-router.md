---
layout: post
title: Traffic Split VPN routing on a Raspberry Pi with Policy Routing 
date: 2021-01-1 13:32:20 +0300
description: Policy Routing with OpenVPN and the local internet connection based on the geo-location of targets.
img: vpn-policy-router/figure-setup.png # Add image post (optional)
fig-caption: # Add figcaption (optional)
excerpt_separator: <!--more-->
tags: [Policy Routing, Raspberry Pi, OpenVPN, IPTables]
---
Raspberry Pi OpenVPN IPTables
Media content such as news or streaming is often restricted to certain geolocations. This can be avoided by setting up a VPN on your home NAT router that tunnels to permitted geolocations, but then all traffic will be routed via that VPN interface. For local, domestic websites, this can either bring speed disadvantages or, in the most annoying case, new access restrictions.

This article describes a solution to this problem, whereby a Raspberry Pi is set up as a NAT router which routes ip packets to the internet either via the broadband router or a VPN tunnel depending on the destination of the respective IP packet, just like a traffic split functionality. If the IP packet has a domestic destination, the broadband router is used, otherwise the VPN tunnel device. The solution is based on OpenVPN, IPTables with the geoip add-on and Linux Policy Routing. The article assumes that you have access to a remote OpenVPN server for example NordVPN and that you have a local Raspberry Pi.

The charm of the solution is that the existing WLAN can continue to be used unchanged. Additionally, the Raspberry Pi is available as an alternative router for individual devices that require the VPN traffic split functionality.

<!--more-->

# Overview of the Setup

The following figure shows the setup that we will build up in this tutorial:

![Target Setup](/assets/img/vpn-policy-router/figure-setup.png)

The Raspberry Pi coexists in the same subnet with the WAN internet router and the local PCs. Local PCs that want to use our traffic split solution configure the Raspberry Pi as the default route. All other local PCs can continue to use the WAN internet router.

The traffic split functionality in the Raspberry Pi is configured to separate outgoing routing traffic based on the country in which the destination is located, with domestic and international destinations being distinguished. Traffic to international destinations is routed via the OpenVPN tunnel and traffic to domestic destinations is routed via the normal WAN internet router in the same subnet.

The solution will allow domestic websites to be accessed via a faster, direct connection and international websites via VPN to circumvent geolocation access restrictions.

The code of this tutorial can be found in my Github repository: [https://github.com/philippmarcus/vpn-policy-router](https://github.com/philippmarcus/vpn-policy-router).

# Configuring Raspberry Pi as a Router

So that the Raspberry Pi can work as a router in our setup, IP forwarding must be activated. This allows the Raspberry Pi to also accept IP packets that are not intended for it.

To do this, the following option must be set in the file `/etc/sysctl.conf`:

```
net.ipv4.ip_forward=1
```

So that this is activated immediately and not only after a reboot, we also execute:

```
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
```

In our case, IP forwarding is sufficient, we do not need masquerading, as this is done by the default home router and the VPN server respectively.

***Warning*** if you have a firewall like ufw installed, it could block your IP forwarding. You have to allow this first in your firewall config, for example follow this post for ufw: [https://askubuntu.com/questions/161346/how-to-configure-ufw-to-allow-ip-forwarding](https://askubuntu.com/questions/161346/how-to-configure-ufw-to-allow-ip-forwarding). This tutorial was tested with ufw deactivated.

# Setting up OpenVPN

First of all, OpenVPN must be installed and set up on the Raspberry Pi. The standard process can be followed. Thus, this tutorial will not go into this in more detail.

# Installing Required Packages

We have to install the GeoIP extension for IPTables and other packages as described below.

## Installing the GeoIP Add-on for IPTables

To mark IP packets based on the destination IP, we need the IPtables add-ons, which have to be installed as kernel modules. At the time of writing this tutorial, the corresponding package `xtables-addons-common` in Raspbian Buster was obviously faulty, which meant that the module had to be compiled manually. If this package is fixed in the future, you can basically [follow this tutorial] (https://www.linux-tips-and-tricks.de/en/raspberry/499-how-to-block-ip-ranges- with-geoip-on-raspbian /). The source code to be compiled can be found at [https://inai.de/projects/xtables-addons/](https://inai.de/projects/xtables-addons/). A prerequisite is that the `raspbian-kernel-headers` packages are installed beforehand. The following script checks out the add-on, does the compilation, and ensures that it is loaded every time it boots:

~~~ shell
cd ~
git clone https://git.inai.de/xtables-addons
cd xtables-addons

# This step is necessary on Raspbian Buster but not part of the official docu
autoreconf -i

./configure
make
make install

# Index newly created modules
depmod -a

# Load the xt_geoip module for this session
modprobe xt_geoip

# Ensure the module will always be loaded at boot
echo xt_geoip >> /etc/modules-load.d/modules.conf
~~~

The module `xt_geoip` will be called later by our IPTables rules. In order to assign IP addresses to country codes, the module requires that a suitable geographic database is available under `/usr/share/xt_geoip`. Two CLI tools are supplied for this purpose [^6] [^7]. We create a CRON job that updates this database once a week. Preparation of the storage location for the files:

~~~ shell
mkdir ~/cron_job_geoip
chmod 700 cron_job_geoip
~~~

The CLI tool `xt_geoip_dl_maxmind` that our CRON job will execute requires you to have a license key available, which can be obtained by registering with MindMax for free: [https://www.maxmind.com/en/geolite2/signup](https://www.maxmind.com/en/geolite2/signup). Store this license key:

~~~shell
echo <your license key here> >> ~/cron_job_geoip/.mindmax_license_key
chmod 700 ~/cron_job_geoip/.mindmax_license_key
~~~
 
To ensure that the GeoIP database is regularly updated, we create a script `geoip-db` in `~/ cron_job_geoip/`, an adapted version of [^9]:

~~~shell
#!/bin/bash

# end script immediately if any command fails
set -euo pipefail

geotmpdir=$(mktemp -d)
csv_files="GeoLite2-Country-Blocks-IPv4.csv GeoLite2-Country-Blocks-IPv6.csv"
OLDPWD="${PWD}"
cd "${geotmpdir}"
/usr/local/libexec/xtables-addons/xt_geoip_dl_maxmind ~/.mindmax_license_key
cd GeoLite2-Country-CSV_*
sudo /usr/local/libexec/xtables-addons/xt_geoip_build_maxmind -D /usr/share/xt_geoip ${csv_files}
cd "${OLDPWD}"
rm -r "${geotmpdir}"
exit 0
~~~

Set the authorization:

~~~ shell
chmod 700 ~/cron_job_geoip/geoip_db
~~~

After that the cron job is defined [^8]. We call and select `nano` as editor:

~~~ shell
crontab -e
~~~

The following entry is defined, whereby the times can be adjusted as required:

~~~ shell
0 	0 	* 	* 	7 	geoip_db
~~~

The database is now automatically updated every 7th day of the week. To make this script work, we have to install also the packages outlined in the next section.

## Other required packages

In a script shown below, we also use the package `ipcalc`. As the downloader scripts for the geoip databases are written in Perl, we also need to install the according Perl libraries `libtext-csv-xs-perl`.

# Automatic Setup of Policy Routing

When OpenVPN starts up the connection there is the possibility to run a setup script via a hook. The same applies to terminating connections. We will use this possibility to adapt the routing of the Raspberry Pi accordingly. Roughly speaking, the solution will take the following steps:

- Packets received from the home network and to be routed are marked with a `marker` 2 or 3 depending on the destination country
- A separate routing table is created for both markers, using either the tunnel or the local home router

These configurations are carried out in a script `up.sh`, which is called automatically by OpenVPN with the relevant parameters when the connection is established. They are made undone in a script `down.sh` as soon as OpenVPN disconnects.

## Marking IP Packages with Linux Netfilter and IPTables

To set the markers described, two Linux mechanisms are used, which are briefly described below for better understanding, Netfilter and IPTables.

The IPv4 protocol stack in the Linux kernel is implemented as a directed acyclic graph (DAG) which is traversed by IP packets. Netfilter is a mechanism in the Linux kernel that allows other kernel modules to register their own functions as callbacks/hooks on nodes of that DAG. These callbacks are then executed as soon as an IP packet passes the registered node in this graph. The callback function has, among other things, the ability to return a tampered version of the packet to Netfilter or to instruct Netfilter to accept or delete the packet.

We will use the ability to manipulate IP packets to set the marker that determines whether the IP packet should reach a domestic or a foreign destination. This marker is a logical marker within the Linux protocol stack, i.e. it will no longer be visible on the outgoing IP packet.

![The simplified IPv4 traversal diagram with Netfilter hooks in blue boxes [1].](/assets/img/vpn-policy-router/figure-hooks.png)

[^1]: [https://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO-3.html](https://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO-3.html)

IPTables is a program for IPv4 and IPv6 packet filtering and NAT that runs in the userspace and will help us to set these markers. It registers itself as callback function for Netfilter hooks by utilizing its associated kernel modules. When invoked by the hook, it applies its internally stored rules on transferred IP packets. The ultimate goal of these rules is, as described above, to instruct Netfilter whether the packet should be accepted, deleted, or to return a modified version of the packet to Netfilter for the further course in the DAG.

Depending on which hook is triggered by Netfilter, IPTables sequentially processes a list of chains. Chains contain the actual rules. Within IPTables, chains are grouped logically in tables, whereby roughly speaking tables correspond to dedicated certain task areas, for example, NAT or packet filtering. This has the advantage that there is always exactly one suitable table for every type of rule. The `man iptables` describes a total of 5 tables:

- `filter`: Table for filtering of IP Pakets.
- `nat`: This table is consulted when a packet  that  creates  a  new connection  is  encountered. 
- ***`mangle`***: This table is used for specialized packet alteration. This is the table where we will have to put the rule to add the marker to the IP Pakets.
- `raw`: This  table  is  used mainly for configuring exemptions from connection tracking in combination with the NOTRACK  target.
- `security`: This  table  is used for Mandatory Access Control (MAC) net‐working rules, such as those enabled  by  the SECMARK and CONNSECMARK  targets.

Transferred to the above DAG, the processing logic of IPtables can then be understood as follows:

![IPTables registers a sequence of chains for each Netfilter callback.](/assets/img/vpn-policy-router/figure-chains.png)

For example, if the `INPUT` hook is triggered in the above figure, the associated chains are processed sequentially, with each chain containing a list of rules. The rules in the chains essentially contain a condition and a target/jump.

A target is a concrete action that is applied to the IP packet. As described above, the most common options are to accept, reject or manipulate the package. Acceptance or rejection is usually determined by `-j ACCEPT` or` -j DROP`. When a rule is executed with one of these two targets, a decision has been made for this hook. Thus, IPTables stops the further evaluation of chains/rules for this hook and passes the decision as return value back to Netfilter.

Some targets can manipulate the IP packet by using a built-in procedure on the IP packet. This procedure returns the manipulated IP packet to the rule. The further execution of the chain is then done with this manipulated IP packet. Common built-in functions are `log`,` MASQUERADE`, and `MARK` that we will use. With a jump, the further execution can be delegated to a custom chain, after which the chain jumps back to the calling chain. We do not need this option for the tutorial [^3].

For the rule for marking the IP packets in this project we will need the table `mangle`, and have to work with the` PREROUTING` chain, because we have to start the hook `NF_IP_PRE_ROUTING` before the routing takes over. The table can be defined with the `-t mangle` option and the command to append our rule to a chain is defined by the `-A PREROUTING` option. In our command the condition is defined with the `-m` option and the target/jump part with ` -j`. From this the first part of the command that we will use can be derived. Placeholders for the missing parts are added as well:

~~~shell
sudo iptables -A PREROUTING -t mangle -m <Condition> -j <Target/Jump>
~~~

The condition can be defined via the previously installed `geoip` module:

~~~shell
iptables -A PREROUTING -t mangle -m geoip --src-cc country[,country...] --dst-cc country[,country...]
~~~

The condition can be negated with a `!`, whereby the rule then applies to all traffic that does not apply to the defined destination country code. With this knowledge, we define in the table `mangle` in the chain` prerouting` a rule for setting the marker 2 and 3, whereby we use the geoip module as a condition and only want to mark traffic coming from eth0 (no traffic coming from the tunnel) and traffic towards the internet:

~~~ shell
# Derive the subnet for eth0
dev="eth0"
dev_subnet_cidr=$(ipcalc $(ip -o -f inet addr show $dev | awk '/scope global/ {print $4}') | awk '/Network:/ {print $2}')

# mark outgoing packages to international destinations (outside of Germany) with marker 2
sudo iptables -A PREROUTING -t mangle -m geoip ! --dst-cc CN -i $dev -j MARK --set-mark 2

# mark outgoing packages to domestic locations (Germany in my case) with marker 3
sudo iptables -A PREROUTING -t mangle -m geoip --dst-cc CN -i $dev ! -d $dev_subnet_cidr -j MARK --set-mark 3
~~~

The variable `dev_subnet_cidr` used at the top of this script will in my case hold the value `192.168.0.0/24`. To adapt this example to your own situation, you have to replace the variable `dev` with the name of your local network adapter and the country code `DE` with the country code of the country in which the RaspberryPi is located. In the next step, we have to define a separate routing table for marker 2 and 3.

Is the marker also applied to packets that are generated locally by the Raspberry Pi? If we look at the DAG shown above, it becomes clear that locally generated packets never flow through the `PREROUTING` chains. So we know that our markers defined above are only applied to incoming IP packets on the RaspberryPi.

## Defining Policy Routing Tables

Three routing tables are created in by the kernel during startup and form the routing policy database (RPDB) of our system:

~~~shell
$ ip rule show
> 0:	from all lookup local
> 32766:	from all lookup main
> 32767:	from all lookup default
~~~

Every entry is a rule with an assigned priority, a selector and an action. In the above default RPDB, the selector is `from all` and the action are the names of the three route tables. When a new package arrives in the routing module, these rules are evaluated step by step with increasing priority from 0 - 32767 until the selector of a rule applies. The actions shown above all point to routing tables (local, main, default). The routing module then scans the selected routing table for a matching rule and stops if it has a match. Otherwise the IP rule list is further processed.

With the tool `ip-rule` we now add a rule to the RPDB that has a selector `fwmark` based on the marker that we set in the package before, and an action that points either to an international or domestic routing table that we will populate later:

~~~shell
# routing table for IP packets towards international destinations
sudo ip rule add fwmark 2 table 2 prio 15000

# routing table for IP packets towards domestic destinations
sudo ip rule add fwmark 3 table 3 prio 15100
~~~

Now our two routing tables have been created and the condition for `fwmark` is visible:

~~~shell
$ ip rule show
> 0:		from all lookup local
> 15000:	from all fwmark 0x2 lookup 2
> 15100:	from all fwmark 0x3 lookup 3
> 32766:	from all lookup main
> 32767:	from all lookup default
~~~

If a local computer connected now via LAN and uses the Raspberry Pi as a router/gateway, the packets get correctly marked now. Since the two new referenced routing tables are still empty, no match is found in them, and the evaluation proceeds to the table `main`. To change this, we use the tool `ip-route` to add a rule to the routing table for international traffic that prohibits any traffic by default:

~~~ shell
# Prohibit packets towards international destinations (overridden by up.sh)
sudo ip route add prohibit default table 2
~~~

The used `prohibit` routing rule is defined according to the ip-route manpage as:

> prohibit - the rule prescribes to generate 'Communication is administratively prohibited' error.

Later, when our OpenVPN connection is established, we will additionally add routing rules to the international table, which route via the tunnel by default.

## Defining OpenVPN up and down Scripts

Traffic to domestic destinations is routed through the domestic table, which we populate using the same home WAN router as in the main table:

~~~ shell
# Route for domestic packets
sudo ip route add default table 3 via $router_ip dev $dev
~~~

OpenVPN can execute a script `up` and` down` when establishing and disconnecting a connection. This can be added in the `.ovpn` configuration file with` up up.sh` and `down down.sh`. The script `up.sh`, which is executed after VPN connection establishment, must extend the international routing table with default routes over the newly installed tunnel device. We define the file `up.sh`:

~~~ shell
#!/bin/bash

# Configure tunnel as default routes for table international
sudo ip route add 0.0.0.0/1 via $route_vpn_gateway dev $dev table 2
sudo ip route add 128.0.0.0/1 via $route_vpn_gateway dev $dev table 2

# SNAT for packages routed via the tunnel
sudo iptables -t nat -A POSTROUTING -o $dev -j MASQUERADE
~~~

So that we can leave the default route via `prohibit`, we use longest prefix matching here and defined the new default route via VPN using:

```
128.0.0.0/128.0.0.0 (covers 0.0.0.0 thru 127.255.255.255)
0.0.0.0/128.0.0.0 (covers 128.0.0.0 thru 255.255.255.255)
```

So that this configuration can be made undone when the VPN connection is disconnected, we define the script `down.sh`:

~~~ shell
#!/bin/bash

# Remove tunnel as default routes for table international
sudo ip route del $untrusted_ip via $route_net_gateway dev eth0 table 2
sudo ip route del $eth0_network dev eth0 proto dhcp scope link src $eth0_ip metric 202 table 2

# Remove SNAT for packages routed via the tunnel
sudo iptables -t nat -D POSTROUTING -o $dev -j MASQUERADE
~~~

Please note that we use source NAT on the packets that are routed via the VPN tunnel. This hides the internal network from incoming internet connections and the VPN provider for security reasons.

Also used are the variables `$ dev` and` $ route_vpn_gateway`, which OpenVPN sets as environmental variables when establishing the connection and which we need for our script [^4]:

- `$dev`: The actual name of the TUN/TAP device, including a unit number if it exists. Set prior to –up or –down script execution.
- `$route_vpn_gateway`: The default gateway used by –route options, as specified in either the –route-gateway option or the second parameter to –ifconfig when –dev tun is specified. Set prior to –up script execution.

If clients now use the Raspberry Pi as a gateway/router, you can check whether the correct routing table is being used via `tracert <ip>` (Windows) or `traceroute <ip>` (Linux), depending on the destination address.

## Package Flow when the Tunnel is Up

In this section, the package flow in the system should be traced in detail for better understanding.

![Package flow in the setup](/assets/img/vpn-policy-router/figure-flow.png)

Based on this figure, the routing processes are discussed below.

### Routing Flow for International Packages

Based on the above figure, the following flow results for outgoing packages to international destinations:

- Packet arrives from PC in local network on eth0
- (1): PREROUTING mangle sets marker 2 on the package
- (2): IP-Rule selects the table internationally and recognizes that the packet must be routed via tun0
- (3): No special action.
- (4): SNAT is applied to the packets and written to the tun0 device
- (5): The VPN driver creates a new IP packet. It no longer has a marker and contains the packet written in tun0 before as a payload. Its destination is the IP of the OpenVPN server.
- (6): No special action
- (7): IP-Rule selects the main routing table and recognizes that the OpenVPN server is reachable via eth0.
- (8): No further modification or SNAT. The packet is passed to eth0.

If the response packets arrive at tun0, the flow essentially runs backward. However, there is the difference that the response packets from international destinations do not get a marker (marker rule was restricted with `-i`). Thus, IP-Rule routes the response via the main routing table back to the local network PC. This works because the main routing table contains a route to this subnet.

### Routing Flow for Domestic Packages

If a PC in the local network routes a package via the Raspberry Pi to domestic destinations, the following workflow runs:

- Packet arrives from PC in local network on eth0
- (1): REROUTING mangle sets marker 3 on the package
- (2): IP-Rule selects the domestic table and recognizes that the packet must be routed to the WAN router via eth0
- (3): No special action.
- (4): The packet is written without SNAT in eth0 in the direction of the home WAN router, which then performs SNAT.

The response flow is essentially exactly the opposite direction, whereby the response has no marker and is thus routed again via the main routing table to the PC in the local network via eth0.

## Lessons Learned und Debugging

Perhaps my biggest learning step in this project was how to use the debugging tools correctly. At first, I forgot to restrict the marking only to packets received via the eth0 interface. The consequence of this was that 
VPN packets that were to be routed back to a computer in the LAN via tun0 also have received this marker and consequently were sent to the domestic routing table. There, no matching rule was found for packets towards the home network. Thus, only domestic routing worked in the beginning, but not international routing.

To find this out, I used the LOG functionality of IPTables to log all packets with marker 2 (international). The goal was to understand where the packets were going to:

~~~ shell
sudo iptables -A PREROUTING -t mangle  -j LOG --log-prefix "PREROUTING mangle :" -m mark --mark 2
sudo iptables -A INPUT -t filter  -j LOG --log-prefix "INPUT filter :" -m mark --mark 2
sudo iptables -A OUTPUT -t filter  -j LOG --log-prefix "OUTPUT filter :" -m mark --mark 2
sudo iptables -A OUTPUT -t raw  -j LOG --log-prefix "OUTPUT raw :" -m mark --mark 2
sudo iptables -A POSTROUTING -t mangle  -j LOG --log-prefix "POSTROUTING mangle :" -m mark --mark 2
sudo iptables -A FORWARD -t filter  -j LOG --log-prefix "FORWARD filter :" -m mark --mark 2
sudo iptables -A FORWARD -t mangle  -j LOG --log-prefix "FORWARD mangle :" -m mark --mark 2
~~~

Then I watched the logged packets on the shell, e.g. for the FORWARD chains:

~~~ shell
journalctl -f | grep FORWARD
~~~

There I was able to observe that incoming packets from tun0 in the direction of a client in the local network also carry this marker and run over the international routing table. Since this has no route to the local network, the solution could not work. So I added the condition `-i eth0` to the iptables rules. The incoming packets from tun0 are then routed via the main routing table.

What was also very helpful at the beginning is the logging functionality of iptables itself, which shows how many packets were matched by which rule. So I could check that the two rules for setting marker 2 and 3 for domestic or international traffic actually worked:

~~~ shell
iptables -L -nv -t mangle
~~~

The number of packets hit and the total number of bytes are then displayed. I also experimented with `tcpdump`:

~~~ shell
tcpdump -i eth0
~~~

However, tcpdump cannot display the set markers. It only served me to show that there was traffic on a certain interface and that the internal clients were correctly sending requests to eth0.

It has also proven helpful to check the routing tables on the clients to see whether the Raspberry Pi or the Internet router is being used. On Windows this is done with `route PRINT` and on Linux as mentioned above e.g. with `ip route show`.

# Conclusion

In this article, I presented my Raspberry Pi based solution for a VPN traffic split router. The Raspberry Pi coexists as a router in the same subnet as the WAN internet router and can be used as a router by individual devices if required. The traffic split functionality separates outgoing routing traffic based on the country in which the destination is located, with domestic and international destinations being distinguished. IPTables was used to mark IP packets with the correct marker. Separate routing tables with policy routing were created for domestic and international destinations and selected based on that marker. Traffic to international destinations is routed via the OpenVPN tunnel and traffic to domestic destinations is routed via the normal WAN internet router in the same subnet.

The solution now allows domestic websites to be accessed via a faster, direct connection and international websites via VPN to circumvent geolocation access restrictions.

The solution is not perfect yet and needs to be expanded in the future. Firstly, there is a problem with the DNS resolver. If a PC in the local network that uses the Raspberry Pi as a router uses a domestic DNS server, there is a risk that geolocation restrictions for international destinations are already applied in the DNS resolution, and VPN routing is useless. Likewise, in countries with officially blocked websites, any page views that are later routed via VPN are first sent unencrypted to a domestic DNS server. The request and response can easily be sniffed, logged, and punished in the worst-case. It is also annoying that in such cases, DNS spoofing can enforce access restrictions/blocks. Contrary, if you use an international DNS server abroad, this problem can again arise for domestic websites. In detail, when the DNS queries are routed there via the traffic split over the VPN tunnel, the international DNS server could already enforce the geolocation restriction based on your VPN server's location. To solve this issue, one idea is to write a Facade DNS resolver on the Raspberry Pi, which forwards the requests to either a domestic or an international DNS server via a mechanism that has yet to be developed.

Another possible improvement would be to configure a DHCP server directly on the Raspberry Pi. The default router can then be specified for each device in the home network, for example, whether to use the Raspberry Pi with the traffic split or the default home router as usual.

I hope that this project is also helpful for others and contributes to a better understanding of routing and iptables.


[^2]: [https://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO-4.html#ss4.1] (https://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO-4.html#ss4.1)
[^3]: [http://www.faqs.org/docs/iptables/targets.html](http://www.faqs.org/docs/iptables/targets.html)
[^4]: [https://openvpn.net/community-resources/reference-manual-for-openvpn-2-4/](https://openvpn.net/community-resources/reference-manual-for-openvpn-2-4/)
[^5]: https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture
[^6]:[https://www.mankier.com/1/xt_geoip_dl](https://www.mankier.com/1/xt_geoip_dl)
[^7]: [https://www.mankier.com/1/xt_geoip_build](https://www.mankier.com/1/xt_geoip_build)
[^8]: [https://daenney.github.io/2017/01/07/geoip-filtering-iptables.html](https://daenney.github.io/2017/01/07/geoip-filtering-iptables.html)
[^9]: [https://www.raspberrypi.org/documentation/linux/usage/cron.md](https://www.raspberrypi.org/documentation/linux/usage/cron.md)
[^10]: [http://www.policyrouting.org/PolicyRoutingBook/](http://www.policyrouting.org/PolicyRoutingBook/)