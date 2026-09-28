# Hysteria 2 on Keenetic Giga KN-1012 without a USB drive

This guide starts with a router that has no Entware installation. It installs Entware in the router's internal storage, then H-wave and Hysteria 2. The `S99georoute` script sends Russian destination IPs directly and other destinations through Hysteria for devices assigned to the `Hwave` access policy.

Routing is based on IP addresses, not a list of blocked websites. A Russian site on a foreign CDN may use the VPN. H-wave intercepts only the ports configured in `/opt/etc/hwave/hwave-conf.json`; this script does not change them.

Commands at `(config)>` belong to the KeeneticOS CLI. Commands at `~ #` or `/ #` belong to the Entware shell. Do not include the prompt when copying a command.

## 1. Prepare the router

In the Keenetic web UI, open **Management → System settings → General system settings → Change component set**. Enable **OPKG packages** and **Netfilter kernel modules**. Reboot if requested. UI labels may vary with the KeeneticOS language and version.

Open **OPKG package manager**, select **Internal storage**, grant your account access to OPKG services, and save. No USB drive is needed on KN-1012. See the [official model guide](https://support.keenetic.ru/giga/kn-1012/ru/18482-installing-opkg-entware-in-the-router-s-internal-memory.html) for screenshots.

## 2. Install Entware in internal storage

Connect to the KeeneticOS CLI with an administrator account. For example, use `ssh admin@192.168.1.1` if that is your router's address and SSH access is enabled. At the `(config)>` prompt, run:

```text
opkg disk storage:/ https://bin.entware.net/aarch64-k3.10/installer/aarch64-installer.tar.gz
```

Wait for `[5/5] ... Entware ... installed` in the router log. `Disk is unchanged` only reports that storage was already selected; it does not confirm installation. Do not format `storage:` or run `no opkg disk` as a routine retry.

Enter the Entware shell from the KeeneticOS CLI:

```text
exec sh
```

The prompt changes to `~ #` or `/ #`. Check installation and free space:

```sh
opkg --version
df -h /opt
```

The installer may also start Entware SSH on port 222. Its log contains the initial username and password. If it reports the defaults `root` and `keenetic`, change that password immediately with `passwd` in the Entware shell. Your KeeneticOS `admin` account and the Entware SSH account may be different.

## 3. Install H-wave

Run the following in the Entware shell. KN-1012 uses the `arm64` package from [H-wave v2.12.2](https://github.com/for6to9si/H-wave/releases/tag/v2.12.2):

```sh
opkg update
opkg install curl nano ipset
curl -fL https://github.com/for6to9si/H-wave/releases/download/v2.12.2/hysteria_2.12.2_arm64.ipk -o /tmp/hwave.ipk
opkg install /tmp/hwave.ipk
df -h /opt
```

Installation should create the `Hwave` access policy and `/opt/etc/init.d/S96hysteria`. If `opkg` reports an error, check it and the free space before proceeding.

## 4. Paste a Hysteria 2 URI into the configuration

Open the configuration in the Entware shell:

```sh
nano /opt/etc/hysteria/config.json
```

If you already have a `hysteria2://...` URI, replace the file contents with this short template:

```json
{
  "server": "<PASTE_FULL_HYSTERIA2_URI_HERE>",
  "fastOpen": true,
  "lazy": true,
  "tcpRedirect": { "listen": ":60018" },
  "udpTProxy": { "listen": ":60020", "timeout": "20s" }
}
```

Replace only `<PASTE_FULL_HYSTERIA2_URI_HERE>` with the complete URI, starting at `hysteria2://`. Keep the JSON quotation marks. Characters such as `@`, `?`, `&`, and `#` need no special escaping here. If your messenger shows `\@`, remove the backslash; the URI must contain a plain `@`. Do not paste a Markdown wrapper such as `[link](address)`.

Hysteria 2 can read the URI directly from `server`. The URI already carries authentication, TLS, and Salamander settings, so do not add separate `auth`, `tls`, or `obfs` fields to this version. Save the file and continue with the validation commands below. Keep the URI private.

### If you need a manual configuration

Use this manual example instead of the short template. Replace each placeholder with your server's settings.

```json
{
  "server": "<SERVER_HOST>:<PORT>",
  "auth": "<AUTH_PASSWORD>",
  "tls": {
    "sni": "<TLS_SERVER_NAME>",
    "insecure": false
  },
  "obfs": {
    "type": "salamander",
    "salamander": {
      "password": "<OBFS_PASSWORD>"
    }
  },
  "fastOpen": true,
  "lazy": true,
  "tcpRedirect": { "listen": ":60018" },
  "udpTProxy": { "listen": ":60020", "timeout": "20s" }
}
```

In `hysteria2://PASSWORD@HOST:PORT?...`, the part before `@` is `auth`, `HOST:PORT` is `server`, `obfs-password` is the Salamander password, and `sni` is the TLS server name. Decode URL-encoded characters such as `%40` and `%3A` before entering values. If `sni` is empty, remove the entire `"sni": "<TLS_SERVER_NAME>",` line; the certificate must still match the server address. If your server does not use Salamander, remove the `obfs` block. Native Hysteria 2 has no `fp=chrome` configuration field.

Save in nano with `Ctrl+O`, `Enter`, `Ctrl+X`. Validate and start:

```sh
jq empty /opt/etc/hysteria/config.json
chmod 600 /opt/etc/hysteria/config.json
sed -i 's/^ENABLED=no$/ENABLED=yes/' /opt/etc/init.d/S96hysteria
/opt/etc/init.d/S96hysteria restart
/opt/etc/init.d/S96hysteria status
```

H-wave normally installs `jq` as a dependency. If it is missing, run `opkg install jq`. A running status confirms the process, not a working server connection; check traffic from one client after assigning the policy. H-wave logs are in `/opt/var/log/hwave/hwave.log`.

## 5. Install split routing

In the Entware shell, download and start the script. It refreshes Russian IPv4 and IPv6 lists from [IPdeny](https://www.ipdeny.com/) daily.

```sh
curl -fL https://raw.githubusercontent.com/artemk1337/keenetic-vpn/main/hysteria2/S99georoute -o /opt/etc/init.d/S99georoute
chmod 700 /opt/etc/init.d/S99georoute
/opt/etc/init.d/S99georoute start
sleep 60
```

Verify IPv4 before assigning devices:

```sh
/opt/etc/init.d/S99georoute status
ipset list ru_geo4 | head
iptables -t nat -S hwave | head
iptables -t mangle -S hwave | head
```

Expect `running`, a populated `ru_geo4` set, and `--match-set ru_geo4 dst -j RETURN` before `REDIRECT` or `TPROXY` in both `hwave` chains. If a rule is missing, read the log:

```sh
tail -40 /opt/var/log/georoute.log
```

If H-wave uses IPv6, inspect it separately:

```sh
ipset list ru_geo6 | head
ip6tables -t nat -S hwave | head
ip6tables -t mangle -S hwave | head
```

Expect `--match-set ru_geo6 dst -j RETURN` in both existing IPv6 chains. Investigate a missing IPv6 rule before switching all devices.

## 6. Assign devices

In the Keenetic web UI, assign the `Hwave` access policy to one device. Test a Russian and a foreign website on that device, then assign the rest. Check that newly joined devices receive the policy too.

Rule counters can help diagnose traffic after the test:

```sh
iptables -t nat -L hwave -n -v --line-numbers | head
```

The `ru_geo4` counter should grow for a Russian destination IP; foreign destinations should reach `REDIRECT`. Confirm tunnel operation from the client device.

## Roll back and maintain

Stop split routing without removing H-wave:

```sh
/opt/etc/init.d/S99georoute stop
```

Devices assigned to H-wave still use Hysteria. To disable script startup, remove it after stopping:

```sh
rm /opt/etc/init.d/S99georoute
```

The script reapplies its rules every 30 seconds after H-wave or firewall restarts and refreshes IP lists daily. If a download fails, it retains the previous list. Logs are in `/opt/var/log/georoute.log`.

The IPv4 list and `iptables` rules in `nat` and `mangle` were verified on KN-1012 with H-wave 2.12.2. Check IPv6, reboot behavior, and client traffic on your own router.
