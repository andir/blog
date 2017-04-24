.. title: SaltStack, Ansible, Puppet, … how can we share data between servers?
.. slug: saltstack-ansible-puppet-how-can-we-share-data
.. date: 2017-04-22 16:50:00 UTC
.. tags: linux, ansible, saltstack, puppet, automation
.. category: linux
.. link:
.. description: How can we share data between servers in a secure fashion?

WARNING: This is still a draft somewhat.. 

Lately I've been depressed by the lack of automation in my life or really in
the projects I take part in.

Most of my stuff is not hosted on AWS, Azure, Digitalocean or other big "cloud"
players. That is due to multiple reasons:

 - We may have hardware, rack, power and network capacity available which is
   basically for free.
 - We might need physical connectivity to the machine - e.g. connecting to a
   local WiFi mesh network, a beamer, some audio equipment and so on…
 - We want to remain in control of our data. That means we are not going to
   trust some random internet business.
 - It is cheaper, even if you've to pay for the servers that are running 24/7.
   This might not be true for a website that sells boat trips on the Atlantic
   Ocean and gets high traffic spikes every 2nd Sunday from April to December
   but for our scenarios it works quite well.


At the office I've been using puppet more then six years and I'm quite happy
with it. In the local Freifunk-Community (https://darmstadt.freifunk.net) we
are using SaltStack. For my personal infrastructure (E-Mail, Webhosting, random
docker containers, pastebin, dn42, …) I'm using Ansible (https://ansible.com).

Neither of those deployments are very trivial, they are probably just the
average project for either of those tools.

A few weeks back I and a few others started writing SaltStack states for an
IRC-Network that we are running. If you think about automating the deployment
of IRCd's it is rather easier. Create a VM, install your favorite flavor of
linux, install build dependencies for the IRCd of your choice and
generate the configuration. That's basically it.

We've been using TLS in our network for about 10 years now. (Both user and
server2server communication). Since it wasn't and still is not desirable to buy
wildcard certificates or share certificates between servers we are running our
own CA (including OCSP service etc.). With the switch to automated server
configuration we also want to deploy LetsEncrypt - actually LetsEncrypt
encouraged us to start using some kind of automation since humans aren't
reliable and having to jupe a server every 3 months because the admin still
didn't configure a LetsEncrypt cronjob is not what we are after.

So I started out writing some states and a Vagrantfile
(https://gist.github.com/andir/2188da38904557a58cd7df29fd277275) for easier
testing of the whole stack. After tackling a few issues with Vagrant and
different `boxes` (as vagrant calls VM images) everything looked fine.

We basically have a zoo of VMs on our laptops wich simulate the production
network. At least so we thought.

For the links between the IRCDs we are using TLS. So far we did exchange
fingerprints, manually add them to the `ircd.conf`, `/quote rehash` and off you
go.

When you automate the deployment you also automate the configuration of links
between your servers. While you can configure `send_password` and
`receive_password` since what I do know when that isn't what we want to rely on
when sending chat messages around the globe. So we still need to exchange
certificate fingerprints. One might say that `TLSA` records are made for that
but I wonder why I can't use the already verified connection between my Salt
Master and the Salt Minion. If you query DNS for TLSA records it requires your
DNS to be up, DNSSEC to work, your local resolver to have a copy of the root
dns zone or an upstream resolver that (hopefully) verifies DNSSEC - unlike
OpenDNS.

Since we are also running our own DNS Servers adding them as a dependency isn't
really a great idea. We might be recovering from the death of some parts of the
infrastructure. The only reasonable approach to me is to use the authenticated
relationship from minion and master to exchange public keys, fingerprints etc.

While I'm currently focusing on exchanging certificate fingerprints and/or
public keys this applies to a bunch of scenarios you might come acress when you
decide to drop all shared secrets and generate them on your servers. A few of
those could be: 

 - Generating and exchanging TSIG keys for nsupdate between your DNS Servers
   and minions.
 - Communicating a shared secret for database access, borg backups, …
 - Distributing SSHFPs between your servers (local verification, publishing
   them to DNS as SSHFP records,…)
 - Trust bootstrapping (after minion key verification) for any kind of internal
   secret database

With SaltStack you've a few options to transfer data from a minion to the master / other minions:

 - use the Mine to export data
 - use grains
 - use pillars (with pre-generated certificates, keys, …) -> not the desired solution
 - trigger events with custom context
 - use a vault

In puppet you could use exported ressources, in Ansible you could write to some
kind of custom api without much overhead or query/configure the other side
on-demand since a single run isn't restricted to a single server.

Puppet has solved the issue for many years - people tend to not like puppet for
various reasons. Maybe because it feels like programming?


Using the Salt Mine
====================

The Salt Mine is a handy construct when it comes to exporting data from a node
as long as that data only exists as some kind of list or with a very low
cardinality. Also it only makes sense if the data is mostly static and doesn't
change. Changes to the exported data (e.g. another information, removal of
information, …) require changes to the minion configuration. This isn't really
practical if you are trying to write modular states where not everything is
hard-wired with another state.

For reference this is how you would export information about a certificate:

.. code:: yaml

   mine_functions:
     my_awesome_certificate:
       - mine_function: tls.cert_info
       - /etc/letsencrypt/live/.../cert.pem


This has to be deployed on each minion and you probably also have to restart the minion when changeing the file.

More information on the salt mine can be found at https://docs.saltstack.com/en/latest/topics/mine/.

Trying to export data via Grains
================================

I'm not going to get into the details or provide a example configuration here.
I've written a bunch of grains and they just work. The limitation is that you
can't really attribute them. So you can't tell your custom grains what to
export without changing the code or introducing custom configuration files,
adding stuff to the minion config etc.. While this approach would automatically
propagate new certificate fingerprints, public keys, … you still have to use
the Salt Mine to actually export them.  Which isn't that bad.

The lack of configuration options for custom grains kills this approach for me.


Pillars
=======

LOLNOPE. Using pillars would require manual collection of fingerprints or (even
worse) central management of all the certificates.

This simply doesn't work for me.

Using a custom event
====================

This is what seems to be a promising but ephemeral approach to the issue.

The basic idea is:

 - Create the public-/private key pair on the minion.
 - On changes to that fail (creation, key rollover, new certificate, …) execute
   a script with the filename as argument
 - The (bash) script then extracts the information we want from the file.
   (using `openssl` or other command line tools) and publishes those
   information using something like
   
   .. code:: bash

       `salt-call event.send my/custom/event/certificate-changed '{"certfp": "ABCDEF01234567890…"}'`

In order to use the "published" fingerprint other minions must be up, running
and listening to those events. Otherwise the information is lost and nobody
cares.

This approach only works if we figure out how to store the fingerprints - after
receiving an event.

It basically sucks as bad as the others but we might be able to configure the
links after randomly restarting salt-minions and running `salt '*' state.apply` …


Using a vault
=============

At the time of this writing this seems to be a valid option. I've not tried it
yet. There will be an update on this soon (tm).

Reading of the vault seems to be rather easy. You can also write to it but only
using infromation that is availbe during state-rendering. So I'm not sure what
the benefit is. One could probably combine this with the event approach to
store the public keys of the minions in the vault by listening to key events on
the master.

Conclusion
==========

This world sucks. We need better tools :/
