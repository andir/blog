.. title: Juniper MCD decided to coredump on commit and rollback
.. slug: juniper-config-recovery-mcd-coredump
.. date: 2016-11-08 15:00:00 UTC
.. tags: juniper, config, recovery
.. category: networking
.. link:
.. description: Recover juniper configuration when rollback and delete of invalid statements segfault mcd
.. type: text


On a Juniper EX3300 a colleague of mine entered an invalid statement:

.. code:: shell

    interface-range some_ports {
        member ge0/0/2;
        member-range ge-0/0/0 to ge-0/0/1;
    }
 

The `member ge0/0/2` is missing a `-` between `ge` and `0/0/2`. Juniper (for whatever reason) accepted the input but `mcd` decided to segfault when asked to delete or rollback the configuration.

.. code:: shell

    pid 75982 (mgd), uid 0: exited on signal 11 (core dumped)


Recovering from that case is actually not that hard. You just know the right command(s) ;-)

You can load an older configuration via the `load override <config file>` command. I did also try the `load replace <config file>` command but that also segfaulted..

.. code:: shell

   andi@foo# load override ?
   Possible completions:
     <filename>           Filename (URL, local, remote, or floppy)
     db/                  Last changed: Oct 27 10:44:35
     juniper.conf+.gz     Size: 6000, Last changed: Nov 08 15:12:55
     juniper.conf.1.gz    Size: 5913, Last changed: Jun 30 13:36:52
     juniper.conf.2.gz    Size: 5881, Last changed: Jun 30 12:59:25
     juniper.conf.3.gz    Size: 5280, Last changed: Jun 30 12:54:02
     [...]
   {master:0}[edit]
   andi@foo# load override juniper.conf+.gz    
   load complete 

In my case the `juniper.conf+.gz` was the desired config file. I recommend inspecting those files before loading them (they are stored in `/config/`).
