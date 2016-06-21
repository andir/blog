.. title: Postgresql-tmpfs with systemd.socket-activation for local (ephemeral) data during development
.. slug: postgresql-tmpfs-with-sytemdsocket-activation-for-local-ephemeral-data-during-development
.. date: 2016-04-22 09:50:09 UTC
.. tags: systemd, postgresql, docker, tmpfs
.. category: linux
.. link: 
.. description:
.. type: text

During development of database related stuff you commonly run into the "issue" (or non-issue depending on your taste) of running a local database server - or multiple of those.

In my case I have to run a local postgresql server on my notebook. I asked myself: I'm not always developing on that piece of software, and I do not always require or want a local postgresql server. What can I do about that?!?

On top of that using my precious SSD to store data I am going to delete anyway souds like a waste (or money). In my development environment I can and want to safely wipe the data often. Also most of the database load comes from running test cases anyway. That stuff doesn't need to end up on my (slow, compared to RAM) disk. Using a tmpfs for that kind of stuff sounds much saner to me.

The part of running a repetitive clean database setup sounded like the use case for a container based thing. These days docker is pretty "hot" and it solves the issue of distributing re-useable images. There is an official postgresql image on docker hub for various versions of postgresql. I've simply build a new image based on that. It is available on docker hub (https://hub.docker.com/r/andir/postgresql-tmpfs/) or if you prefer to build it on your own you can download the Dockerfile on GitHub (https://github.com/andir/postgresql-tmpfs).

Now that we are past the introductional blabla here are the systemd unit files I'm using to achieve this:

.. gist:: https://gist.github.com/andir/d8307bcead6d83945db462698163ff40

You can either put those unit files in `/etc/systemd/system` or install them as systemd-user units in `~/.config/systemd/user`.

.. code:: bash

    systemctl daemon-reload
    systemctl enable postgresql-docker.socket

If you try to connect to the postgresql server (:code:`nc 127.0.0.1 5432`) you can observe the container while it is starting (:code:`journalctl -f`).

The default username, password and database name is `postgres`. You can change that by modifying the startup arguments of the docker container. Those are documented at https://hub.docker.com/_/postgres/.

Happy data trashing \\o/

P.S.: If you've an idea on how to stop the service after x minutes of inactivity please let me know. Stopping the service manually isn't really what I'm after.
