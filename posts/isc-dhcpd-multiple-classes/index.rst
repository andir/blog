.. title: Using multiple client classes with ISC DHCPd
.. slug: isc-dhcpd-multiple-classes
.. date: 2016-05-23 11:00:00 UTC
.. tags: dhcpd, isc
.. category: linux
.. link:
.. description:
.. type: text

Since the internet is lacking examples of how to use multiple classes with a single pool here is one:

.. code:: nginx

    class "mac-filtered-clients"
    {
        match binary-to-ascii (16, 8, ":", substring(hardware, 1, 6)); 
    }

    subclass "mac-filtered-clients" "50:7b:00:00:00:00"; # some cool host!

    class "J-client" {
        spawn with option agent.circuit-id;
        match if (substring(option agent.circuit-id, 30, 9) = "foo-bar");
        lease limit 1;
    }

    subnet 192.168.0.0 255.255.0.0 {
         pool {
              range 192.168.0.10 192.168.0.150;
              allow members of "J-client";
              allow members of "mac-filtered-clients";
         }
    }


This isn't very special compared to a setup with just a single class but it can be confusing since debugging classes is a pita.. One pitfall I did run into was using the byte representation of the mac-addresses (without the quotes) and using :code:`match hardware;`. The example above works for me (tm). 
