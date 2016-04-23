# Docker IPv6
This respository contains various scripts and tools to enable [Docker](https://www.docker.com/) to
use a IPv6 prefix obtained through DHCPv6 Prefix Delegation.

This allows containers to use native IPv6 without any need for proxies.

The scripts and tooling in this repository were tested on Ubuntu 14.04 (upstart)
and 16.04 (systemd).

# Prefix Delegation
[Prefix Delegation](https://tools.ietf.org/html/rfc37690) (PD) is a mechanism which allows a client to request a
IPv6 prefix to be routed towards that host.

Usually a client will obtain a prefix larger than a /64 to allow splitting
up this prefix in multiple smaller prefixes.

# Docker and IPv6
Docker can use IPv6 for it's containers if you start the docker daemon with
the --ipv6 flag.

For example:

```docker daemon --ipv6 --fixed-cidr-v6=2001:db8:100::/80```

IPv6 addresses of Docker containers are based on the MAC address. A /80 subnet
combined with the 48-bits of a MAC address sums op to 128-bits for a full IPv6
address.

More information about Docker and IPv6 can be found in Docker's [documentation](https://docs.docker.com/engine/userguide/networking/default_network/ipv6/).

## sysctl
A few sysctl settings have to be applied for IPv6 to work properly.


To do so create */etc/sysctl.d/99-ipv6.conf* and add:

<pre>net.ipv6.conf.all.forwarding=1
net.ipv6.conf.eth0.accept_ra=2</pre>

**NOTE:** Replace eth0 by your primary interface. This might be *ens3* on newer kernels.

You can now apply these settings:

<pre>sysctl -p /etc/sysctl.d/99-ipv6.conf</pre>

A reboot of the system is suggested to make sure the settings are applied.

# dhclient
dhclient is part of ISC DHCP and can request a IPv6 Prefix through DHCPv6.

In order to do so, dhclient should be run with these flags:

```dhclient -6 -P eth0```

It will request a prefix through DHCPv6.

You can find the DHCPv6 information in */var/lib/dhcp/dhclient6.leases*

## Upstart
Under Ubuntu 14.04 you can run dhclient using upstart to
request the prefix and renew then needed.

See the upstart file in the repository to do so.

You should place this file in */etc/init/dhclient6-pd.conf*

Afterwards you can start and stop with:

``sudo start dhclient6-pd``

``sudo stop dhclient6-pd``

## systemd
Ubuntu 16.04 using systemd and to run dhclient for Prefix Delegation to have to install *dhclient6-pd.service* in */etc/systemd/system/*

Now reload systemd and start dhclient:

<pre>systemctl daemon-reload</pre>

<pre>systemctl start dhclient6-pd</pre>

<pre>systemctl status dhclient6-pd</pre>

### Interface
By default it uses eth0 as the configured interface. Change this in the service file if this differs on your system.

With newer kernels this might be *ens3* for example.

## Docker IPv6 hook
The *docker-ipv6* dhclient hook in this repository should be placed in
**/etc/dhcp/dhclient-enter-hooks.d/** where it will be executed after dhclient
obtains a prefix.

The hook will get the first **/80** subnet out of the delegated prefix and
write it to */etc/docker/ipv6.prefix*

The Docker daemon is then restarted so it will use the new subnet as the fixed
IPv6 cidr.

Depending on your Ubuntu version (14.04 or 16.04) configuration has to be done differently due to the Upstart vs systemd changes. The end result is the same.

Afterwards you can print the processlist and see docker running with these arguments:

``/usr/bin/docker daemon --ipv6 --fixed-cidr-v6=2001:00db8:100:0000:0000:0000:0000:0000/80``

### Ubuntu 14.04
In order for this to work the *DOCKER_OPTS* in /etc/default/docker should be set to:

```DOCKER_OPTS="--ipv6 --fixed-cidr-v6=`cat /etc/docker/ipv6.prefix`"```

### Ubuntu 16.04
First we copy the systemd service file for docker to /etc:

<pre>cp /lib/systemd/system/docker.service /etc/systemd/system</pre>

Now modify the *ExecStatrt* line so that it contains:

<pre>ExecStart=/bin/bash -c "/usr/bin/docker daemon --ipv6 --fixed-cidr-v6=`cat /etc/docker/ipv6.prefix` -H fd://"</pre>

Now reload systemd and stop/start Docker:

<pre>systemctl daemon-reload</pre>

<pre>systemctl stop docker</pre>

<pre>systemctl start docker</pre>

