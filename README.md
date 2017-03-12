openvpn-netns
=============

Start OpenVPN connection inside Linux network namespace.

These scripts allow some programs to use the VPN connection while the
rest of the system uses the normal network connection. For programs
inside the namespace, the only network connection to the outside world
is through the VPN tunnel. This prevents VPN leaks. Multiple VPN
connections can be opened at the same time each in its separate
namespace.

This is based on https://github.com/pekman/openvpn-netns and modified
to work with systemd and Debian Jessie OpenVPN package, and to setup
an exta bridged interface inside the VPN namespace (careful of leaks
in that case!!!) if desired.

Installing
----------

    sudo ln -s "$PWD"/systemd/system/netns@.service /etc/systemd/system/netns@.service
    sudo ln -s "$PWD"/systemd/system/netns-bridge@.service /etc/systemd/system/netns-bridge@.service
    sudo ln -s "$PWD"/openvpn/netns /etc/openvpn/netns

Then see the `example` folder for using it, all of the subfolders and files are
meant to be put under `/etc`.

Configuration
-------------

In Debian, an OpenVPN tunnel configured in the file `/etc/openvpn/foovpn.conf`
is started using:

    sudo systemctl start openvpn@foovpn.service

To setup such a tunnel inside its own namespace (named `foovpn`, based
on the openvpn configuration filename):

1. Add the following into the drop-in configuration file
`/etc/systemd/system/openvpn@foovpn.service.d/netns.conf`:

    [Unit]
    Requires=netns@foovpn.service
    After=netns@foovpn.service

2. Modify `/etc/openvpn/foovpn.conf` to add the following:

    # ensure there is no up/down for the openvpn resolvconf script
    script-security 2
    route-noexec
    ifconfig-noexec
    up /etc/openvpn/netns
    route-up /etc/openvpn/netns
    down /etc/openvpn/netns

Normally, DNS settings received from OpenVPN server are used inside
the namespace. To override them, create the file
`/etc/netns/foovpn/resolv.conf`, which will be bind mounted into
`/etc/resolv.conf` inside the namespace (see `man ip-netns`). If the
file doesn't exist, it is automatically generated from DNS settings
from the server when the connection is started and deleted when the
connection is terminated.

Then to run a command inside the foovpn network namespace, use:

    sudo ip netns exec foovpn /usr/bin/command...

To setup a systemd service (e.g., foo.service) to be run inside the `foovpn`
network namespace:

1. Add the following into the drop-in configuration file
`/etc/systemd/system/foo.service.d/netns.conf`:

    [Unit]
    Requires=netns@foovpn.service openvpn@foovpn.service
    After=netns@foovpn.service openvpn@foovpn.servic

2. Modify the `foo.service` file (put it in `/etc/systemd/system` if it is
coming from a Debian package) to have the following:

    [Service]
    # Run as root, user/group is setup in the ExecStart
    User=root
    # this needs to be wrapped in netns exec and runuser to work
    ExecStart=/bin/ip netns exec foovpn runuser -g group -u user -- /usr/bin/DAEMON --DAEMON-ARGUMENTS
    # Ensure it will not restart too fast for the vpn to be fully setup if needed
    RestartSec=10s

To setup such a systemd service but with an extra interface bridged to one of
the existing interface of the host system (useful if you need the service
to listen to a local interface on top of using the openvpn network):

1. Modify the drop-in configuration file
`/etc/systemd/system/foo.service.d/netns.conf` to be as following instead:

    [Unit]
    Requires=netns@foovpn.service netns-bridge@foovpn.service openvpn@foovpn.service
    After=netns@foovpn.service netns-bridge@foovpn.service openvpn@foovpn.service

2. Add a configuration file /etc/netns/foovpn/bridge.env containing the
followin:

    INTERFACE=eth0
    ADDRESS=192.168.0.50/24

It will setup an interface named `mv.eth0` (`mv` is for `macvlan`) inside
the namespace with the address `192.168.0.50/24`. Note that it would make
most sense to use an address available in the same network as the original
`eth0` interface.
Then `eth0` will be able to connect to `192.168.0.50` as well as any
machine on the network of `eth0`.
