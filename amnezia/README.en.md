# AmneziaWG 3.1 in KeeneticOS

[Русский](README.ru.md) · [Entware method](ENTWARE.en.md)

KeeneticOS supports AmneziaWG 3.1 natively starting with **5.2 Alpha 11**. Amnezia's [model list and full guide](https://docs.amnezia.org/ru/documentation/instructions/keenetic-os-awg/) include Keenetic Giga KN-1012 and Netcraze Giga NC-1012. You set up the connection in the router's web interface; Entware is not required.

Version 5.2 Alpha 11 is available through the developer update channel. A development build may be less stable than your current firmware. If you already run a supported version, go straight to the component installation. On older firmware, decide whether to update first. The [Entware build](ENTWARE.en.md) was tested by its author only on one NC-1012 running firmware 5.1.5; it is not a general replacement for the native feature.

## Prepare

1. Check the router model and KeeneticOS version in the web interface.
2. Download both `firmware` and `startup-config` from system settings. On newer versions, they are on the Files tab.
3. Before moving to the developer channel, review the [current build's known issues](https://forum.keenetic.ru/) and disable automatic updates so a later development build is not installed without your involvement.
4. Amnezia Premium users can get a client `.conf` from their account. For Self-hosted, create a separate AmneziaWG client for the router in AmneziaVPN, share that connection, and choose **Original AmneziaWG format**. A `vpn://…` key or `.vpn` file is not the format for native import. Keep the `.conf` private; it contains a private key.

## Create the connection

1. In System settings, open the component list, find **WireGuard VPN**, and install it with Update NDMS. Wait for installation and any required reboot.
2. Under Internet → Other connections, choose Upload from file and select the client `.conf`.
3. Enable Use for internet access, save, then turn the connection on.

For a first test, leave the regular ISP connection above WireGuard in the default policy. Create a separate policy containing WireGuard, assign one device to it, and check the device's public IP and website access. If the policy contains only WireGuard, the device loses internet when the VPN fails. Adding the ISP connection in second place keeps a fallback path but may send traffic directly if the VPN fails. Assign more devices after the test.

## DNS and IPv6

In [step 1 of Amnezia's guide](https://docs.amnezia.org/ru/documentation/instructions/keenetic-os-awg/#шаг-1-настройка-dns), Amnezia recommends adding public DNS servers, optionally ignoring ISP DNS, and disabling IPv6 on the main connection. These changes affect the whole home network. For a first test on one device, set up the connection and policy above; then adjust DNS and IPv6 for the routing scheme you choose.

For domain-based routing, follow the DNS Routes section of Amnezia's guide. Devices must use the router's DNS: custom DNS, Private DNS, and browser DNS-over-HTTPS bypass those rules. If the VPN carries only IPv4, IPv6 left enabled on the main connection may go out directly. Test both IP families before moving all devices.

Save the router configuration and repeat the connection test after reboot.
