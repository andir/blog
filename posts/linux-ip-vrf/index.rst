.. title: Using VRFs with linux
.. slug: linux-ip-vrf
.. date: 2016-06-14 17:00:00 UTC
.. tags: linux, routing, vrf, iproute2
.. category: linux
.. link:
.. description: Using vrf interfaces to seperate routing on a linux server
.. type: text

Ever since I've heard about VRF support (or VRF-lite like it is called in Documentation/network/vrf.txt) I wanted to start tinkering with it.
Since the topic is currently only covered in the previously mentioned linux kernel documentation I thought it would be a good idea to post some notes.


It basically boils down to adding an VRF interface and creating two `ip rule`-entries.

I'm using a local VM with ArchLinux since the VRF feature seems to require a rather recent kernel. My experience with kernels below version 4.6 weren't that great.


.. code:: shell

    $ ip -br link # this is where we are start off
    lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
    ens3             DOWN           52:54:00:12:34:56 <BROADCAST,MULTICAST>


Now are adding a new interface named `vrf-customer1` with the table `customer1` assigned to it.
The table parameter is used to place routes from your devices within the VRF into the right routing table


.. code:: shell

    $ ip link add vrf-customer1 type vrf table customer1

    $ ip -d link show vrf-customer1 # verify that the interface indeed exists and has the correct table assigned to it
    4: vrf-customer1: <NOARP,MASTER> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
        link/ether ca:22:59:ba:05:da brd ff:ff:ff:ff:ff:ff promiscuity 0
        vrf table 100 addrgenmode eui64

Next: redirect the traffic from and to the vrf to the customer1 table and verify the rules are indeed as expected:

.. code:: shell

    $ ip -4 rule add oif vrf-customer1 lookup customer1
    $ ip -4 rule add iif vrf-customer1 lookup customer1
    $ ip -6 rule add oif vrf-customer1 lookup customer1
    $ ip -6 rule add iif vrf-customer1 lookup customer1

    $ ip -4 rule
    0:	from all lookup local
    32764:	from all iif vrf-customer1 lookup customer1
    32765:	from all oif vrf-customer1 lookup customer1
    32766:	from all lookup main
    32767:	from all lookup default

    $ ip -6 rule
    0:	from all lookup local
    32764:	from all oif vrf-customer1 lookup customer1
    32765:	from all iif vrf-customer1 lookup customer1
    32766:	from all lookup main

To make any use of our VRF we will have to add an device to it. In my case I'll add the only available "physical" device `ens3`.

.. code:: shell

    $ ip link set ens3 master vrf-customer1
    $ # verify the interface is indeed a member of the VRF
    $ ip -br link show master vrf-customer1
    ens3             DOWN           52:54:00:12:34:56 <BROADCAST,MULTICAST>

Now that we've an interface to receive send send packets with it we should consider adding an IP-Address to it. Since IPv6 is enabled per default we don't need to configure a LL-Address for that protocol.


.. code:: shell

    $ # add an IP to the interface
    $ ip addr add 10.0.0.1/24 dev ens3
    $ ip route show table customer1
    local 10.0.0.1 dev ens3  proto kernel  scope host  src 10.0.0.1

Seeing a route like that might confuse the average linux user. Those routes usually exist within the local table which you can check via `ip route show table local`

The route to the /24 we've added is still missing from the interface. Why is that?
You'll have to change the state of the interface to "UP":

.. code:: shell

    $ ip link set ens3 up
    $ ip route show table customer1
    broadcast 10.0.0.0 dev ens3  proto kernel  scope link  src 10.0.0.1
    10.0.0.0/24 dev ens3  proto kernel  scope link  src 10.0.0.1
    local 10.0.0.1 dev ens3  proto kernel  scope host  src 10.0.0.1
    broadcast 10.0.0.255 dev ens3  proto kernel  scope link  src 10.0.0.1


    $ ip -6 route show table customer1
    local fe80::5054:ff:fe12:3456 dev lo  proto none  metric 0  pref medium
    fe80::/64 dev ens3  proto kernel  metric 256  pref medium
    ff00::/8 dev vrf-customer1  metric 256  pref medium
    ff00::/8 dev ens3  metric 256  pref medium


suddenly routes \\o/
