Title: Install Quassel on CentOS 7
Summary: The other day I installed Quassel Core on a CentOS 7 server and Quassel Client on Fedora 21.
Tags: CentOS, Fedora, Linux, Quassel, IRC

In this post I explain how I installed the IRC client Quassel on my CentOS 7
server.

## What is Quassel?

Quassel is an IRC client with both a server and client component. It is
possible to install `quassel-core` (the server component) locally on your
laptop and then connect `quassel-client` to localhost. However, that pretty
much defeats the purpose of Quassel, and you might as well run something like
[HexChat](https://hexchat.github.io/) in that case.

The advantage with Quassel's server-client model is that the client can be
offline while the server stays connected and acts as a
[bouncer](http://en.wikipedia.org/wiki/BNC_(software)#IRC). You appear as
constantly connected and when the client reconnects to the core the chat
history is retrieved.

## EPEL

Quassel is not available in CentOS's default repositories. However, it is
available in [EPEL](https://fedoraproject.org/wiki/EPEL),
which stands for "Extra Packages for Enterprise Linux," as
[`quassel-core`](http://dl.fedoraproject.org/pub/epel/7/x86_64/repoview/quassel-core.html).

To install EPEL on CentOS, execute the following command:

    yum install epel-release

## Install Quassel Core

Then install Quassel Core on the server:

    yum install quassel-core

## Install PostgreSQL

For PostgreSQL support we also need to install the following two packages:

    yum install postgresql-server qt-postgresql

## Create a new user

As explained on <http://badadmin.net/quassel-core-on-fedora/>, we need to
create a user for running Quassel:

    useradd -r -U quassel
    mkdir /srv/quassel
    chown -R quassel:quassel /srv/quassel

## Set up the database

On the page <https://getpantheon.atlassian.net/wiki/display/PUBLIC/Quassel+IRC>
there are a set of instructions explaning how to set up a PostgreSQL database
for Quassel:

    su - postgres -c initdb
    systemctl start postgresql.service
    su - postgres -c "createuser quassel"
    su - postgres -c "createdb --owner=quassel --encoding=UTF8 quassel"

    su - quassel -c "quasselcore --select-backend=PostgreSQL --configdir=/srv/quassel"

    # If creating a user fails, run the following:
    su - quassel -c "quasselcore --add-user"

In addition I also had to add the following three lines to the file
`/var/lib/pgsql/data/pg_hba.conf`:

    local   quassel         quassel                                 md5
    host    quassel         quassel         127.0.0.1/32            md5
    host    quassel         quassel         ::1/128                 md5

Otherwise I would get an authentication error when executing

    quasselcore --select-backend=PostgreSQL --configdir=/srv/quassel

as the `quassel` user. Do not forget to also restart Postgres if you add those
lines.

## Generate a certificate

For us to be able to connect to our Quassel core over SSL, we need to generate
a self-signed certificate. Do that by running the following command:

    openssl req -x509 -nodes -days 365 -newkey rsa:4096 -keyout /srv/quassel/quasselCert.pem -out /srv/quassel/quasselCert.pem

## Create a systemd unit file

Furthermore as explained on <http://badadmin.net/quassel-core-on-fedora/>, we
need to create a systemd unit file. Mine at
`/etc/systemd/system/quassel.service` looks like this:

    [Unit]
    Description=Quassel Core
    After=network.target

    [Service]
    User=quassel
    Group=quassel
    PIDFile=/var/run/quassel.pid
    ExecStart=/usr/bin/quasselcore --require-ssl --configdir=/srv/quassel

    [Install]
    WantedBy=multi-user.target

## Start Quassel core

Finally start Quassel with the following two commands:

    systemctl start quassel
    systemctl enable quassel

Then see if it is running with the following command:

    systemctl status quassel

## Install the Quassel client

Now, on your laptop, install the Quassel client. I am running Fedora and I
installed it with:

    dnf install quassel-client

Then connect to the IP address of your server and a username and password of
your choice.
