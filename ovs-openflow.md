## Network Namespaces connected via OVS Bridge

### Create network namespaces with veth pairs
```
sudo ip netns add vm1
sudo ip link add veth-vm1 type veth peer name veth-vm1-ns
sudo ip link set veth-vm1-ns netns vm1
sudo ip netns exec vm1 ip addr add 10.0.0.1/24 dev veth-vm1-ns
sudo ip netns exec vm1 ip link set veth-vm1-ns up
sudo ip netns exec vm1 ip link set lo up
sudo ip link set veth-vm1 up

sudo ip netns add vm2
sudo ip link add veth-vm2 type veth peer name veth-vm2-ns
sudo ip link set veth-vm2-ns netns vm2
sudo ip netns exec vm2 ip addr add 10.0.0.2/24 dev veth-vm2-ns
sudo ip netns exec vm2 ip link set veth-vm2-ns up
sudo ip netns exec vm2 ip link set lo up
sudo ip link set veth-vm2 up
```

### Create OVS Bridge with ports linked to veth pairs

```
sudo ovs-vsctl add-br br0
sudo ovs-vsctl add-port br0 veth-vm1
sudo ovs-vsctl add-port br0 veth-vm2
```

Pings should work

vm1 -> vm2
```
sudo ip netns exec vm1 ping 10.0.0.2
```
Results in
```
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.528 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.069 ms
```
vm2 -> vm1
```
sudo ip netns exec vm2 ping 10.0.0.1
```
Results in
```
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.563 ms
64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=0.068 ms
```

### Clear flows

Notice there is already a flow:
```
sudo ovs-ofctl dump-flows br0
```
outputs
```
 cookie=0x0, duration=358.531s, table=0, n_packets=46, n_bytes=3524, priority=0 actions=NORMAL
```

Delete all flows on br0
```
sudo ovs-ofctl del-flows br0
```

And after dumping the flows again, that NORMAL flow is now gone
```
sudo ovs-vsctl dump-flows br0
```

Ping do not work now
```
sudo ip netns exec vm1 ping 10.0.0.2
```
output
```
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
From 10.0.0.1 icmp_seq=1 Destination Host Unreachable
```

The switch has no flows, so everything gets dropped.  We need to add some.

### Define custom flows

## ARP Reply

tcpdump while pinging vm1 to vm2 will show this
```
sudo tcpdump -i veth-vm1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth-vm1, link-type EN10MB (Ethernet), capture size 262144 bytes
16:33:13.377518 ARP, Request who-has 10.0.0.2 tell 10.0.0.1, length 28
16:33:14.380032 ARP, Request who-has 10.0.0.2 tell 10.0.0.1, length 28
16:33:15.404073 ARP, Request who-has 10.0.0.2 tell 10.0.0.1, length 28
```

The first thing we want to do is get ARP to work.

We're going to have the switch do the ARP reply with a fake MAC.

```
sudo ovs-ofctl add-flow br0 "priority=99,arp,in_port="veth-vm1" actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:00:11:22:33:44:55,load:0x2->NXM_OF_ARP_OP[],move:NXM_OF_ARP_TPA[]->NXM_OF_ARP_SPA[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0x1122334455->NXM_NX_ARP_SHA[],IN_PORT"
```

The match section
```
priority=99,arp,in_port="veth-vm1"
```
Matches any ARP traffic on port veth-vm1

The action section contains many actions, lets break it down:
```
move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[]
```
Sets the Ethernet destination to the Ethernet source

```
mod_dl_src:00:11:22:33:44:55
```
Assigns a fake mac address to the ethernet source

```
load:0x2->NXM_OF_ARP_OP[]
```
Sets the ARP OP code to be a ARP Reply

```
move:NXM_OF_ARP_TPA[]->NXM_OF_ARP_SPA[]
```
Sets the ARP source IPv4 address to the ARP target IPv4 address

```
move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[]
```
Sets the ARP Target Ethernet Address to the ARP Source Ethernet Address

```
move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[]
```
Sets the ARP Target IPv4 Address to the ARP Source IPv4 Address

```
load:0x1122334455->NXM_NX_ARP_SHA[]
```
Sets the ARP Source Ethernet Address to the fake MAC

```
IN_PORT
```
Send new data out on the port it came in on

Now we can do the same thing for namespace vm2
```
sudo ovs-ofctl add-flow br0 "priority=99,arp,in_port="veth-vm2" actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:00:11:22:33:44:55,load:0x2->NXM_OF_ARP_OP[],move:NXM_OF_ARP_TPA[]->NXM_OF_ARP_SPA[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0x1122334455->NXM_NX_ARP_SHA[],IN_PORT"
```

Now tcpdump while pinging vm1 to vm2
```
sudo tcpdump -i veth-vm1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth-vm1, link-type EN10MB (Ethernet), capture size 262144 bytes
17:01:25.807289 ARP, Request who-has 10.0.0.2 tell 10.0.0.1, length 28
17:01:25.807610 ARP, Reply 10.0.0.2 is-at 00:11:22:33:44:55 (oui Unknown), length 28
17:01:25.807624 IP 10.0.0.1 > 10.0.0.2: ICMP echo request, id 1776, seq 1, length 64
17:01:26.828286 IP 10.0.0.1 > 10.0.0.2: ICMP echo request, id 1776, seq 2, length 64
```

## Send IP data to correct port
Now that ARP works, the client will now know what IP to send data to.
We need to tell the switch where to send this data for each IP.
```
sudo ovs-ofctl add-flow br0 "priority=99,ip,nw_dst=10.0.0.2 actions=mod_dl_dst:9a:be:b7:13:fe:d6,output:veth-vm2"
sudo ovs-ofctl add-flow br0 "priority=99,ip,nw_dst=10.0.0.1 actions=mod_dl_dst:c6:d3:0f:db:79:db,output:veth-vm1"
```

The first rule matches any ip traffic with the destination of vm2 and sets the ethernet destination address to that of the MAC address of the link in namespace vm2. And sends the traffic out on port veth-vm2.

The second rule is just the same thing for vm1.

Now pings should work!