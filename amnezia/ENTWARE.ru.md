# AmneziaWG 3.1 на Keenetic / Netcraze с Entware

[English](ENTWARE.en.md) · [Штатная установка в KeeneticOS](README.ru.md)

Эта инструкция опирается на [Amnezia Netcraze](https://github.com/Parsefall/amnezia-netcraze). Автор проверил пакет на Netcraze Giga NC-1012 с аппаратной ревизией `1210C000`, прошивкой `5.1.5 / 5.01.C.5.0-0` и ядром `4.9-ndm-5`. Проект запускает AmneziaWG 3.1 через `/dev/net/tun` без стороннего модуля ядра. На других моделях и прошивках работа пакета не подтверждена.

Если роутер поддерживает KeeneticOS 5.2 Alpha 11 или новее, начните со [штатной установки](README.ru.md). Этот вариант через Entware нужен для прошивки без нативной поддержки AWG 3.1 и проверен только на указанном NC-1012.

## Перед установкой

Нужны ARM64 (`aarch64`), установленный Entware в `/opt`, доступ `root` к его оболочке SSH и устройство `/dev/net/tun`. Понадобится место для Python, приложения и резервных копий. Автор проверял проект на роутере с 512 МиБ ОЗУ, но минимальный объём памяти не указан.

Модель и версию прошивки посмотрите в веб-интерфейсе роутера. Затем войдите в **оболочку Entware** и выполните:

```sh
uname -m
uname -r
ls -l /dev/net/tun
df -h /opt
free -m
opkg --version
```

Приглашение Entware обычно выглядит как `~ #`. Приглашение KeeneticOS `(config)>` означает другую консоль: команды `opkg`, `tar` и `sh` ниже вводят в Entware. Порт SSH Entware зависит от настроек; в [примере Keenetic](https://support.keenetic.ru/giga/kn-1012/ru/20980-installing-the-entware-repository-on-a-usb-drive.html) это `222`, а без отдельного SSH-сервера KeeneticOS может быть `22`.

Если Entware ещё нет, установите его по [инструкции Keenetic для USB-накопителя](https://support.keenetic.ru/giga/kn-1012/ru/20980-installing-the-entware-repository-on-a-usb-drive.html). Для KN-1012 есть и [инструкция по встроенной памяти](https://support.keenetic.ru/giga/kn-1012/ru/18482-installing-opkg-entware-in-the-router-s-internal-memory.html). Обе страницы описывают установку Entware; они не подтверждают совместимость пакета AWG 3.1 с другими прошивками.

Сохраните настройки роутера до установки. Оставьте обычное подключение к провайдеру рабочим, пока не проверите VPN.

## Получите профиль AmneziaWG

Создайте в Amnezia отдельного клиента для роутера. Экспортируйте его файл `.vpn`, файл `.txt` с полным ключом `vpn://…` или скопируйте сам ключ. Один клиентский профиль нельзя одновременно использовать на телефоне и роутере. Ключ подписки и один только `PrivateKey` для импорта не подходят. Не публикуйте профиль и полный ключ в issue или журнале терминала.

## Установите пакет

Скачайте на компьютер `awg3-netcraze-arm64-userspace.tar.gz` и одноимённый файл `.sha256` из [последнего релиза](https://github.com/Parsefall/amnezia-netcraze/releases/latest). На macOS проверьте сумму и сравните её с содержимым `.sha256`:

```sh
shasum -a 256 awg3-netcraze-arm64-userspace.tar.gz
cat awg3-netcraze-arm64-userspace.tar.gz.sha256
```

Копируйте архив на роутер. Замените адрес и порт в примере своими:

```sh
scp -O -P 222 awg3-netcraze-arm64-userspace.tar.gz root@192.168.1.1:/opt/tmp/
ssh -p 222 root@192.168.1.1
```

Следующие команды выполняйте в оболочке Entware. Переходите к следующей команде только после успешного завершения предыдущей:

```sh
opkg update
opkg install python3-light python3-codecs python3-openssl python3-email python3-urllib python3-logging openssl-util ca-bundle
mkdir -p /opt/tmp/awg3-setup
tar -xzf /opt/tmp/awg3-netcraze-arm64-userspace.tar.gz -C /opt/tmp/awg3-setup
cd /opt/tmp/awg3-setup/awg3-userspace
sh router/install.sh
sh router/install-web.sh
```

Это команды [автора пакета](https://github.com/Parsefall/amnezia-netcraze/blob/main/INSTALL.md). На этапе установки профиль ещё не импортирован и маршруты не меняются.

## Настройте панель и туннель

Подставьте локальный IP роутера и подсеть вашей домашней сети. Пароль панели вводится интерактивно; запишите показанный отпечаток сертификата.

```sh
/opt/bin/python3 /opt/lib/awg3/web/server.py --setup --bind 192.168.1.1 --network 192.168.1.0/24 --port 8088
/opt/etc/init.d/S101awg3-web enable
```

Откройте `https://192.168.1.1:8088` из домашней сети и сверьте отпечаток самоподписанного сертификата. В панели откройте «Туннели», добавьте туннель и загрузите `.vpn` или `.txt`. Полный `vpn://…` можно вставить прямо в поле ключа. Затем нажмите «Создать и запустить». HTTP на том же порту тоже работает, но передаёт пароль и профиль без шифрования.

## Направьте трафик через VPN

Создание туннеля само по себе не переводит устройства на VPN. Приложение регистрирует интерфейс `OpkgTunN`; назначьте нужные устройства этому подключению в штатных настройках маршрутизации Keenetic. Поле `AllowedIPs` в профиле не создаёт маршруты прошивки. DNS из импортированного профиля автоматически не применяется, IPv6 через VPN проект не настраивает. Подробности есть в [документе о маршрутах](https://github.com/Parsefall/amnezia-netcraze/blob/main/docs/ROUTING.md).

На устройстве, которому назначили VPN, отключите его собственный VPN и проверьте внешний IP и доступ к сайтам. После проверки включите автозапуск туннеля в панели, сохраните конфигурацию KeeneticOS и повторите проверку после перезагрузки. PingCheck для контроля связи настраивается отдельно.

Если установка или туннель не работают, начните с [раздела диагностики проекта](https://github.com/Parsefall/amnezia-netcraze/blob/main/docs/TROUBLESHOOTING.md). Для другой модели роутера сначала ищите сборку под её архитектуру и прошивку. Готовый `amneziawg.ko` из чужой сборки может не совпасть с ядром вашего роутера.
