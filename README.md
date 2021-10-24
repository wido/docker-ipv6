# Docker IPv6
This respository contains various scripts and tools to enable [Docker](https://www.docker.com/) to
use a IPv6 prefix obtained through DHCPv6 Prefix Delegation.

This allows containers to use native IPv6 without any need for proxies.

The scripts and tooling in this repository were tested on:
* Ubuntu 14.04 (upstart)
* Ubuntu 16.04 (systemd)
* ArchLinux rolling 2021-10-24 (systemd + systemd-networkd)

# Prefix Delegation
[Prefix Delegation](https://tools.ietf.org/html/rfc37690) (PD) is a mechanism which allows a client to request a
IPv6 prefix to be routed towards that host.

Usually a client will obtain a prefix larger than a /64 to allow splitting
up this prefix in multiple smaller prefixes.

# Docker and IPv6
Docker can use IPv6 for it's containers if you start the docker daemon with
the --ipv6 flag.

For example:

```dockerd --ipv6 --fixed-cidr-v6=2001:db8:100::/80```

IPv6 addresses of Docker containers are based on the MAC address. A /80 subnet
combined with the 48-bits of a MAC address sums up to 128-bits for a full IPv6
address.

More information about Docker and IPv6 can be found in Docker's [documentation](https://docs.docker.com/engine/userguide/networking/default_network/ipv6/).

# Interface
**NOTE:**
This documentation uses `eth0` as an example interface.
Change this in this accordingly in all commands and configurationfiles.

Replace all occurences of eth0 by your primary interface.
With newer kernels this might be something like *ens3*.

# Dependencies

Install the packages required for these commands:
* sipcalc
* grep
* head
* awk
* command
* dhclient

# Kernel settings

## systemd-networkd

Make sure these settings are set within your network unit (See https://www.freedesktop.org/software/systemd/man/systemd.network.html):
```ini
[Network]
DHCP=yes
IPv6AcceptRA=yes
IPForward=yes
```

Now restart systemd-networkd via `systemctl restart systemd-networkd` to load the changed configuration

## without systemd-networkd (sysctl)
A few sysctl settings have to be applied for IPv6 to work properly.

To do so create `/etc/sysctl.d/99-ipv6.conf` and add:

```ini
net.ipv6.conf.all.forwarding=1
net.ipv6.conf.eth0.accept_ra=2
```

You can now apply these settings:

```bash
sysctl -p /etc/sysctl.d/99-ipv6.conf
```

You can also apply these settings **temporarily** until a reboot by running this instead:
```
sysctl net.ipv6.conf.all.forwarding=1
sysctl net.ipv6.conf.eth0.accept_ra=2
```

# dhclient
dhclient is part of ISC DHCP and can request an IPv6 Prefix through DHCPv6.

Therefore run dhclient with these flags:

```bash
dhclient -6 -P -d eth0
```

It will request a prefix through DHCPv6.
The `-d` keeps dhclient running in the foreground.

You can find the DHCPv6 information in `/var/lib/dhcp/dhclient6.leases`

## Upstart
Under Upstart (e.g. Ubuntu 14.04) you can run dhclient for Prefix Delegation
using the provided configuration file.
Just place `dhclient6-pd.conf` in `/etc/init/dhclient6-pd.conf`
to have upstart request the prefix and renew when needed.

Now start dhclient:
```bash
sudo start dhclient6-pd
```

## systemd
Using systemd (e.g. Ubuntu 16.04) you can run dhclient for Prefix Delegation
using the provided configuration file.
Just place `dhclient6-pd.service` in `/etc/systemd/system/`
to have systemd request the prefix and renew when needed.

Now reload systemd and start dhclient:
```bash
systemctl daemon-reload
systemctl start dhclient6-pd
systemctl status dhclient6-pd
```

## Docker IPv6 hook
The `docker-ipv6` dhclient hook in this repository should be placed in
`/etc/dhcp/` where it will be executed after dhclient obtains a prefix.

The hook will get the first **/80** subnet out of the delegated prefix and
write it to `/etc/docker/ipv6.prefix`

The Docker daemon is then restarted so it will use the new subnet as the fixed
IPv6 cidr.

Depending on your system (upstart vs systemd), configuration has to be done differently.
The end result is the same.

Afterwards you can print the processlist and see docker running with these arguments:
```bash
/usr/bin/dockerd --ipv6 --fixed-cidr-v6=2001:00db8:100:0000:0000:0000:0000:0000/80
```

Before changing the startup parameters, make sure that neither `ipv6` nor `fixed-cidr-v6`
are specified within `/etc/docker/daemon.json`

### Upstart
With systems using Upstart (like Ubuntu 14.04) you can specify the command line options
within a configuration file within `/etc/default`.

Set this within the file `/etc/default/docker`:
```bash
DOCKER_OPTS="--ipv6 --fixed-cidr-v6=`cat /etc/docker/ipv6.prefix`"
```

### Systemd
With systems using Systemd (like Ubuntu 16.04) you can specify the command line options
within the unit file itself.

Edit the docker service via:
```bash
systemctl edit docker.service
```
and modify the `ExecStart` command like this:
```ini
### Editing /etc/systemd/system/docker.service.d/override.conf
### Anything between here and the comment below will become the new content of the file

[Service]
ExecStart=
ExecStart=/bin/bash -c "/usr/bin/dockerd --ipv6 --fixed-cidr-v6=`cat /etc/docker/ipv6.prefix` -H fd://"

### Lines below this comment will be discarded
```

Now reload systemd and stop/start Docker:

```bash
systemctl daemon-reload
systemctl stop docker
systemctl start docker
```
