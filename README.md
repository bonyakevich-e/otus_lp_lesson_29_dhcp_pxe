### OTUS Linux Professional Lesson #29 | Subject: DHCP. PXE.

#### Цель домашнего задания
Отработать навыки установки и настройки DHCP, TFTP, PXE загрузчика и автоматической загрузки

#### Описание домашнего задания
1. Настроить загрузку по сети дистрибутива Ubuntu 24
2. Установка должна проходить из HTTP-репозитория.
3. Настроить автоматическую установку c помощью файла user-data
4. *Настроить автоматическую загрузку по сети дистрибутива Ubuntu 24 c использованием UEFI

#### Инструкция по выполнению домашнего задания

__1. Настройка DHCP и TFTP-сервера__

Подготовим Vagrantfile в котором будут описаны 2 виртуальные машины:

• pxeserver (хост к которому будут обращаться клиенты для установки ОС)

• pxeclient (хост, на котором будет проводиться установка)

Создаем виртуальные машины:
```
$ vagrant up
```
Для того, чтобы клиент мог получить ip-адрес нам требуется DHCP-сервер. Для того, чтобы клиент мог получить файл pxelinux.0 нам потребуется TFTP-сервер. Утилита dnsmasq совмещает в себе сразу и DHCP и TFTP-сервер.
Настраиваем сервер. Отключаем firewall:
```
root@pxeserver:~# systemctl stop ufw
root@pxeserver:~# systemctl disable ufw
```
Обновляем apt кэш и устанавливаем dnsmasq:
```
root@pxeserver:~# apt update
root@pxeserver:~# apt install dnsmasq
```
Создаём файл /etc/dnsmasq.d/pxe.conf и добавляем в него следующее содержимое:
```bash
# интерфейс на котором будет работать DHCP/TFTP
interface=eth1
bind-interfaces
# интерфейс и range адресов которые будут выдаваться по DHCP
dhcp-range=eth1,10.0.0.100,10.0.0.120
# имя файла, с которого надо начинать загрузку для Legacy boot 
dhcp-boot=pxelinux.0
# имена файлов для UEFI-загрузки
dhcp-match=set:efi-x86_64,option:client-arch,7
dhcp-boot=tag:efi-x86_64,bootx64.efi
# включаем TFTP-сервер
enable-tftp
# указываем каталог для TFTP-сервера
tftp-root=/srv/tftp/amd64
```
Создаём каталоги для файлов TFTP-сервера:
```
root@pxeserver:~# mkdir -p /srv/tftp
```
Cкачиваем файлы для сетевой установки Ubuntu 24.04 и распаковываем их в каталог /srv/tftp:
```
root@pxeserver:~# wget https://releases.ubuntu.com/noble/ubuntu-24.04-netboot-amd64.tar.gz
root@pxeserver:~# tar -xzvf ubuntu-24.04-netboot-amd64.tar.gz  -C /srv/tftp
```
В каталоге видим следующие файлы:
```
root@pxeserver:~# tree /srv/tftp/
/srv/tftp/
└── amd64
    ├── bootx64.efi
    ├── grub
    │   └── grub.cfg
    ├── grubx64.efi
    ├── initrd
    ├── ldlinux.c32
    ├── linux
    ├── pxelinux.0
    └── pxelinux.cfg
        └── default
```
Перезапускаем службу dnsmasq:
```
root@pxeserver:~# systemctl restart dnsmasq
```

__2. Настройка Web-сервера__

Для того, чтобы отдавать файлы по HTTP нам потребуется настроенный веб-сервер.

Устанавливаем Web-сервер apache2:
```
root@pxeserver:~# apt install apache2
```
Cоздаём каталог /srv/images в котором будут храниться iso-образы для установки по сети:
```
root@pxeserver:~# mkdir /srv/images
```
Переходим в каталог /srv/images и скачиваем iso-образ ubuntu 24.04:
```
root@pxeserver:/srv/images# wget https://releases.ubuntu.com/noble/ubuntu-24.04-live-server-amd64.iso
```
Cоздаём файл /etc/apache2/sites-available/ks-server.conf и добавлем в него следующее содержимое:
```apache
#Указываем IP-адрес хоста и порт на котором будет работать Web-сервер
<VirtualHost 10.0.0.20:80>

  DocumentRoot /

  # Указываем директорию /srv/images из которой будет загружаться iso-образ
  <Directory /srv/images>
    Options Indexes MultiViews
    AllowOverride All
    Require all granted
  </Directory>

</VirtualHost>
```
Активируем конфигурацию ks-server в apache:
```
root@pxeserver:~# a2ensite ks-server.conf
```
Вносим изменения в файл /srv/tftp/amd64/pxelinux.cfg/default:
```
DEFAULT install
LABEL install
KERNEL linux
INITRD initrd
APPEND root=/dev/ram0 ramdisk_size=3000000 ip=dhcp iso-url=http://10.0.0.20/srv/images/ubuntu-24.04-live-server-amd64.iso autoinstall
```
В данном файле мы указываем что файлы linux и initrd будут забираться по tftp, а сам iso-образ ubuntu 24.04 будет скачиваться из нашего веб-сервера. 

Из-за того, что образ достаточно большой (2.6G) и он сначала загружается в ОЗУ, необходимо указать размер ОЗУ до 3 гигабайт (root=/dev/ram0 ramdisk_size=3000000).

Перезагружаем web-сервер apache:
```
systemctl restart apache2
```

__3. Настройка автоматической установки Ubuntu 24.04__
Создаём каталог для файлов с автоматической установкой:
```
root@pxeserver:~# mkdir /srv/ks
```
Cоздаём файл _/srv/ks/user-data_ и добавляем в него следующее содержимое:
```
#cloud-config
autoinstall:
apt:
disable_components: []
geoip: true
preserve_sources_list: false
primary:
- arches:
- amd64
- i386
uri: http://us.archive.ubuntu.com/ubuntu
- arches:
- default
uri: http://ports.ubuntu.com/ubuntu-ports
drivers:
install: false
identity:
hostname: linux
password: $6$sJgo6Hg5zXBwkkI8$btrEoWAb5FxKhajagWR49XM4EAOfO/
Dr5bMrLOkGe3KkMYdsh7T3MU5mYwY2TIMJpVKckAwnZFs2ltUJ1abOZ.
realname: otus
username: otus
kernel:
package: linux-generic
keyboard:
layout: us
toggle: null
variant: ''
locale: en_US.UTF-8
network:
ethernets:
enp0s3:
dhcp4: true
enp0s8:
dhcp4: true
version: 2
ssh:
allow-pw: true
authorized-keys: []
install-server: true
updates: security
version: 1
```
Cоздаём файл с метаданными _/srv/ks/meta-data_:
```
root@pxeserver:~# touch /srv/ks/meta-data
```
Файл с метаданными хранит дополнительную информацию о хосте, мы сейчас не будем добавлять дополнительную информацию.

В конфигурации веб-сервера добавим каталог /srv/ks идёнтично каталогу /srv/images:
```
root@pxeserver:~# vim /etc/apache2/sites-available/ks-server.conf

...
...
  <Directory /srv/ks>
    Options Indexes MultiViews
    AllowOverride All
    Require all granted
  </Directory>
...
...
```
В файле _/srv/tftp/amd64/pxelinux.cfg/default_ добавляем параметры автоматической установки:
```
DEFAULT install
LABEL install
KERNEL linux
INITRD initrd
APPEND root=/dev/ram0 ramdisk_size=3000000 ip=dhcp iso-url=http://10.0.0.20/srv/images/ubuntu-24.04-live-server-amd64.iso autoinstall __ds=nocloud-net;s=http://10.0.0.20/srv/ks/__
```
Перезапускаем службы dnsmasq и apache2:
```
root@pxeserver:~# systemctl restart dnsmasq
root@pxeserver:~# systemctl restart apache2
```
На этом настройка автоматической установки завершена. Теперь можно перезапустить ВМ pxeclient и мы должны увидеть автоматическую установку. После успешной установки выключаем ВМ и в её настройках ставим запуск ВМ из диска. После запуска нашей ВМ мы сможем залогиниться под пользователем otus.
