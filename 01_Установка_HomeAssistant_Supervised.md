# Установка Home Assistant Supervised

Будем ставить на Debian 12. Почему? Потому что. Можете ставить и на другие дистрибютивы, но нет гарантии что все пройдет по плану и ничего не сломается.

------------


Нам понадобится собственно сам [дистрибутив с официального сайта](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-12.5.0-amd64-netinst.iso "дистрибутив с официального сайта"). Если ставите напрямую на ПК, то пишем его на флешку через [Rufus](https://rufus.ie/ru/ "Rufus") или любыми другими способами, или используем как есть если ставите на виртуалку. ПК/виртуалка должна подключаться к роутеру через DHCP, так проще будет настраивать.

------------

Устанавливаем Debian все по стандарту, без графической оболочки, без всяких излишеств. При установке выбираем только `SSH server`. Статичный адрес после установки не спешим настраивать, оставляем по умолчанию DHCP, настройки сети все равно слетят во время установки.

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
```
```shell
apt install apparmor bluez cifs-utils curl dbus jq libglib2.0-bin lsb-release network-manager nfs-common systemd-journal-remote systemd-resolved udisks2 wget -y
```

------------


Далее устанавливаем докер:
```shell
curl -fsSL get.docker.com | sh
```
... и внезапно обнаруживаем, что отвалился резольвер (на самом деле, установился другой: `systemd-resolved`):
```
root@homeassistant:~# curl -fsSL get.docker.com | sh
curl: (6) Could not resolve host: get.docker.com
```
Проверяем:
```
root@homeassistant:~# resolvectl dns
Global:
Link 2 (ens192):
```
Сразу не работает, нужно прежде всего перезапустить `systemctl restart systemd-resolved.service`.

Отсутствуют DNS-сервера на интерфейсах.

Добавляем на сетевой интерфейс DNS-сервер роутера, `1.1.1.1` и/или `8.8.8.8` и проверяем:
```
root@homeassistant:~# resolvectl dns ens192 192.168.1.1
root@homeassistant:~# resolvectl dns
Global:
Link 2 (ens192): 192.168.1.1
root@homeassistant:~# ping4 google.com
PING  (74.125.131.100) 56(84) bytes of data.
64 bytes from lu-in-f100.1e100.net (74.125.131.100): icmp_seq=1 ttl=107 time=30.0 ms
64 bytes from lu-in-f100.1e100.net (74.125.131.100): icmp_seq=2 ttl=107 time=30.0 ms
64 bytes from lu-in-f100.1e100.net (74.125.131.100): icmp_seq=3 ttl=107 time=29.8 ms
^C
```
Отлично! Повторяем попытку установки докера.

### Внимание! После перезагрузки обязательно перепроверять, настройки могут слететь!

------------

Далее нам нужно установить OS-Agent. Для этого скачиваем его и устанавливаем:
```shell
wget https://github.com/home-assistant/os-agent/releases/download/1.6.0/os-agent_1.6.0_linux_x86_64.deb
```
```shell
dpkg -i os-agent_1.6.0_linux_x86_64.deb
```
Проверяем его работоспособность:
```
root@debian:~# gdbus introspect --system --dest io.hass.os --object-path /io/hass/os
node /io/hass/os {
  interface org.freedesktop.DBus.Introspectable {
    methods:
      Introspect(out s out);
    signals:
    properties:
  };
  interface org.freedesktop.DBus.Properties {
    methods:
      Get(in  s interface,
          in  s property,
          out v value);
      GetAll(in  s interface,
             out a{sv} props);
      Set(in  s interface,
          in  s property,
          in  v value);
    signals:
      PropertiesChanged(s interface,
                        a{sv} changed_properties,
                        as invalidates_properties);
    properties:
  };
  interface io.hass.os {
    methods:
    signals:
    properties:
      @org.freedesktop.DBus.Property.EmitsChangedSignal("invalidates")
      readonly s Version = '1.6.0';
      @org.freedesktop.DBus.Property.EmitsChangedSignal("true")
      readwrite b Diagnostics = false;
  };
};
```
Если нет ошибок, нормально выводятся объекты `interface`, то все установилось корректно.

------------

Далее скачиваем и устанавливаем Home Assistant:
```shell
wget -O homeassistant-supervised.deb https://github.com/home-assistant/supervised-installer/releases/latest/download/homeassistant-supervised.deb
```
```shell
apt install ./homeassistant-supervised.deb
```
После установки очень желательно машину перезагрузить, и скорее всего адрес поменяется. Зайти в консоль и посмотреть текущий адрес с помощью команды `ip -4 a`. Далее через 2-3 минуты (или дольше) заходим по адресу http://IP_ADDRESS:8123/ (где IP_ADDRESS - новый IP-адрес машины), ждем завершения и настраиваем Home Assistant.

![](https://github.com/GennPen/HomeAssistant/blob/main/images/01%20-%202024-06-01%20012114.jpg)

Если в процессе установки отвалился резольвер и в консоль идут бесконечные ошибки:
```
ping: checkonline.home-assistant.io: Temporary failure in name resolution
[info] Waiting for checkonline.home-assistant.io - network interface might be down...
ping: checkonline.home-assistant.io: Temporary failure in name resolution
[info] Waiting for checkonline.home-assistant.io - network interface might be down...
```
Не завершая сеанс установки! Заходим в параллельный сеанс (Alt-F2 если не через SSH, а напрямую) и прописываем DNS-сервер на сетевой интерфейс (выше описано как). Установка сама продолжится (вернуться в первый сеанс - Alt-F1).

------------

### Важно, для жителей РФ!
Т.к. Home Assistant работает на докере, который успешно заблокировал доступ, нужно прописать зеркала. Подробности: https://huecker.io
