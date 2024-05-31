# Установка Home Assistant Supervised

Будем ставить на Debian 12. Почему? Потому что. Можете ставить и на другие дистрибютивы, но нет гарантии что все пройдет по плану и ничего не сломается.

------------


Нам понадобится собственно сам [дистрибютив с оффициального сайта](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-12.5.0-amd64-netinst.iso "дистрибютив с оффициального сайта"). Пишем его на флешку через [Rufus](https://rufus.ie/ru/ "Rufus") или любыми другими способами, если ставите напрямую на ПК или используем как есть, если ставите на виртуалку. ПК/виртуалка должна подключаться к роутеру через DHCP, так проще будет настраивать.

------------

Устанавливаем Debian все по стандарту, без графической оболочки, без всяких излишеств. При установке выбираем только `SSH server` и `standard system utilites`. Статичный адрес после установки не спешим настраивать, оставляем по умолчанию DHCP, настройки сети все равно слетят во время установки.
![](https://github.com/GennPen/HomeAssistant/blob/main/images/01%20-%202024-05-31%20225437.jpg)

------------

После установки системы настраиваем SSH для безопасного доступа, здесь расписывать не вижу смысла, в Гугле инфы вагон и маленькая тележка. Если устанавливаете на виртуалку, то на данном этапе крайне желательно сделать снапшот системы, чтобы лишний раз не переустанавливать, если что-то пойдет не так. Да и в целом, не стесняйтесь делать снапшоты после каждого успешного этапа, это сильно экономит время.

------------

### Внимание! Все команды установки проводятся под рутом.
Заходим в рут с помощью команды `su -` (тире в конце не забудьте). Должен быть запрос пароля рут, который вы вводили при установке системы. После перехода в рут строка приветствия должна быть вида:
```
root@homeassistant:~# 
```

------------

Устанавливаем/обновляем необходимые пакеты:
```shell
apt update
apt install apparmor bluez cifs-utils curl dbus jq libglib2.0-bin lsb-release network-manager nfs-common systemd-journal-remote systemd-resolved udisks2 wget -y
```

------------


Далее устанавливаем докер:
```shell
curl -fsSL get.docker.com | sh
```
... и внезапно обнаруживаем, что отвалился резольвер (на самом деле, установился другой, `systemd-resolved`):
```
root@homeassistant:~# curl -fsSL get.docker.com | sh
curl: (6) Could not resolve host: get.docker.com
```
Редактируем файл `nano /etc/systemd/resolved.conf` раскомментировав строку и добавив DNS-сервер роутера, `1.1.1.1` и/или `8.8.8.8`. Далее перезагружаем сервис или систему в целом (чтобы наверняка) и проверяем:
```
root@homeassistant:~# systemctl restart systemd-resolved.service
root@homeassistant:~# resolvectl dns
Global: 192.168.1.1
Link 2 (ens192):
root@homeassistant:~# ping4 google.com
PING  (74.125.131.102) 56(84) bytes of data.
64 bytes from lu-in-f102.1e100.net (74.125.131.102): icmp_seq=1 ttl=107 time=29.8 ms
64 bytes from lu-in-f102.1e100.net (74.125.131.102): icmp_seq=2 ttl=107 time=29.8 ms
64 bytes from lu-in-f102.1e100.net (74.125.131.102): icmp_seq=3 ttl=107 time=30.0 ms
^C
```
Отлично! Повторяем попытку установки докера.

------------

Далее нам нужно установить OS-Agent. Для этого скачиваем его и устанавливаем:
```shell
wget https://github.com/home-assistant/os-agent/releases/download/1.6.0/os-agent_1.6.0_linux_x86_64.deb
dpkg -i os-agent_1.6.0_linux_x86_64.deb
```
Проверяем его работоспособность:
```shell
gdbus introspect --system --dest io.hass.os --object-path /io/hass/os
```
Если нет ошибок, нормально выводятся объекты `interface`, то все установилось корректно.

------------

