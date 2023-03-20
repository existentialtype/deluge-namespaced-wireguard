# deluge-namespaced-wireguard

systemd config files for running [Deluge](https://deluge-torrent.org/) within an isolated network
namespace, with [WireGuard](https://www.wireguard.com/) as the only network interface within the
namespace. This forces all Deluge traffic to go through the WireGuard tunnel without impacting any
other processes running on the system.

## Requirements

* A Linux system running systemd 242 or newer
* WireGuard with [`wireguard-tools`](https://git.zx2c4.com/wireguard-tools) installed
* A WireGuard config file that can be used with `wg-quick` to connect to your VPN provider
* Deluge Daemon and Deluge Web installed
* Python 3.7 or newer (optionally with [PyYAML](https://pypi.org/project/PyYAML/))
