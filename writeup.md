# EVPN Configuration Guide

## Leaf 1 Underlay Explanation

```
int Et11
 description spine1
 no switchport
 ip add 10.0.1.1/31
 mtu 9214

int Et12
 description spine2
 no switchport
 ip add 10.0.2.1/31
 mtu 9214
```

These represent the routed links between this leaf and the spines. We have two spines, and each of these interfaces corresponds to another interface on the spine (Et11 == Spine 1: Et1, Et12 = Spine2: Et1). There are four interfaces in each spine for the four leaves and two interfaces in each leaf for the two spines. The `no switchport` command turns it into a routed link. The address assignment basically started at 10.0.1.0 for spine1-et1 and then incremented so every even number is in the spine and every odd number is in the leaf (but this is arbitrary). This is basically only used in the underlay routing and the actual address is not critically important.

```
interface Loopback0
 ip add 10.0.250.11/32
```

This is also used as the router ID in the BGP process in each switch. The only reason we are adding it here is to couple it with the router ID, but this has nothing to do with the actual underlay except that we will advertise it as reachable from this host in the eBGP control plane. What this will be used for later is to peer with for the EVPN family in the same way the above physical interface addresses are used for peering in the IPv4 unicast family.

```
ip routing
```

This just allows us to use the `router bgp` command below to start routing.

```
router bgp 65001
 router-id 10.0.250.11
 no bgp default ipv4-unicast
 distance bgp 20 200 200
```

We somewhat arbitrarily selected 65001 from the available private AS space that IANA provided for us and is recorded in RFC 6996. Having it correspond to our naming convention a little bit will make mistakes a bit harder. To enter into configuration for our BGP routing instance denoted by the AS 65001 we run the first command. The second command sets the router ID. While you technically could pick any unique RID across the space, you would then have to manage two underlay-scoped unique number sets and, based on experience, managing one is hard enough.

The `no bgp default ipv4-unicast` is critically important (and not mandatory). If we don't use this command all the neighbors by default will be in the IPv4-unicast family. Since we are actually utilizing MP-BGP and the EVPN address family, this default could mess us up if we forget to specify the address family of the neighbor. It essentially forces us to explicitly declare neighbors as IPv4 unicast or EVPN.

`distance bgp 20 200 200` might look familiar to people with minimal EVPN experience. It sets the administrative distance to defaults, but prefers eBGP over local or internal BGP routes.

```
router bgp 65001
 neighbor underlay-spine peer group
 neighbor underlay-spine remote-as 65000
 neighbor underlay-spine maximum-routes 12000 warning-only
 neighbor 10.0.1.0 peer group underlay-spine
 neighbor 10.0.2.0 peer group underlay-spine
```

Here we again enter into the BGP instance configuration and create the underlay peer group. This peer group is given the remote-as of the two spine routers and a `maximum-routes` of 12000. There are lots of fun stories of internet routers that didn't set this number with disastrous results (see AS 7007 incident).

After this we add the two spines as a part of the peer group using their portion of the /31 point-to-point address from earlier.

```
router bgp 65001
 address-family ipv4
  neighbor underlay-spine activate
  network 10.0.250.11/32
  maximum-paths 4 ecmp 64
```

Now we actually get to activate BGP and see some stuff up and running! In the BGP instance configuration mode we enter into a new configuration mode, the IPv4 address family. Our underlay is based upon this address family, and we see that in our activation of the leaf1-spine1 and leaf1-spine2 peerings by running the activate command on the whole peer group. We start advertising our loopback IP, which is how other EVPN sessions will know where to send control plane information. We also set the `maximum-paths 4 ecmp 64` based on our network topology. We know that there will (hopefully) never be more than four equal cost paths to a host through the underlay, so we can set this limit to avoid errors. `ecmp 64` is the maximum number of ECMP paths for each route.

## Spine 1 Underlay Explanation

```
int Et1
 desc leaf1
 no switchport
 ip add 10.0.1.0/31
 mtu 9214
int Et2
 desc leaf2
 no switchport
 ip add 10.0.1.2/31
 mtu 9214
int Et3
 desc leaf3
 no switchport
 ip add 10.0.1.4/31
 mtu 9214
int Et4
 desc leaf4
 no switchport
 ip add 10.0.1.6/31
 mtu 9214

interface Loopback0
 ip add 10.0.250.1/32

ip routing

router bgp 65000
 router-id 10.0.250.1
 no bgp default ipv4-unicast
 distance bgp 20 200 200

router bgp 65000
 neighbor 10.0.1.1 remote-as 65001
 neighbor 10.0.1.3 remote-as 65002
 neighbor 10.0.1.5 remote-as 65003
 neighbor 10.0.1.7 remote-as 65004

router bgp 65000
 address-family ipv4
  neighbor 10.0.1.1 activate
  neighbor 10.0.1.3 activate
  neighbor 10.0.1.5 activate
  neighbor 10.0.1.7 activate
  network 10.0.250.1/32
  maximum-paths 4 ecmp 64
```

The underlay configuration for the spine node is nearly identical to the leaf node. This is expected, since every node is functioning almost identically in the underlay (we are just doing routing). The only real differences are more connections, since we are downlinking to the four leaves, and different IP values (almost by definition).

## Underlay Diagram

![Underlay Diagram](pictures/evpn(1)-ebgp%20underlay.webp)

## Leaf 1 EVPN Control Plane Explanation

```
service routing protocols model multi-agent
```

This simply allows us to use EVPN.

```
router bgp 65001
 neighbor evpn peer group
 neighbor evpn remote-as 65000
 neighbor evpn update-source Loopback0
 neighbor evpn ebgp-multihop 3
 neighbor evpn send-community extended
 neighbor evpn maximum-routes 12000 warning-only
 neighbor 10.0.250.1 peer group evpn
 neighbor 10.0.250.2 peer group evpn
 !
 address-family evpn
  neighbor evpn activate
```

This section looks familiar and that is both the gift and the curse of using MP-BGP for our network. The gift being that the technologies are similar and the commands are as well. The curse is that you have to be mindful to mentally separate the BGP IPv4 address family that is doing the underlay routing of the encapsulated packets, and this address family which decides which underlay address a packet should go to given either its destination IP (for routed L3VNIs) or its destination MAC (for L2 VNIs).

Once we enter into the BGP configuration, we follow nearly the same procedure as when we created our first peer group, but for our self-named "evpn" peer group and then we activate the peer group in the EVPN address family (note: the address-family is what actually does the work here, the peer group name is arbitrary).

The three differences are noted here:

**`neighbor evpn update-source Loopback0`**: This is a key part of the EVPN BGP config that differentiates it from our underlay BGP. In our underlay we created a routed link on each of our connected interfaces. In the leaf1 example this was 10.0.1.1 to the spine with 10.0.1.0 and the neighbor used that IP to peer with it. The peering was from physical interface to physical interface. The source and destination IP address of packets sent between these peers naturally matched this. If we sniff the interface we will see the control plane information being exchanged with these two IP addresses, but we need a different IP now because we are starting a whole separate BGP session to do our control plane info. This is why we often see this described as loopback to loopback.

It is a little confusing because we are just sending control plane information between these IPs, meaning you will never send a packet to a machine and see it use one of these IP addresses. It is just here so the nodes can advertise the fun route types in the EVPN address family.

**`neighbor evpn ebgp-multihop 3`**: This maxes out the amount of hops we can use through the underlay. Essentially the TTL.

**`neighbor evpn send-community extended`**: Extended communities carry a ton of stuff so we need to enable that. Route targets are notably one of those things.

```
interface Loopback1
 ip add 10.0.255.11/32
!
router bgp 65001
 address-family ipv4
  network 10.0.255.11/32
!
int vxlan1
  vxlan source-interface Loopback1
```

I mentioned before with Loopback0 that it can be confusing, since we are just sending control plane information. Loopback1 is the resolution of that confusion because it is what we actually use for the data plane! When I want to send a packet, EVPN will look up where it should go based on the EVPN control plane information, and get a target VTEP address (Loopback1) and then send the packet with that destination IP and the source IP of our own Loopback1. We see below the creation that we are actually advertising that we have that network to the eBGP underlay, which means when I send a packet to another VTEP the eBGP underlay, I can just use the simple routed links between nodes as the next hop to reach the other VTEP.

Be very mindful to separate this in your head (in fact they could be completely different protocols): the EVPN address family is Loopback0→Loopback0 and just sends information about what VTEPs own what so we can select a target VTEP. Once we have this target selected it is all just IPv4 routing with next hops that you are familiar with in eBGP.

After the loopback we are creating the VXLAN interface so our EVPN control plane knows what to use and give it Loopback1 as discussed above.

## Spine 1 EVPN Control Plane Explanation

```
service routing protocols model multi-agent

router bgp 65000
 neighbor evpn peer group
 neighbor evpn next-hop-unchanged 
 neighbor evpn update-source Loopback0
 neighbor evpn ebgp-multihop 3
 neighbor evpn send-community extended
 neighbor evpn maximum-routes 12000 warning-only
 neighbor 10.0.250.11 peer group evpn
 neighbor 10.0.250.11 remote-as 65001
 neighbor 10.0.250.12 peer group evpn
 neighbor 10.0.250.12 remote-as 65002
 neighbor 10.0.250.13 peer group evpn
 neighbor 10.0.250.13 remote-as 65003
 neighbor 10.0.250.14 peer group evpn
 neighbor 10.0.250.14 remote-as 65004
 !
 address-family evpn
  neighbor evpn activate
```

Again, this is very similar to the leaf configuration, but also a point where we can get really confused. Since we are doing control plane exchanges between all the VTEPs and the routing between them for the control plane information happens over eBGP, why are we peering with the spines for the EVPN family? The answer is we are not really peering with them. In fact if we replaced the peering information for an all-to-all relationship in the leaves (that is, every leaf peers with every leaf using the Loopback0 IP) we would achieve the exact same effect. Those that have taken graph theory may be able to figure out why we do this. The peer count would be the same as the number of edges in a complete graph, meaning:

$$E = \binom{n}{2} = \frac{n(n-1)}{2}$$

To prevent this we use the spines as an aggregation layer for the various routes we will be sending, which then passes it along to the VTEPs that actually use them. In fancy terms this is called a route reflector. The `neighbor evpn next-hop-unchanged` will help with this role so all VTEPs will receive packets with the source IP of the other VTEP.

## Control Plane Diagram

![Control Plane Diagram](pictures/evpn(1)-evpn%20control%20plane.webp)

## Leaf 1 VTEP Usage Explanation

```
vrf instance gold
!
ip routing vrf gold
!
int vxlan1
  vxlan vrf gold vni 100001
!
router bgp 65001
 vrf gold
    rd 10.0.250.11:1
    route-target export evpn 1:100001
    route-target import evpn 1:100001
    redistribute connected
```

Now we have a working VTEP and everything is great, but we need to actually put this thing to use. We will do this through symmetric routing and a L3VNI. This means our L2 connectivity to each host will end at the SVI, where it will be sent over VNI 100001, which is mapped to our gold VRF. Above the first three exclamation marks we put this into play, creating the VRF and mapping it. Then we go into the BGP config to add the VRF. Here is where things start to get spicy.

**`rd 10.0.250.11:1`**: The best explanation for route descriptors I have heard is a namespace. It doesn't actually do anything logic-wise, but it does make the "network layer reachability information" globally unique. Because we want multi-tenancy, we need to allow multiple of the same MAC addresses or multiple prefixes in the control plane. RDs allow us to do this by giving each VNI a namespace for these type 2 and type 5 routes a unique value. We also add the loopback IP so we can have the same MAC address under multiple VTEPs. In practice we never do this, but it allows us to handle clients moving in a more graceful way by using the MAC extended community and incrementing the value there so we know what is the more recent entry. The naming convention we use is completely arbitrary, but as we mentioned before maintaining multiple globally unique values is hard.

```
route-target export evpn 1:100001
route-target import evpn 1:100001
```

While the RD tags routes to make them globally unique per VTEP and per VRF, the RT similarly tags routes, but it is actually used for logic. Whenever our VTEP learns something in our VRF it will add this RT to the information, and then all other VTEPs with the same RT import rule will accept the route as a part of its VTEP. You can think of this as the VNI for the control plane. While in the data plane we add a VXLAN header with the VNI to identify what the packets belong to, in the control plane we add the RT to delineate what the routes go to. Messing up these values can lead to some funky errors.

**`redistribute connected`**: For our loopbacks we explicitly advertised it into the eBGP control plane. We could also manually do this for the EVPN control plane, but we want it to dynamically learn about all the connected routes and tell the other VTEPs. Once I add this command if I start adding SVIs with IP addresses they will have reachability from everywhere.

```
vlan 11
int vlan 11
 vrf gold
 ip address 10.11.11.1/24
```

Here is a simple SVI that we will attach host1 to in order to test reachability to host2 on vlan 12 in leaf2.

## Host Connection 

Once we add the two interfaces into the correct vlans we can configure the hosts to ping each other.

### Host 1 Connection
```bash
apt update && apt install -y iproute2 iputils-ping
ip addr add 10.11.11.2/24 dev eth1
ip link set eth1 up
ip route add 10.12.12.0/24 via 10.11.11.1
```

### Host 2 Connection
```bash
apt update && apt install -y iproute2 iputils-ping
ip addr add 10.12.12.2/24 dev eth1
ip link set eth1 up
ip route add 10.11.11.0/24 via 10.12.12.1
```

## VTEP Usage Diagram

![VTEP Usage Diagram](pictures/evpn(1)-VTEPs.webp)