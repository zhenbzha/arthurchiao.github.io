---
layout    : post
title     : "Understanding Virtual Networking Models"
date      : 2019-11-30
lastupdate: 2019-11-30
categories: network
---

## Index

1. [Background Knowledge](#ch_1)
2. [Physical networking](#ch_2)
3. [Network virtualization](#ch_3)
4. [Virtual networking models](#ch_4)
5. [](#ch_5)

# 1. Introduction

There are so many the so called **virtual networking solutions**. If you are
related to this, have you wondered how these solutions come from?

In this post, I will share some of my understandings. This post will not cover
all the aspects (acutally the main purpose of this post is serving as the
background knowledge of my [another post]({% link _posts/2019-11-30-cracking-k8s-node-proxy.md %})).

Solutions come from pratical needs. So we start from a physical world - an age
when there were not any virtualization technologies.

# 2. Physical networking

In the old days before virtualization, all applications run directly on baremetal
servers. Requirements for networking solutions at this age including:

1. **connectivity**: hosts were reachable from each other so applications
   between different servers could communicate with each other
1. **security**: not covered in this post

## 2.1. L2 forwarding

A flat L2 network just meets the **connectivity** requirement.

If the hosts are in the same L2 network/domain,
then packets from host A to host B just need **L2 switching**: switches forward
packets (frames to be more accurate) solely based on `dst_mac`.

> TODO: add learning mechanism here.

<p align="center"><img src="/assets/img/understanding-virtual-networking-models/l2-forward.png" width="65%" height="65%"></p>
<p align="center"> Fig. Hosts connected via L2 forwarding</p>

## 2.2. L2 forwarding + L2 segmentation

Large L2 networks may cause broadcasting storm.

If network quality degrades because there are too many hosts or
too much traffic, we want seperate parts of node into distinct L2 domains.
Here comes L2 segmentation technology, the mostly used one is VLAN:

> TODO: add more about L2 segmentation.

<p align="center"><img src="/assets/img/understanding-virtual-networking-models/l2-segmentation.png" width="65%" height="65%"></p>
<p align="center"> Fig. L2 forwarding + L2 segmentation</p>

## 2.3. L3 routing

servers between different domains cannot reach each other without going
through some special purpose device

L2 networks must be physically directly-connected via cables. If we want to
connect two L2 networks, we need a router (L3 device): communication between two
L2 networks needs routing.

<p align="center"><img src="/assets/img/understanding-virtual-networking-models/l3-network.png" width="65%" height="65%"></p>
<p align="center"> Fig. L3 routing</p>

OK. That's all we need, we are able to:

* connect hosts
* isolate hosts for performance considerations

# 3. Network virtualization

Why we need network virtualization when we already solved the above mentioned
needs? The answer may be:

* resource utilization: applications run on host having a low CPU/memory/etc utilization
* flexibility: physical servers and networks are cummbersome, it usually takes
  monthes even years to deploy or scale up a data center. Can we achieve this in
  several hours or even minutes?
* mobility: if one server moves from one place to another, can I keep it's IP
  and MAC unchanged?
* many many other strong needs

With all these needs, we need virtualization:

* spawn virtual machines (VM) in physical servers (called **nodes**)
* assgin each VM distinct IP address and/or MAC address
* delopy applications in VM

In this way, let's see how we achieved the above goal:

1. resource utilization: with multiple VMs run on each node, utilization rate
   increases
1. flexibility: previously we need to purchase and deploy baremetal servers in
   months, now only need to boot VMs in hours or minutes
1. mobility: when previously we need to move a physical server from one place to
   another, now we only need to move a VM - all the underlying infrastructure
   stays unchanged.

So, what need the network engineers to do in order to support the above?
Network virtualization.

# 4. Virtual networking model

If we run containers (or VMs) inside a host, and assign distinct IP addresses to
those instances, how will those instances communicate with each other? The
answer is, just like the physical network, we add virtual networks inside each
host:

* intra-host instances communication: virtual L2 switch
    *  inside the host: if you think of each physical host as an entire
      data center, and each instance inside that physical host as a host, then
      this model exactly mimics the **L2 forwarding** model in section 1.2.1.
* inter-host (or cross-host) instances communication
    * direct routing
    * tunneling
    * others

## 3.1 Direct routing

The first fashion is: **making all the instances' IP addresses routable inside
entire network**. In other words, we **announce these IP addresses (or CIDRs) to
the entire network**, so both the physical routers/switches and hosts knows how
to route these IP addresses correctly:

<p align="center"><img src="/assets/img/understanding-virtual-networking-models/direct-routing.png" width="90%" height="90%"></p>
<p align="center"> Fig. Direct routing</p>

In real world, BGP is mostly used in this scenario. E.g. **Calico + Bird** for K8S.

## 3.2 Tunneling

The second way to achieve network connectivity for cross-host instances is:
encapsulate every instance packet into another packet, with the outer packet
using hosts' IPs. As host IP addresses are routable inside entire network, this
packet can reach any host. After the packet arrived the destination host, a
de-capsulation will remove the outer layer, and deliver the inner packet into
the correct instance on this host.

<p align="center"><img src="/assets/img/understanding-virtual-networking-models/tunneling.png" width="90%" height="90%"></p>
<p align="center"> Fig. Tunneling</p>

This fashion needs each host running a dedicated agent for
encapsulation/decapsulation, and synchronizing encapsulation/decapsulation info
(e.g. which instance IP is on which host) between all hosts. This method is
often called **tunneling**, or **overlay** (because we added another layer over host network).

Real world K8S networking solutions of this kind includs:

* Calico + VxLAN
* Flannel + VxLAN/IPIP/GRE

## 3.3 Direct forwarding

This fashion is not widely used in K8S, but in OpenStack. And I named this
teminology here (havn't seen this verb at elsewhere).

In some sense, this is one is much similar with **direct routing**, because it
also announces all the instances' connectivity information to the entire
network. The different part is:

* direct routing announces **routing information** (L3)
* direct forwarding announces **forwarding information** (L2)

In these scenarios, the network solution is often called **large L2 network**,
because:

* it is a L2 network (instances of the same L2 network can be forwardable within
  the entire physical network)
* it may be really large (10K+ nodes)

<p align="center"><img src="/assets/img/understanding-virtual-networking-models/direct-forwarding.png" width="90%" height="90%"></p>
<p align="center"> Fig. Direct forwarding</p>

If you are interested in more of this, you can refer to my previous talk
[Ctrip Network Architecture Evolution in the Cloud Computing Era](2019-04-17-ctrip-network-arch-evolution.md)
(or [Chinese version](2019-04-17-ctrip-network-arch-evolution-zh.md)).


## References

