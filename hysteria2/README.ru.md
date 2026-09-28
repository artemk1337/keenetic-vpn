# Hysteria 2 на Keenetic Giga KN-1012 без флешки

Эта инструкция начинается с чистого Keenetic Giga KN-1012. Entware ставится во внутреннюю память роутера. Затем ставим H-wave и Hysteria 2, а скриптом `S99georoute` разделяем трафик: российские IP-адреса идут напрямую, остальные - через Hysteria.

Маршрутизация идёт по IP-адресам, а не по списку заблокированных сайтов. Сайт на зарубежном CDN может попасть в VPN. H-wave перехватывает только порты, указанные в `/opt/etc/hwave/hwave-conf.json`; этот скрипт список портов не меняет.

Команды с приглашением `(config)>` относятся к CLI KeeneticOS. Команды с `~ #` или `/ #` запускайте в оболочке Entware. Не копируйте приглашение вместе с командой.

## 1. Подготовить роутер

В веб-интерфейсе Keenetic откройте «Управление», затем «Параметры системы», «Общие настройки системы» и «Изменить набор компонентов». Установите «Пакеты OPKG» и «Модули ядра подсистемы Netfilter». Если роутер просит перезагрузку, выполните её.

Затем откройте «Менеджер пакетов OPKG», выберите накопитель «Встроенное хранилище», разрешите своей учётной записи доступ к OPKG и сохраните настройки. Для KN-1012 флешка не нужна. [Инструкция Keenetic для этой модели](https://support.keenetic.ru/giga/kn-1012/ru/18482-installing-opkg-entware-in-the-router-s-internal-memory.html) содержит те же шаги со скриншотами.

## 2. Установить Entware во внутреннюю память

Подключитесь к CLI KeeneticOS учётной записью администратора. Например, на Mac это команда `ssh admin@192.168.1.1`, если `192.168.1.1` - адрес вашего роутера и SSH-доступ включён. Приглашение должно выглядеть как `(config)>`. Выполните:

```text
opkg disk storage:/ https://bin.entware.net/aarch64-k3.10/installer/aarch64-installer.tar.gz
```

Дождитесь записи `[5/5] ... Entware ... installed` в журнале роутера. Сообщение `Disk is unchanged` говорит лишь о том, что накопитель уже выбран. Само по себе оно не подтверждает установку. Не форматируйте `storage:` и не запускайте `no opkg disk` для обычного повтора.

Из CLI KeeneticOS откройте оболочку Entware:

```text
exec sh
```

Приглашение сменится на `~ #` или `/ #`. Проверьте установку и место:

```sh
opkg --version
df -h /opt
```

Установщик может также открыть SSH для Entware на порту 222. В журнале установки указан логин и временный пароль. Если там указаны стандартные `root` и `keenetic`, сразу смените пароль в оболочке Entware командой `passwd`. Учётная запись `admin` в CLI KeeneticOS и логин SSH Entware могут различаться.

## 3. Установить H-wave

Все команды в этом разделе выполняются в Entware (`~ #`). Для KN-1012 нужен пакет `arm64` из [релиза H-wave v2.12.2](https://github.com/for6to9si/H-wave/releases/tag/v2.12.2):

```sh
opkg update
opkg install curl nano ipset
curl -fL https://github.com/for6to9si/H-wave/releases/download/v2.12.2/hysteria_2.12.2_arm64.ipk -o /tmp/hwave.ipk
opkg install /tmp/hwave.ipk
df -h /opt
```

Установка должна создать политику доступа `Hwave` и файл `/opt/etc/init.d/S96hysteria`. Если пакет не установился, не переходите к настройке маршрутов: сначала прочитайте ошибку `opkg` и проверьте свободное место.

## 4. Вставить ссылку Hysteria 2 в конфигурацию

Откройте конфигурацию в Entware:

```sh
nano /opt/etc/hysteria/config.json
```

Если у вас уже есть ссылка `hysteria2://...`, замените содержимое файла на этот короткий шаблон:

```json
{
  "server": "<PASTE_FULL_HYSTERIA2_URI_HERE>",
  "fastOpen": true,
  "lazy": true,
  "tcpRedirect": {
    "listen": ":60018"
  },
  "udpTProxy": {
    "listen": ":60020",
    "timeout": "20s"
  }
}
```

Замените только текст `<PASTE_FULL_HYSTERIA2_URI_HERE>` на всю ссылку целиком, от `hysteria2://` до конца. Кавычки оставьте. В ссылке могут быть `@`, `?`, `&` и `#` - внутри JSON-строки они не требуют замены. Если мессенджер показал `\@`, уберите обратную косую черту: в самой ссылке должен быть обычный `@`. Не копируйте Markdown-обёртку вида `[ссылка](адрес)`.

Hysteria 2 умеет читать такую ссылку прямо из поля `server`. Пароль, TLS и Salamander уже находятся в ссылке; отдельные поля `auth`, `tls` и `obfs` в этом варианте добавлять не нужно. Сохраните файл и перейдите к командам проверки ниже. Ссылку с паролями не публикуйте.

### Если нужна ручная настройка

Впишите параметры подключения вручную вместо короткого шаблона:

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
  "tcpRedirect": {
    "listen": ":60018"
  },
  "udpTProxy": {
    "listen": ":60020",
    "timeout": "20s"
  }
}
```

В ссылке `hysteria2://PASSWORD@HOST:PORT?...` часть перед `@` - это `auth`, `HOST:PORT` - это `server`, параметр `obfs-password` - пароль Salamander, а `sni` - имя для TLS. Перед вставкой раскодируйте символы вида `%40` и `%3A`, если они есть в значении. Если `sni` в ссылке пустой, удалите строку `"sni": "<TLS_SERVER_NAME>",` целиком. Сертификат сервера при этом должен подходить к адресу в `server`; не отключайте проверку TLS без причины. Если у вашего сервера нет Salamander, удалите весь блок `obfs`. Параметр ссылки `fp=chrome` не имеет одноимённого поля в штатной конфигурации Hysteria 2.

Сохраните файл в nano клавишами `Ctrl+O`, `Enter`, `Ctrl+X`. Затем проверьте JSON, ограничьте доступ и запустите Hysteria:

```sh
jq empty /opt/etc/hysteria/config.json
chmod 600 /opt/etc/hysteria/config.json
sed -i 's/^ENABLED=no$/ENABLED=yes/' /opt/etc/init.d/S96hysteria
/opt/etc/init.d/S96hysteria restart
/opt/etc/init.d/S96hysteria status
```

`jq` обычно приходит как зависимость H-wave. Если команда `jq` не найдена, выполните `opkg install jq`. Статус `запущен` подтверждает работу процесса; подключение к серверу дополнительно проверьте с одного устройства после назначения политики. Журнал H-wave находится в `/opt/var/log/hwave/hwave.log`.

## 5. Установить разделение трафика

Скрипт запускается в Entware и обновляет списки российских IPv4 и IPv6 у [IPdeny](https://www.ipdeny.com/) раз в сутки:

```sh
curl -fL https://raw.githubusercontent.com/artemk1337/keenetic-vpn/main/hysteria2/S99georoute -o /opt/etc/init.d/S99georoute
chmod 700 /opt/etc/init.d/S99georoute
/opt/etc/init.d/S99georoute start
sleep 60
```

Проверьте IPv4 до назначения политики устройствам:

```sh
/opt/etc/init.d/S99georoute status
ipset list ru_geo4 | head
iptables -t nat -S hwave | head
iptables -t mangle -S hwave | head
```

Нужны статус `running`, непустой список `ru_geo4` и строка `--match-set ru_geo4 dst -j RETURN` перед `REDIRECT` или `TPROXY` в обеих цепочках `hwave`. Если правило не появилось, посмотрите `/opt/var/log/georoute.log`:

```sh
tail -40 /opt/var/log/georoute.log
```

Если H-wave использует IPv6, проверьте его отдельно:

```sh
ipset list ru_geo6 | head
ip6tables -t nat -S hwave | head
ip6tables -t mangle -S hwave | head
```

Для IPv6 ожидается строка `--match-set ru_geo6 dst -j RETURN` в обеих существующих цепочках. Если IPv6 в H-wave не работает, оставьте текущую политику устройств и разберите ошибку до массового переключения.

## 6. Назначить политику устройствам

В веб-интерфейсе Keenetic назначьте политику доступа `Hwave` сначала одному устройству. Откройте на нём российский и зарубежный сайты. Если оба работают, назначьте политику остальным устройствам. Новые устройства тоже должны получить `Hwave`; проверьте их назначение в интерфейсе Keenetic.

После открытия сайтов можно посмотреть, какие правила получили пакеты:

```sh
iptables -t nat -L hwave -n -v --line-numbers | head
```

Для российского IP должен расти счётчик у `ru_geo4`, для зарубежного - у `REDIRECT`. Счётчики показывают выбор правила, а работу VPN проверяйте с клиентского устройства.

## Откат и обслуживание

Отключить разделение трафика можно без удаления H-wave:

```sh
/opt/etc/init.d/S99georoute stop
```

После этого устройства в политике `Hwave` продолжают пользоваться Hysteria. Чтобы убрать автозапуск скрипта, удалите его после остановки:

```sh
rm /opt/etc/init.d/S99georoute
```

Скрипт повторно применяет свои правила каждые 30 секунд после перезапуска H-wave или межсетевого экрана. Если загрузка нового списка не удалась, прежний список остаётся на месте. Журнал скрипта: `/opt/var/log/georoute.log`.

На KN-1012 с H-wave 2.12.2 проверены загрузка IPv4-списка и правила `iptables` в `nat` и `mangle`. IPv6, автозапуск после перезагрузки и работа VPN на каждом клиенте требуют отдельной проверки на вашем роутере.
