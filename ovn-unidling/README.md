# OVN Unidling:

Open Virtual Network (OVN) is a project born as subcomponent of [Open vSwitch (OVS)](http://www.openvswitch.org/), a performant, programmable, multi-platform virtual switch.
OVN allows OVS users to natively creates overlay networks by introducing
virtual network abstractions such as virtual switches and routers.  Moreover,
OVN provides methods for setting up Access Control Lists (ACLs) and network
services such as DHCP. Many Red Hat products, like Red Hat OpenStack Platform,
Red Hat Virtualization and Red Hat OpenShift Container Platform relies on OVN
to configure network functionalities.

In this article, I will cover the OVN <em>unidling</em> issue and how the proposed
solution can be used to forward events to the CMS (e.g OpenStack or OpenShift).

### Unidling problem (Openshift use case):
A simplified OVN-Kubernetes deployment is shown below where the overlay network is
connected to an external one through a localnet port (ln-public, in this case):

<p align="center"> 
<img src="https://github.com/LorenzoBianconi/ovn-unidling-blog/blob/master/images/ovn-unidling.svg">
</p>

Below is reported related OVN NB db network configuration:

> switch e2564770-8658-4086-8f41-9995d5ff0da2 (sw1)  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port sw1-p0  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;addresses: ["00:00:00:00:00:33 192.168.2.11"]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port lrp1-attachment  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;type: router  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;addresses: ["00:00:00:ff:00:02"]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;router-port: lrp1  
> switch 512be578-1c95-4ac0-b196-8f5ef38a1517 (sw0)  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port sw0-p0  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;addresses: ["00:00:00:00:00:11 192.168.1.11"]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port sw0-p1  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;addresses: ["00:00:00:00:00:12 192.168.1.12"]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port lrp0-attachment  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;type: router  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;addresses: ["00:00:00:ff:00:01"]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;router-port: lrp0  
> switch ee2b44de-7d2b-4ffa-8c4c-2e1ac7997639 (public)  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port ln-public  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    type: localnet  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    addresses: ["unknown"]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port lrp2-attachment  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    type: router  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    addresses: ["00:00:00:00:ff:03"]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    router-port: lrp2  
> router 681dfe85-6f90-44e3-9dfe-f1c81f4cfa32 (lr0)  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port lrp2  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    mac: "00:00:00:00:ff:03"  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    networks: ["192.168.3.254/24"]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port lrp1  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    mac: "00:00:00:00:ff:02"  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    networks: ["192.168.2.254/24"]  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port lrp0  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    mac: "00:00:00:00:ff:01"  
>    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    networks: ["192.168.1.254/24"]  

OVN Load Balancer (LB) services are used to demultiplex traffic between running pods.
LB configuration is stored in the Load_Balancer table of OVN Northd (OVN-NB) db:

> _uuid               : 7381bdc2-cb26-40e9-93db-d7f733c8afbd  
> external_ids        : {}  
> health_check        : []  
> ip_port_mappings    : {}  
> name                : lb0  
> protocol            : tcp  
> vips                : {"192.168.1.100:80"="192.168.1.11:80,192.168.1.12:80"}  

However, after an inactivity timeout a given pod can be powered down by Openshift
and the related back-ends are removed from the load balancer configuration resulting
in a Virtual IP (VIP) with no backends:

> _uuid               : f93bca28-87b4-4d98-9193-b49644f15ee6  
> external_ids        : {}  
> health_check        : []  
> ip_port_mappings    : {}  
> name                : lb0  
> protocol            : tcp  
> vips                : {"192.168.1.100:80"=""}  

As a consequence the system results in a deadlock state since a new packet for
the 'suspended' service will not be forwarded by OVN to the related pod without
a proper network configuration.

### Proposed solution (Controller_Event):
In order to overcome this limitation, a [solution](https://github.com/openvswitch/ovs/commit/f732a1ab9c574c1c17858a84cf7d25f294dfb151) has been proposed by which
a new table,
[Controller_Event](https://github.com/ovn-org/ovn/blob/master/ovn-sb.ovsschema#L355),
has been added to the OVN South Bound db. Moreover, new <em>trigger_event</em> logical flows have been introduced into
OVN pipelines in order to generate a <em>controller event</em> whenever an IP packet for
a LB rule with no back-ends is received by OVN.

> table=4 (ls_in_pre_lb       ), priority=130  , match=(ip4.dst == 192.168.1.100 && tcp && tcp.dst == 80), action=(trigger_event(event = "empty_lb_backends", meter = "event-elb", vip = "192.168.1.100:80", protocol = "tcp", load_balancer = "38350663-862f-4aae-94e7-c0149e11d293");)                                                                           

The OVN trigger_event action will
convert an unsolicited event into a new row in the Controller_Event table allowing
the CMS to be notified about the request for the 'suspended' service.

> _uuid               : c4d5493a-a630-47f8-adbb-e20a402e69de  
> chassis             : 24852cd2-bea6-48fd-b77a-95d2e47c836c  
> event_info          : {load_balancer="9d6542eb-6533-4d3c-b0a5-4e54826968b6", protocol=tcp, vip="192.168.1.100:80"}  
> event_type          : empty_lb_backends  
> seq_num             : 1  

Recently [controller event](https://github.com/ovn-org/ovn-kubernetes/commit/7a789d00f89e90f29bdba3abfab8a797c242c8dc) has been also integrated in even ovn-kubernetes. 

### Future development:
Since the proposed framework is not tied just to the undling scenario, a possible
future enhancement to the described methodology could be to extend the trigger_event
action in order to report more unsolicited events to the attention of the CMS in order
to allow the CMS to take necessary actions
