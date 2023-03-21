# deluge-namespaced-wireguard

`systemd` unit files for running [Deluge](https://deluge-torrent.org/) within an isolated network
namespace, with [WireGuard](https://www.wireguard.com/) as the only network interface within the
namespace. This forces all Deluge traffic to go through the WireGuard tunnel without impacting any
other processes running on the system.

## Rationale

The approach taken here is based on the containerization section of the
[WireGuard Namespace Integration](https://www.wireguard.com/netns/) document. This simple solution
combines WireGuard interfaces with Linux network namespaces to achieve many nice security
properties. Using this approach, we can ensure that processes running in the namespace can only
access the Internet over the WireGuard tunnel, without the need to manually configure any
kill-switches. Since the only interfaces in the namespace are the loopback interface and the
WireGuard interface, should the WireGuard tunnel go down, the processes in the namespace will
simply lose connectivity.

## Requirements

* A Linux system running `systemd` 242 or newer
* WireGuard with [`wireguard-tools`](https://git.zx2c4.com/wireguard-tools) installed
* A WireGuard config file that can be used with `wg-quick` to connect to your VPN provider
* Deluge Daemon and Deluge Web installed
* Python 3.7 or newer (optionally with [PyYAML](https://pypi.org/project/PyYAML/))

## Usage

The full solution consists of a set of `systemd` unit files (and one drop-in) in the
[systemd](systemd) directory. These files go into the `/etc/systemd/system` directory on your
system. Step-by-step instructions for using these files are found below.

### Step 1: Set up `wg-netns`

To set up an isolated network namespace with WireGuard as the only network interface, we need to
automate the steps outlined at [WireGuard Namespace Integration](https://www.wireguard.com/netns/).
Fortunately, there exists the [wg-netns](https://github.com/dadevel/wg-netns) project which
implements all the necessary steps. Follow the instructions there to install the `wg-netns` script.
`wg-netns` is essentially a namespace-aware version of `wg-quick`. It can bring up and down a
WireGuard interface using a config file similar to `wg-quick`, but in addition to the steps
performed by `wg-quick`, `wg-netns` takes care of creating the network namespace and moving the
WireGuard interface to the namespace.

Once the `wg-netns` script has been installed, we need to create a couple of `systemd` unit files in
the `/etc/systemd/system` directory to let `systemd` manage the namespaced WireGuard tunnel. The
first file we need is a templated service file, `wg-netns@.service`. The file contents below assume
`wg-netns` has been installed at `/usr/local/bin/wg-netns`. If `wg-netns` was installed at a
different location on your system, adjust the relevant values.

https://github.com/existentialtype/deluge-namespaced-wireguard/blob/aa351e71ab7dc8f80e3de9a37ad0daa8dd383e3d/systemd/wg-netns%40.service#L1-L18

The second file we need is a target file for all instances of the `wg-netns@` service.

https://github.com/existentialtype/deluge-namespaced-wireguard/blob/aa351e71ab7dc8f80e3de9a37ad0daa8dd383e3d/systemd/wg-netns.target#L1-L2

The target file makes it easier to declare dependencies on `wg-netns`-created interfaces, since we
can declare the dependency on the target instead of explicitly depending on a specific interface.

### Step 2: Configure WireGuard in network namespace

`wg-netns` cannot directly use the INI-style config files used by `wg-quick`. Instead, it uses JSON
or YAML config files. Fortunately the `wg-quick` config file has almost all the information we need
to create the `wg-netns` config file, the only additional parameter we need to supply is the name of
the network namespace to manage. For the remaining files in this guide, the network namespace is
assumed to be named `vpn`. Below is an example `wg-netns` config file in YAML:

https://github.com/existentialtype/deluge-namespaced-wireguard/blob/aa351e71ab7dc8f80e3de9a37ad0daa8dd383e3d/wireguard/vpn-wg0.yaml#L1-L15

This config file should be placed in `/etc/wireguard/` alongside config files used by `wg-quick`, so
that `wg-netns` can locate it when the name of file is used to bring up an interface.

Note that [PyYAML](https://pypi.org/project/PyYAML/) is needed for `wg-netns` to parse YAML config
files. Use JSON config files if PyYAML is not available. Refer to
[wg-netns](https://github.com/dadevel/wg-netns) for JSON examples.

Also note that in the above example, the private key is loaded from a separate file in the post-up
command. This allows the config file to be world-readable without exposing the private key.

Make sure DNS servers are specified in the interface configuration. Otherwise, DNS requests may leak
to the non-VPN interface.

With the interface config file in place, we can enable and start the interface using `systemctl`:

```shell
sudo systemctl enable --now wg-netns@vpn-wg0.service
```

Change `vpn-wg0` to the name of your interface config file. If everything went well, a network
namespace named `vpn` was created and a WireGuard interface named `wg0` was moved into it. We can
double check this is the case by starting an interactive shell within the `vpn` namespace:

```shell
sudo ip netns exec vpn sudo -u $USER -i
```

Once in the namespaced shell we can confirm that the loopback interface and the WireGuard interface
are the only network interfaces in this namespace with the following command:

```shell
ip addr
```

Also, make sure that the DNS servers specified in the config file are used for DNS resolution:

```shell
cat /etc/resolv.conf
```

Finally, check that our public facing IPs come from the VPN provider:

```
curl -4 icanhazip.com
curl -6 icanhazip.com
```

### Step 3: Run Deluge Daemon in network namespace

To run the Deluge Daemon `deluged` in the `vpn` network namespace, we will add a `systemd` drop-in
file to set the namespace parameters. This means that, if you already have a `systemd` service file
that runs `deluged`, there's no need to modify it. If you are not yet using `systemd` to run
`deluged`, you can put the following file in `/etc/systemd/system`.

https://github.com/existentialtype/deluge-namespaced-wireguard/blob/aa351e71ab7dc8f80e3de9a37ad0daa8dd383e3d/systemd/deluged.service#L1-L15

The above file assumes `deluged` is installed at `/usr/bin/deluged` and that a `deluge` user
`deluge` group exist on the system which are used to run the `deluged` process. It also starts
`deluged` with its default daemon port of 58846.

The `deluged.service` file does not depend on network namespaces at all. If started without any
drop-ins, it will run `deluged` in the root namespace as normal. To modify it for running in the
`vpn` namespace, we add the following drop-in to the `/etc/systemd/system/deluged.service.d`
directory. The last part of the directory name needs to be changed if your service file for
`deluged` is named something other than `deluged.service`.

https://github.com/existentialtype/deluge-namespaced-wireguard/blob/aa351e71ab7dc8f80e3de9a37ad0daa8dd383e3d/systemd/deluged.service.d/netns.conf#L1-L7

In this file we make use of the `NetworkNamespacePath` parameter which was added in `systemd` 242.
This parameter points to the file created when the `vpn` network namespace was created. Consult the
manual page for `ip-netns` to determine where this file is placed on your system.

Instead of using `NetworkNamespacePath`, we can instead change the `ExecStart` setting to invoke
`ip netns exec vpn` before running `deluged`, which will work with older versions of `systemd`. But
using `ip netns exec` means we lose some of the benefits of having `systemd` manage the namespace
settings, which will be seen in the later steps.

We also specify `BindReadOnlyPaths` to bind mount the namespace's `resolv.conf` in `/etc`. Normally,
`ip netns exec` takes care of the bind mount, but since we are using `systemd`'s namespace support
instead of invoking `ip netns exec`, we have to take care of the bind mount explicitly.

This drop-in declares a `Requires` and `After` dependency on `wg-netns.target` because the
namespaced WireGuard tunnel needs to start first in order for the namespace files to exist.

With the drop-in in place we can enable and start the Deluge Daemon:

```shell
sudo systemctl enable --now deluged.service
```

### Step 4: Enable proxy to Deluge Daemon in network namespace

With `deluged` started in the network namespace, connecting to it using either `deluge-console`
or `deluge-web` from outside of the namespace is no longer possible. To get the `deluged` clients
working from outside of the namespace again, we need to run a reverse proxy which listens for
connections in the root namespace and forwards them to `deluged` in the `vpn` namespace.

Usually, this forwarding is accomplished using [`socat`](https://linux.die.net/man/1/socat).
However, since we are using `systemd`, there's something more convenient we can use:
[systemd-socket-proxyd](https://www.freedesktop.org/software/systemd/man/systemd-socket-proxyd.html),
a utility which is distributed with `systemd`. `systemd-socket-proxyd` can create the proxy
connection we need using a `systemd` socket, and can cross the namespace boundary using `systemd`'s
built-in support for namespaces.

Compared to the `socat` approach, this approach has a number of benefits:

* No need to install an additional package.
* No need to run as root. Since this approach does not invoke `ip netns exec`, the proxy process can
  be run as the `deluge` user.
* Simple configuration: `systemd` takes care of locating the right namespace to proxy to.
* Socket activation: the proxy process only starts when a Deluge client tries to access the daemon.
* Idle deactivation: the proxy process can be configured to terminate after all Deluge clients
  disconnect.

To set up the proxy first we need to add a socket file to `/etc/systemd/system`:

https://github.com/existentialtype/deluge-namespaced-wireguard/blob/aa351e71ab7dc8f80e3de9a37ad0daa8dd383e3d/systemd/proxy-to-deluged.socket#L1-L8

We also need the `systemd-socket-proxyd` service on the socket:

https://github.com/existentialtype/deluge-namespaced-wireguard/blob/aa351e71ab7dc8f80e3de9a37ad0daa8dd383e3d/systemd/proxy-to-deluged.service#L1-L11

We set `JoinsNamespaceOf=deluged.service` to have `systemd` determine which namespace the proxy
needs to join to forward traffic to `deluged`. This setting needs to be combined with
`PrivateNetwork=yes` to have effect. We also pass `--exit-idle-time=5min` to the proxy process to
have it automatically shut down after 5 minutes of no connections. Note that in Deluge Web, we need
to disconnect from the daemon in the Connection Manager for the proxy to shut down. Otherwise,
Deluge Web holds the connection open. This isn't very important, as the proxy process is very
lightweight and there's nothing wrong with keeping it running. The `--exit-idle-time=` parameter can
be omitted altogether.

After setting up the unit files, we can bring up the proxy socket:

```shell
sudo systemctl enable --now proxy-to-deluged.socket
```

If you have `deluge-console` working prior to the namespace setup, you can try running the console,
and it should be able to establish a connection to the daemon through the proxy.

### Step 5: Run Deluge Web

Since the proxy forwards the same port from the root namespace to the isolated namespace, there's no
need to modify Deluge Web's configuration to work with the daemon in the isolated namespace. If you
already have Deluge Web configured to run, it should work without any changes. If you don't have
Deluge Web configured yet, the following `systemd` service file can be placed in the
`/etc/systemd/system` directory:

https://github.com/existentialtype/deluge-namespaced-wireguard/blob/aa351e71ab7dc8f80e3de9a37ad0daa8dd383e3d/systemd/deluge-web.service#L1-L15

Enable and start the Deluge Web service with:

```shell
sudo systemctl enable --now deluge-web.service
```

The above configuration assumes Deluge Web is installed at `/usr/bin/deluge-web` and the system has
a `deluge` user and a `deluge` group.

Navigate to Deluge Web's default listening port of 8112, and use the Connection Manager to connect
to the Deluge Daemon running on `locahost:58846` (assuming `deluge-web` is running on the same host
as `deluged`). This will trigger the socket to activate the proxy and allow Deluge Web to connect to
the daemon inside the namespace.

## Additional considerations

The approach used here is similar to that of the
[namespaced-openvpn](https://github.com/slingamn/namespaced-openvpn) project, substituting WireGuard
in place of OpenVPN. The documentation of the namespaced-openvpn project contains an in-depth
discussion of the security considerations of this approach, many of which also apply here. In
particular, the discussion around DNS leakage using the bind mount `resolv.conf` inside the
namespace and the solution proposed may be applied here.
