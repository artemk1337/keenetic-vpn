# AmneziaWG 3.1 on Keenetic / Netcraze with Entware

[Русский](README.ru.md)

This guide follows [Amnezia Netcraze](https://github.com/Parsefall/amnezia-netcraze). Its author tested the package on a Netcraze Giga NC-1012, hardware revision `1210C000`, firmware `5.1.5 / 5.01.C.5.0-0`, and kernel `4.9-ndm-5`. It runs AmneziaWG 3.1 through `/dev/net/tun` without a third-party kernel module. Other router models and firmware versions have not been verified.

## Check the router first

You need ARM64 (`aarch64`), Entware mounted at `/opt`, root access to its SSH shell, and `/dev/net/tun`. Leave room for Python, the application, and backups. The tested router has 512 MiB of RAM; the project does not state a minimum.

Find the router model and firmware version in its web interface. In the **Entware shell**, run:

```sh
uname -m
uname -r
ls -l /dev/net/tun
df -h /opt
free -m
opkg --version
```

An Entware prompt usually looks like `~ #`. The KeeneticOS `(config)>` prompt is a different command line; run the `opkg`, `tar`, and `sh` commands below in Entware. The Entware SSH port depends on your setup. The [Keenetic example](https://support.keenetic.ru/giga/kn-1012/ru/20980-installing-the-entware-repository-on-a-usb-drive.html) uses `222`; it can be `22` when the separate KeeneticOS SSH server is absent.

If Entware is missing, follow Keenetic's [USB installation guide](https://support.keenetic.ru/giga/kn-1012/ru/20980-installing-the-entware-repository-on-a-usb-drive.html). KN-1012 also has an [internal storage guide](https://support.keenetic.ru/giga/kn-1012/ru/18482-installing-opkg-entware-in-the-router-s-internal-memory.html). These guides install Entware; they do not establish AWG 3.1 compatibility with other firmware.

Back up the router configuration. Keep the regular ISP connection working until you have tested the VPN.

## Get an AmneziaWG client profile

Create a separate Amnezia client for the router. Export its `.vpn` file, a `.txt` file containing the full `vpn://…` key, or copy the key itself. Do not use the same client profile on the router and a phone at the same time. A subscription key or a `PrivateKey` alone cannot be imported. Keep the profile and full key out of issues and terminal logs.

## Install the package

On your computer, download `awg3-netcraze-arm64-userspace.tar.gz` and its `.sha256` file from the [latest release](https://github.com/Parsefall/amnezia-netcraze/releases/latest). On macOS, calculate the checksum and compare it with the published file:

```sh
shasum -a 256 awg3-netcraze-arm64-userspace.tar.gz
cat awg3-netcraze-arm64-userspace.tar.gz.sha256
```

Copy the archive to the router. Replace the example address and port with yours:

```sh
scp -O -P 222 awg3-netcraze-arm64-userspace.tar.gz root@192.168.1.1:/opt/tmp/
ssh -p 222 root@192.168.1.1
```

Run the following in the Entware shell. Stop if a command fails:

```sh
opkg update
opkg install python3-light python3-codecs python3-openssl python3-email python3-urllib python3-logging openssl-util ca-bundle
mkdir -p /opt/tmp/awg3-setup
tar -xzf /opt/tmp/awg3-netcraze-arm64-userspace.tar.gz -C /opt/tmp/awg3-setup
cd /opt/tmp/awg3-setup/awg3-userspace
sh router/install.sh
sh router/install-web.sh
```

These commands come from the [package author's installation guide](https://github.com/Parsefall/amnezia-netcraze/blob/main/INSTALL.md). No profile has been imported or route changed at this point.

## Set up the web panel and tunnel

Replace the router address and home network below with yours. The panel prompts for a password; record the displayed certificate fingerprint.

```sh
/opt/bin/python3 /opt/lib/awg3/web/server.py --setup --bind 192.168.1.1 --network 192.168.1.0/24 --port 8088
/opt/etc/init.d/S101awg3-web enable
```

Open `https://192.168.1.1:8088` from your home network and check the self-signed certificate fingerprint. In the panel, open **Tunnels**, add a tunnel, and upload the `.vpn` or `.txt` file. You can also paste the full `vpn://…` key. Choose **Create and start**. HTTP works on the same port, but sends the password and profile without encryption.

## Route traffic through the VPN

Starting a tunnel does not move any devices to the VPN. The application creates an `OpkgTunN` interface. Assign the devices you want to that connection in Keenetic's routing settings. `AllowedIPs` in the profile does not add KeeneticOS routes. Imported DNS settings are not applied automatically, and the project does not route IPv6 through the VPN. See the project's [routing guide](https://github.com/Parsefall/amnezia-netcraze/blob/main/docs/ROUTING.md).

On an assigned device, turn off any VPN running on the device itself, then check its public IP and website access. After that, enable tunnel autostart in the panel, save the KeeneticOS configuration, and test again after reboot. Set up PingCheck separately if you need connection monitoring.

If installation or the tunnel fails, start with the project's [troubleshooting guide](https://github.com/Parsefall/amnezia-netcraze/blob/main/docs/TROUBLESHOOTING.md). For another router model, find a build for its architecture and firmware first. A prebuilt `amneziawg.ko` for another kernel may not load on your router.
