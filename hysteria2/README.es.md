# Hysteria 2 en Keenetic Giga KN-1012 sin memoria USB

Esta guía parte de un router sin Entware. Primero instala Entware en la memoria interna del KN-1012, después H-wave y Hysteria 2. El script `S99georoute` envía directamente las IP rusas; los demás destinos pasan por Hysteria para los dispositivos asignados a la política `Hwave`.

La decisión se toma por IP, no por una lista de sitios bloqueados. Un sitio ruso alojado en una CDN extranjera puede usar la VPN. H-wave solo intercepta los puertos indicados en `/opt/etc/hwave/hwave-conf.json`; el script no modifica esa lista.

Los comandos del indicador `(config)>` pertenecen a la consola KeeneticOS. Los de `~ #` o `/ #` pertenecen a Entware. No copies el indicador junto al comando.

## 1. Preparar el router

En la interfaz web de Keenetic, abre **Administración → Configuración del sistema → Configuración general → Cambiar componentes**. Activa **Paquetes OPKG** y **Módulos Netfilter del núcleo**. Reinicia si el router lo solicita. Los nombres de los menús pueden variar según el idioma de KeeneticOS.

Abre **Gestor de paquetes OPKG**, selecciona **Almacenamiento interno**, concede a tu cuenta acceso a OPKG y guarda. El KN-1012 no necesita memoria USB. La [guía oficial del modelo](https://support.keenetic.ru/giga/kn-1012/ru/18482-installing-opkg-entware-in-the-router-s-internal-memory.html) contiene capturas.

## 2. Instalar Entware en la memoria interna

Conéctate a la consola KeeneticOS con una cuenta administradora. Por ejemplo, usa `ssh admin@192.168.1.1` si esa es la dirección del router y SSH está activado. En `(config)>`, ejecuta:

```text
opkg disk storage:/ https://bin.entware.net/aarch64-k3.10/installer/aarch64-installer.tar.gz
```

Espera el mensaje `[5/5] ... Entware ... installed` en el registro del router. `Disk is unchanged` solo indica que la unidad ya estaba seleccionada; no confirma la instalación. No formatees `storage:` ni uses `no opkg disk` como intento rutinario.

Entra en la consola de Entware desde KeeneticOS:

```text
exec sh
```

El indicador cambia a `~ #` o `/ #`. Comprueba Entware y el espacio libre:

```sh
opkg --version
df -h /opt
```

El instalador también puede abrir SSH de Entware en el puerto 222. El registro muestra el usuario y la contraseña iniciales. Si indica `root` y `keenetic`, cambia la contraseña inmediatamente con `passwd` dentro de Entware. La cuenta `admin` de KeeneticOS puede ser diferente de la cuenta SSH de Entware.

## 3. Instalar H-wave

Ejecuta estos comandos en Entware. El KN-1012 utiliza el paquete `arm64` de [H-wave v2.12.2](https://github.com/for6to9si/H-wave/releases/tag/v2.12.2):

```sh
opkg update
opkg install curl nano ipset
curl -fL https://github.com/for6to9si/H-wave/releases/download/v2.12.2/hysteria_2.12.2_arm64.ipk -o /tmp/hwave.ipk
opkg install /tmp/hwave.ipk
df -h /opt
```

La instalación debe crear la política `Hwave` y `/opt/etc/init.d/S96hysteria`. Si `opkg` devuelve un error, revisa el mensaje y el espacio libre antes de continuar.

## 4. Pegar un enlace de Hysteria 2 en la configuración

Abre el archivo en Entware:

```sh
nano /opt/etc/hysteria/config.json
```

Si ya tienes un enlace `hysteria2://...`, sustituye el contenido del archivo por esta plantilla corta:

```json
{
  "server": "<PASTE_FULL_HYSTERIA2_URI_HERE>",
  "fastOpen": true,
  "lazy": true,
  "tcpRedirect": { "listen": ":60018" },
  "udpTProxy": { "listen": ":60020", "timeout": "20s" }
}
```

Sustituye solo `<PASTE_FULL_HYSTERIA2_URI_HERE>` por el enlace completo desde `hysteria2://`. Conserva las comillas JSON. Los caracteres `@`, `?`, `&` y `#` no necesitan cambios. Si el mensajero muestra `\@`, elimina la barra invertida: el enlace debe contener `@` normal. No pegues una envoltura Markdown como `[enlace](dirección)`.

Hysteria 2 acepta el enlace completo en `server`. La autenticación, TLS y Salamander ya van en el enlace; no añadas campos separados `auth`, `tls` ni `obfs` a esta variante. Guarda el archivo y sigue con los comandos de comprobación. Mantén privado el enlace.

### Si necesitas configurarlo a mano

Usa este ejemplo manual en lugar de la plantilla corta. Rellena los marcadores con los datos del servidor.

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

En `hysteria2://PASSWORD@HOST:PORT?...`, la parte anterior a `@` corresponde a `auth`, `HOST:PORT` a `server`, `obfs-password` a la contraseña Salamander y `sni` al nombre TLS. Decodifica caracteres de URL como `%40` y `%3A` antes de introducir los valores. Si `sni` está vacío, elimina la línea `"sni": "<TLS_SERVER_NAME>",`; el certificado debe seguir coincidiendo con la dirección del servidor. Si el servidor no usa Salamander, elimina el bloque `obfs`. Hysteria 2 no tiene un campo equivalente a `fp=chrome`.

Guarda en nano con `Ctrl+O`, `Enter`, `Ctrl+X`. Comprueba el JSON e inicia Hysteria:

```sh
jq empty /opt/etc/hysteria/config.json
chmod 600 /opt/etc/hysteria/config.json
sed -i 's/^ENABLED=no$/ENABLED=yes/' /opt/etc/init.d/S96hysteria
/opt/etc/init.d/S96hysteria restart
/opt/etc/init.d/S96hysteria status
```

H-wave suele instalar `jq` como dependencia. Si falta, ejecuta `opkg install jq`. Un proceso activo no confirma por sí solo la conexión al servidor; prueba el tráfico desde un dispositivo después de asignarle la política. El registro de H-wave está en `/opt/var/log/hwave/hwave.log`.

## 5. Instalar el enrutamiento dividido

En Entware, descarga y ejecuta el script. Actualiza diariamente las redes IPv4 e IPv6 rusas desde [IPdeny](https://www.ipdeny.com/):

```sh
curl -fL https://raw.githubusercontent.com/artemk1337/keenetic-vpn/main/hysteria2/S99georoute -o /opt/etc/init.d/S99georoute
chmod 700 /opt/etc/init.d/S99georoute
/opt/etc/init.d/S99georoute start
sleep 60
```

Comprueba IPv4 antes de asignar dispositivos:

```sh
/opt/etc/init.d/S99georoute status
ipset list ru_geo4 | head
iptables -t nat -S hwave | head
iptables -t mangle -S hwave | head
```

Debes ver `running`, miembros en `ru_geo4` y `--match-set ru_geo4 dst -j RETURN` antes de `REDIRECT` o `TPROXY` en ambas cadenas `hwave`. Si falta una regla, revisa el registro:

```sh
tail -40 /opt/var/log/georoute.log
```

Si H-wave usa IPv6, compruébalo también:

```sh
ipset list ru_geo6 | head
ip6tables -t nat -S hwave | head
ip6tables -t mangle -S hwave | head
```

Busca `--match-set ru_geo6 dst -j RETURN` en ambas cadenas IPv6 existentes. Resuelve los errores antes de cambiar todos los dispositivos.

## 6. Asignar dispositivos

En la interfaz web de Keenetic, asigna primero la política `Hwave` a un dispositivo. Abre un sitio ruso y otro extranjero; si ambos funcionan, asigna los demás dispositivos. Comprueba también la política de los dispositivos nuevos.

Los contadores ayudan a ver qué regla recibe tráfico:

```sh
iptables -t nat -L hwave -n -v --line-numbers | head
```

El contador de `ru_geo4` debe crecer para una IP rusa. Las IP extranjeras deben llegar a `REDIRECT`. Comprueba que el túnel funciona desde el dispositivo cliente.

## Revertir y mantener

Detén el enrutamiento dividido sin quitar H-wave:

```sh
/opt/etc/init.d/S99georoute stop
```

Los dispositivos asignados a H-wave seguirán usando Hysteria. Para quitar el arranque automático, borra el script después de detenerlo:

```sh
rm /opt/etc/init.d/S99georoute
```

El script reaplica las reglas cada 30 segundos tras reinicios de H-wave o del cortafuegos y actualiza las listas cada día. Si falla una descarga, conserva la lista anterior. El registro está en `/opt/var/log/georoute.log`.

En un KN-1012 con H-wave 2.12.2 se verificaron la lista IPv4 y las reglas `iptables` de `nat` y `mangle`. Comprueba IPv6, el reinicio y el tráfico de clientes en tu router.
