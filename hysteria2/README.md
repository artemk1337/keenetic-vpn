# Hysteria 2 on Keenetic: GeoIP split routing

Start with a Keenetic Giga KN-1012 without Entware. The guides cover internal-storage Entware installation, [H-wave](https://github.com/for6to9si/H-wave), Hysteria 2 configuration, and split routing. Devices assigned to the `Hwave` policy use their normal connection for Russian destination IPs. Other destinations follow the H-wave route through Hysteria 2.

Read the setup guide in your language:

- [Русский](README.ru.md)
- [English](README.en.md)
- [Español](README.es.md)

The script uses country IP lists from [IPdeny](https://www.ipdeny.com/). Country IP routing does not identify blocked domains. The script contains no VPN credentials.
