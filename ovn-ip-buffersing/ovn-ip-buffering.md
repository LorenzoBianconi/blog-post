# IP Buffering in OVN

### Introduction:
_Open Virtual Network (OVN)_ is a subproject of [Open vSwitch (OVS)](http://www.openvswitch.org/), a performant
programmable multi-platform virtual switch. OVN adds to the OVS existing
capabilities the support for overlay networks introducing virtual network
abstractions such as virtual switches and routers. Moreover OVN provides native
methods for setting up Access Control Lists (ACLs) and network services such as
DHCP.

### OVN Architecture:
An OVN deployment consists of several components:  
* The OVN/CMS Plugin (e.g Neutron) is the CMS interface component for OVN. The plugin’s main purpose
  is to translate the CMS’s notion of logical network configuration into an intermediate representation
  composed by logical switches and routers which can be interpreted by OVN.
* The OVN Northbound database is an OVSDB instance responsible for storing network representation
  received from the CMS plugin. The OVN Northbound Database has only two clients: the OVN/CMS Plugin
  above it and ovn−northd daemon
* ovn−northd daemon connects to the OVN Northbound Database and the OVN Southbound
  Database. It translates the logical network configuration in terms of conventional
  network concepts, taken from the OVN Northbound Database, into logical datapath
  flows in the OVN Southbound Database
* The OVN Southbound database, an OVSDB database, is characterized by a quite different
  schema with respect to the NB DB. In particular, instead of familiar networking concepts,
  the SB DB defines the network in terms of match-action rule collections called
  "logical flows". The “logical flows”, while conceptually similar to OpenFlow flows,
  exploit logical concepts, such as virtual machine instances, instead of physical ones
  like physical Ethernet ports. In particular, SB DB includes three data types:
  * Physical network data, such as the VM's IP address and tunnel encapsulation
    format
  * Logical network data, such as packet forwarding mode
  * Binding relationship between physical network and logical network
* The ovn-controller daemon. A copy of ovn-controller runs on each hypervisor.
  ovn-controller reads logical flows & bindings from the SB DB, translates them
  into OpenFlow flows, and sends them to Open vSwitch's ovs-vswitchd daemon

<p align="center"> 
<img src="https://github.com/LorenzoBianconi/ip_buffersing_blog/blob/master/img/architecture.svg">
</p>

### L2 address resolution problem:
A typical OVN deployment is reported below where the overlay network
is connected to an external one through a _localnet_ port (ext-localnet in this
case)

<p align="center"> 
<img src="https://github.com/LorenzoBianconi/ip_buffersing_blog/blob/master/img/ip_buff_diagram.svg"
</p>

Below is reported related OVN NBDB network configuration:

> switch 35b34afe-ee16-469b-9893-80b024510f33 (sw2)  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port sw2-port4  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;addresses: ["00:00:02:00:00:04 1.2.0.4 2001:db8:2::14 "]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port sw2-port3  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;addresses: ["00:00:02:00:00:03 1.2.0.3 2001:db8:2::13 "]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port sw2-portr0  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;type: router  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;addresses: ["00:00:02:ff:00:02"]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;router-port: lrp2  
> switch c16e344a-c3fe-4884-9121-d1d3a2a9d9b1 (sw1)  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port sw1-port1  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;addresses: ["00:00:01:00:00:01 1.1.0.1 2001:db8:1::11 "]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port sw1-portr0  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;type: router  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;addresses: ["00:00:01:ff:00:01"]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;router-port: lrp1  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port sw1-port2  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;addresses: ["00:00:01:00:00:02 1.1.0.2 2001:db8:1::12 "]  
> switch ee2b44de-7d2b-4ffa-8c4c-2e1ac7997639 (sw-ext)  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port ext-localnet  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    type: localnet  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    addresses: ["unknown"]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port ext-lr0  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    type: router  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    addresses: ["02:0a:7f:00:01:29"]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    router-port: lr0-ext  
> router 681dfe85-6f90-44e3-9dfe-f1c81f4cfa32 (lr0)  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port lrp2  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    mac: "00:00:02:ff:00:02"  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    networks: ["1.2.254.254/16", "2001:db8:2::1/64"]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port lr0-ext  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    mac: "02:0a:7f:00:01:29"  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    networks: ["192.168.123.254/24", "2001:db8:f0f0::1/64"]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port lrp1  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    mac: "00:00:01:ff:00:01"  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    networks: ["1.1.254.254/16", "2001:db8:1::1/64"]  

Whenever a device belonging to the overlay network (e.g PC1) tries to reach an external
device (e.g PC-EXT) it forwards the packet to the OVN logical router (LR0). If LR0
has not already resolved the L2/L3 address correspondence for PC-EXT yet, it
will send an ARP frame (or a Neighbor Discovery for IPv6 traffic) for PC-EXT.
The current OVN implementation employs  _arp action_ to perform L2 address resolution. In
other words, OVN will instruct OVS to perform a 'packet in' action whenever it
needs to forward an IP packet for an unknown L2 destination. The ARP action
replaces the IPv4 packet being processed by an ARP frame that is forwarded on
the external network to resolve PC-EXT mac address.
Below is reported the IPv4/IPv6 OVN SBDB rules corresponding to that processing:

> table=10(lr_in_arp_request  ), priority=100  , match=(eth.dst == 00:00:00:00:00:00), action=(arp { eth.dst = ff:ff:ff:ff:ff:ff; arp.spa = reg1; arp.tpa = reg0; arp.op = 1; output; };)  
> table=10(lr_in_arp_request  ), priority=100  , match=(eth.dst == 00:00:00:00:00:00), action=(nd_ns { nd.target = xxreg0; output; };)  

The main drawback introduced by the described processing is the loss of the first
packet of the connection (as shown in the following ICMP traffic) introducing
latency in TCP connections established with devices not belonging to the
overlay network

> PING 192.168.123.10 (192.168.123.10) 56(84) bytes of data.  
> 64 bytes from 192.168.123.10: icmp_seq=2 ttl=63 time=0.649 ms  
> 64 bytes from 192.168.123.10: icmp_seq=3 ttl=63 time=0.321 ms  
> 64 bytes from 192.168.123.10: icmp_seq=4 ttl=63 time=0.331 ms  
> 64 bytes from 192.168.123.10: icmp_seq=5 ttl=63 time=0.137 ms  
> 64 bytes from 192.168.123.10: icmp_seq=6 ttl=63 time=0.125 ms  
> 64 bytes from 192.168.123.10: icmp_seq=7 ttl=63 time=0.200 ms  
> 64 bytes from 192.168.123.10: icmp_seq=8 ttl=63 time=0.244 ms  
> 64 bytes from 192.168.123.10: icmp_seq=9 ttl=63 time=0.224 ms  
> 64 bytes from 192.168.123.10: icmp_seq=10 ttl=63 time=0.271 ms  
>
> --- 192.168.123.10 ping statistics ---  
> 10 packets transmitted, 9 received, 10% packet loss, time 9214ms  

In order to overcame this limitation a
[solution](https://github.com/openvswitch/ovs/commit/d7abfe39cfd234227bb6174b7f959a16dc803b83) has been proposed
where incoming IP frames that have no corresponding L2 address yet are
queued and will be re-injected to ovs-vswitchd as soon as the neighbor discovery
processed is completed. Repeating the above tests prove that even the first
ICMP echo request is received by PC-EXT

> PING 192.168.123.10 (192.168.123.10) 56(84) bytes of data.  
> 64 bytes from 192.168.123.10: icmp_seq=1 ttl=63 time=1.92 ms  
> 64 bytes from 192.168.123.10: icmp_seq=2 ttl=63 time=0.177 ms  
> 64 bytes from 192.168.123.10: icmp_seq=3 ttl=63 time=0.277 ms  
> 64 bytes from 192.168.123.10: icmp_seq=4 ttl=63 time=0.139 ms  
> 64 bytes from 192.168.123.10: icmp_seq=5 ttl=63 time=0.281 ms  
> 64 bytes from 192.168.123.10: icmp_seq=6 ttl=63 time=0.247 ms  
> 64 bytes from 192.168.123.10: icmp_seq=7 ttl=63 time=0.211 ms  
> 64 bytes from 192.168.123.10: icmp_seq=8 ttl=63 time=0.187 ms  
> 64 bytes from 192.168.123.10: icmp_seq=9 ttl=63 time=0.439 ms  
> 64 bytes from 192.168.123.10: icmp_seq=10 ttl=63 time=0.253 ms  
>  
> --- 192.168.123.10 ping statistics ---  
> 10 packets transmitted, 10 received, 0% packet loss, time 9208ms  

### Future development:
A possible future enhancement to the described methodology could be use the developed
IP buffering infrastructure to queue packets waiting for a given events
(e.g. the container for which the packet is destinated for come up) and then send
them back to ovs-vswitchd as soon as the requested message has been received.
Stay tuned :)
