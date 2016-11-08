---
title: Frequently Asked Questions
layout: docwithnav
---

* TOC
{:toc}

## "Why use Calico?"

The problem Calico tries to solve is the networking of workloads (VMs,
containers, etc) in a high scale environment. Existing L2 based methods
for solving this problem have problems at high scale. Compared to these,
we think Calico is more scalable, simpler and more flexible. We think
you should look into it if you have more than a handful of nodes on a
single site.

Calico also provides a rich network security model that
allows operators and developers to declare intent-based network security
policy that is automatically rendered into distributed firewall rules 
across a cluster of containers, VMs, and/or servers.

For a more detailed discussion of this topic, see our blog post at 
[Why Calico?](http://www.projectcalico.org/why-calico/).

## "Does Calico work with IPv6?"

Yes! Calico's core components support IPv6 out-of-the box. However,
not all orchestrators that we integrate with support IPv6 yet.

## "Why does my container have a route to 169.245.1.1?"

In a Calico network, each host acts as a gateway router for the 
workloads that it hosts.  In container deployments, Calico uses 
169.245.1.1 as the address for the Calico router.  By using a 
link-local address, Calico saves precious IP addresses and avoids 
burdoning the user with configuring a suitable address.

While the routing table may look a little odd to someone who is used to 
configuring  LAN networking, using explicit routes rather than 
subnet-local gateways is fairly common in WAN networking.

## Why can't I see the 169.245.1.1 address mentioned above on my host?

Calico tries hard to avoid interfering with any other configuration
on the host.  Rather than adding the gateway address to the host side
of each workload interface, Calico sets the `proxy_arp` flag on the 
interface.  This makes the host behave like a gateway, responding to
ARPs for 169.245.1.1 without having to actually allocate the IP address 
to the interface.

## Can I prevent my Kubernetes pods from initiating outgoing connections?

The Kubernetes [NetworkPolicy](http://kubernetes.io/docs/api-reference/extensions/v1beta1/definitions/#_v1beta1_networkpolicy) 
API doesn't currently support this.  However,
Calico does!  You can use `calicoctl` to configure egress policy to prevent
Kubernetes pods from initiating outgoing connections based on the full set of 
supported Calico policy primitives including labels, Kubernetes namespaces,
CIDRs, and ports.

## I've heard Calico uses proxy ARP, doesn't proxy ARP cause a lot of problems?

It can, but not in the way that Calico uses it.

In container deployments, Calico only uses proxy ARP for resolving the 
169.245.1.1 address.  The routing table inside the container ensures
that all traffic goes via the 169.245.1.1 gateway so that is the only
IP that will be ARPed by the container.

## "Is Calico compliant with PCI/DSS requirements?"

PCI certification applies to the whole end-to-end system, of which
Calico would be a part. We understand that most current solutions use
VLANs, but after studying the PCI requirements documents, we believe
that Calico does meet those requirements and that nothing in the
documents *mandates* the use of VLANs.

## "How does Calico maintain saved state?"

State is saved in a few places in a Calico deployment, depending on
whether it's global or local state.

Local state is state that belongs on a single compute host, associated
with a single running Felix instance (things like kernel routes, tap
devices etc.). Local state is entirely stored by the Linux kernel on the
host, with Felix storing it only as a temporary mirror. This makes Felix
effectively stateless, with the kernel acting as a backing data store on
one side and etcd as a data source on the other.

If Felix is restarted, it learns current local state by interrogating
the kernel at start up. It then reads from `etcd` all the local state
which it should have, and updates the kernel to match. This approach has
strong resiliency benefits, in that if Felix restarts you don't suddenly
lose access to your VMs or containers. As long as the Linux kernel is
running, you've still got full functionality.

The bulk of global state is mastered in whatever component hosts the
plugin.

-   In the case of OpenStack, this means a Neutron database. Our
    OpenStack plugin (more strictly a Neutron ML2 driver) queries the
    Neutron database to find out state about the entire deployment. That
    state is then reflected to `etcd` and so to Felix.
-   In certain cases, `etcd` itself contains the master copy of
    the data. This is because some Docker deployments have an `etcd`
    cluster that has the required resiliency characteristics, used to
    store all system configuration -and so `etcd` is configured so as to
    be a suitable store for critical data.
-   In other orchestration systems, it may be stored in distributed
    databases, either owned directly by the plugin or by the
    orchestrator itself.

The only other state storage in a Calico network is in the BGP sessions,
which approximate a distributed database of routes. This BGP state is
simply a replicated copy of the per-host routes configured by Felix
based on the global state provided by the orchestrator.

This makes the Calico design very simple, because we store very little
state. All of our components can be shutdown and restarted without risk,
because they resynchronize state as necessary. This makes modelling
their behaviour extremely simple, reducing the complexity of bugs.

## "I heard Calico is suggesting layer 2: I thought you were layer 3! What's happening?"

It's important to distinguish what Calico provides to the workloads
hosted in a data center (a purely layer 3 network) with what the Calico
project *recommends* operators use to build their underlying network
fabric.

Calico's core principle is that *applications* and *workloads*
overwhelmingly need only IP connectivity to communicate. For this reason
we build an IP-forwarded network to connect the tenant applications and
workloads to each other, and the broader world.

However, the underlying physical fabric obviously needs to be set up
too. Here, Calico has discussed how both a layer 2 (see
[here]({{site.baseurl}}/{{page.version}}/reference/private-cloud/l2-interconnect-fabric)) 
or a layer 3 (see 
[here]({{site.baseurl}}/{{page.version}}/reference/private-cloud/l3-interconnect-fabric)) 
fabric
could be integrated with Calico. This is one of the great strengths of
the Calico model: it allows the infrastructure to be decoupled from what
we show to the tenant applications and workloads.

We have some thoughts on different interconnect approaches (as noted
above), but just because we say that there are layer 2 and layer 3 ways
of building the fabric, and that those decisions may have an impact on
route scale, does not mean that Calico is "going back to Ethernet" or
that we're recommending layer 2 for tenant applications. In all cases we
forward on IP packets, no matter what architecture is used to build the
fabric.

## "I need to use hard-coded private IP addresses: how do I do that?"

While this isn't supported today, this is on our roadmap using a
stateless variant of RFC 6877 (464-XLAT). For more detail, see
[this document]({{site.baseurl}}/{{page.version}}/reference/advanced/overlap-ips).

## "How do I control policy/connectivity without virtual/physical firewalls?"

Calico provides an extremely rich security policy model, detailed 
[here]({{site.baseurl}}/{{page.version}}/reference/security-model). 
This model applies the policy at the first and last hop
of the routed traffic within the Calico network (the source and
destination compute hosts).

This model is substantially more robust to failure than a centralised
firewall-based model. In particular, the Calico approach has no
single-point-of-failure: if the device enforcing the firewall has failed
then so has one of the workloads involved in the traffic (because the
firewall is enforced by the compute host).

This model is also extremely amenable to scaling out. Because we have a
central repository of policy configuration, but apply it at the edges of
the network (the hosts) where it is needed, we automatically ensure that
the rules match the topology of the data center. This allows easy
scaling out, and gives us all the advantages of a single firewall (one
place to manage the rules), but none of the disadvantages (single points
of failure, state sharing, hairpinning of traffic, etc.).

Lastly, we decouple the reachability of nodes and the policy applied to
them. We use BGP to distribute the topology of the network, telling
every node how to get to every endpoint in case two endpoints need to
communicate. We use policy to decide *if* those two nodes should
communicate, and if so, how. If policy changes and two endpoints should
now communicate, where before they shouldn’t have, all we have to do is
update policy: the reachability information does not change. If later
they should be denied the ability to communicate, the policy is updated
again, and again the reachability doesn’t have to change.

## "How does Calico interact with the Neutron API?"

[This document]({{site.baseurl}}/{{page.version}}/reference/advanced/calico-neutron-api) 
document goes into extensive detail about how
various Neutron API calls translate into Calico actions.

## Can a guest container have multiple networked IP addresses?
Yes. You can add IP addresses using the `calicoctl container <CONTAINER> ip (add|remove) <IP>` command.

## Why isn't the `-p` flag on `docker run` working as expected?
The `-p` flag tells Docker to set up port mapping to connect a port on the
Docker host to a port on your container via the `docker0` bridge.

If a host's containers are connected to the `docker0` bridge interface, Calico
would be unable to enforce security rules between workloads on the same host;
all containers on the bridge would be able to communicate with one other.

You can securely configure port mapping by following our [guide on Exposing
Container Ports to the Internet]({{site.baseurl}}/{{page.version}}/usage/external-connectivity).

## Can Calico containers use any IP address within a pool, even subnet network/broadcast addresses?

Yes!  Calico is fully routed, so all IP address within a Calico pool are usable as
private IP addresses to assign to a workload.  This means addresses commonly
reserved in a L2 subnet, such as IPv4 addresses ending in .0 or .255, are perfectly
okay to use.

## How do I get network traffic into and out of my Calico cluster?
The recommended way to get traffic to/from your Calico network is by peering to
your existing data center L3 routers using BGP and by assigning globally
routable IPs (public IPs) to containers that need to be accessed from the internet.
This allows incoming traffic to be routed directly to your containers without the
need for NAT.  This flat L3 approach delivers exceptional network scalability
and performance.

A common scenario is for your container hosts to be on their own
isolated layer 2 network, like a rack in your server room or an entire data
center.  Access to that network is via a router, which also is the default
router for all the container hosts.

If this describes your infrastructure, the
[External Connectivity tutorial]({{site.baseurl}}/{{page.version}}/usage/external-connectivity) explains in more detail
what to do. Otherwise, if you have a layer 3 (IP) fabric, then there are
detailed datacenter networking recommendations given
in the main [this article]({{site.baseurl}}/{{page.version}}/reference/private-cloud/l3-interconnect-fabric).
We'd also encourage you to [get in touch](http://www.projectcalico.org/contact/)
to discuss your environment.

### How can I enable NAT for outgoing traffic from containers with private IP addresses?
If you want to allow containers with private IP addresses to be able to access the
internet then you can use your data center's existing outbound NAT capabilities
(typically provided by the data center's border routers).

Alternatively you can use Calico's built in outbound NAT capability by enabling it on any
Calico IP pool. In this case Calico will perform outbound NAT locally on the compute
node on which each container is hosted.
```
./calicoctl pool add <CIDR> --nat-outgoing
```
Where `<CIDR>` is the CIDR of your IP pool, for example `192.168.0.0/16`.

Remember: the security profile for the container will need to allow traffic to the
internet as well. You can read about how to configure security profiles in the
[Advanced Network Policy tutorial]({{site.baseurl}}/{{page.version}}/usage/configuration/advanced-network-policy).

### How can I enable NAT for incoming traffic to containers with private IP addresses?
As discussed, the recommended way to get traffic to containers that
need to be accessed from the internet is to give them public IP addresses and
to configure Calico to peer with the data center's existing L3 routers.

In cases where this is not possible then you can configure incoming NAT
(also known as DNAT) on your data centers existing border routers. Alternatively
you can configure incoming NAT with port mapping on the host on which the container
is running on.
```
iptables -A PREROUTING -t nat -i eth0 -p tcp --dport <EXPOSED_PORT> -j DNAT  --to <CALICO_IP>:<SERVICE_PORT>
```
For example, you have a container to which you've assigned the CALICO_IP of 192.168.7.4, and you have NGINX running on port 80 inside the container. If you want to expose this service on port 80, then you could run the following command:
```
iptables -A PREROUTING -t nat -i eth0 -p tcp --dport 80 -j DNAT  --to 172.168.7.4:80
```
The command will need to be run each time the host is restarted.

Remember: the security profile for the container will need to allow traffic to the exposed port as well.  You can read about how to configure security profiles in the [Advanced Network Policy]({{site.baseurl}}/{{page.version}}/usage/configuration/advanced-network-policy) guide.

### Can I run Calico in a public cloud environment?
Yes.  If you are running in a public cloud that doesn't allow either L3 peering or L2 connectivity between Calico hosts then you can specify the `--ipip` flag your Calico IP pool:

```shell
./calicoctl pool add <CIDR> --ipip --nat-outgoing
```
Calico will then route traffic between Calico hosts using IP in IP.

In AWS, you disable `Source/Dest. Check` instead of using IP in IP as long as all your instances are in the same subnet of your VPC.  This will provide the best performance.  You can disable this with the CLI, or right click the instance in the EC2 console, and `Change Source/Dest. Check` from the `Networking` submenu.

```shell
aws ec2 modify-instance-attribute --instance-id <INSTANCE_ID> --source-dest-check "{\"Value\": false}"
...
./calicoctl pool add <CIDR> --nat-outgoing
```