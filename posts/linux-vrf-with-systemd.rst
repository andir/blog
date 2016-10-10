.. title: Using VRFs with linux and systemd-networkd
.. slug: linux-ip-vrf-systemd-networkd
.. date: 2016-10-09 13:00:00 UTC
.. tags: linux, routing, vrf, iproute2, systemd
.. category: linux, network
.. link:
.. description: Using vrf interfaces with systemd-networkd 
.. type: text


While working on a systemd-networkd patch to implement (at least basic) VRF interfaces I did write :doc:`my other post <linux-ip-vrf>`. This post should give you a brief example on how you can create a VRF with systemd-networkd.

At this point it really only created the interfaces and enslaves potential customer interfaces to a given VRF.

You still have to implement all the `ip rule`-stuff yourself. For example a `systemd.unit` file might be the right approach which is executed/started after the network is "up".


First you've to create the systemd.netdev `vrf-customer1.netdev` file:

.. gist:: https://gist.github.com/andir/146803a9343e04fffabc8e7105dff3cd


After restarting `systemd-networkd` with `systemctl restart systemd-networkd` you should see the corresponding interface:

.. code:: shell
   
   $ ip -d link show dev vrf-customer1
   9: vrf-customer1: <NOARP,MASTER> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
       link/ether 02:74:c7:e1:de:64 brd ff:ff:ff:ff:ff:ff promiscuity 0 
       vrf table 42 addrgenmode eui64 numtxqueues 1 numrxqueues 1 


Note the last line which states `vrf table 42`.


To add an interface to the VRF you'll have to modify/create the corresponding .network file. This is how the file `/etc/systemd/network/enp0s31f6.network` would look on my notebook:

.. gist:: https://gist.github.com/andir/ee155492e3af2f83df39b0808fda5718

Restarting `systemd-networkd` again and checking the status using `ip -d link` gives us:

.. code:: shell

   $ip -d link show  dev enp0s31f6            
   3: enp0s31f6: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel master vrf-customer1 state DOWN mode DEFAULT group default qlen 1000
    link/ether 50:7b:9d:cf:34:dc brd ff:ff:ff:ff:ff:ff promiscuity 0 
    vrf_slave table 42 addrgenmode eui64 numtxqueues 1 numrxqueues 1 
 
Again note the last line which states `vrf_slave table 42`. Also in the first line you can see that it belongs to the VRF `vrf-customer`.


And that is all for now. You still have to add the `ip rule` commands in some way or another (there is no support in systemd-networkd yet and I've not had a good idea without inventing `ip rule` management in systemd).
