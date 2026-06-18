Задание №1
Настройка имён хостов

hostnamectl set-hostname isp.au-team.irpo; exec bash

hostnamectl set-hostname hq-rtr.au-team.irpo; exec bash

hostnamectl set-hostname hq-srv.au-team.irpo; exec bash

hostnamectl set-hostname hq-cli.au-team.irpo; exec bash

hostnamectl set-hostname br-rtr.au-team.irpo; exec bash

hostnamectl set-hostname br-srv.au-team.irpo; exec bash


Настройка IP адресов
===ИНТЕРФЕЙСЫ ISP===

mkdir -p /etc/net/ifaces/enp7s{2,3}

echo 'TYPE=eth' | tee /etc/net/ifaces/enp7s{2,3}/options
echo '172.16.50.1/28' > /etc/net/ifaces/enp7s2/ipv4address
echo '172.16.60.1/28' > /etc/net/ifaces/enp7s3/ipv4address

systemctl restart network

ip -c --br a
=======================



===ИНТЕРФЕЙСЫ BR-RTR===

mkdir -p /etc/net/ifaces/{enp7s{1,2},gre1}

echo 'TYPE=eth' | tee /etc/net/ifaces/enp7s{1,2}/options

-------to ISP------

echo '172.16.60.2/28' > /etc/net/ifaces/enp7s1/ipv4address

echo 'default via 172.16.60.1' > /etc/net/ifaces/enp7s1/ipv4route

echo 'nameserver 8.8.8.8' > /etc/net/ifaces/enp7s1/resolv.conf

-------to BR-SRV---

echo '192.168.0.1/28' > /etc/net/ifaces/enp7s2/ipv4address

-------включение маршрутизации---------

vim /etc/net/sysctl.conf

systemctl restart network

ip -c --br a
=======================










===ИНТЕРФЕЙСЫ BR-SRV===

echo 'TYPE=eth' > /etc/net/ifaces/enp7s1/options

echo '192.168.0.2/28' > /etc/net/ifaces/enp7s1/ipv4address

echo 'default via 192.168.0.1' > /etc/net/ifaces/enp7s1/ipv4route

echo $'search au-team.irpo\nnameserver 192.168.100.2' > /etc/net/ifaces/enp7s1/resolv.conf

systemctl restart network

ip -c --br a
=========================


===ИНТЕРФЕЙСЫ HQ-RTR===
mkdir -p /etc/net/ifaces/{enp7s{1,2},vlan{113,213,813},gre1}

echo 'TYPE=eth' | tee /etc/net/ifaces/enp7s{1,2}/options

-------to ISP------

echo '172.16.50.2/28' > /etc/net/ifaces/enp7s1/ipv4address

echo 'default via 172.16.50.1' > /etc/net/ifaces/enp7s1/ipv4route

echo 'nameserver 8.8.8.8' > /etc/net/ifaces/enp7s1/resolv.conf

--------настройка VLAN-----------

echo $'113\n213\n813' | xargs -i bash -c 'echo -e "TYPE=vlan\nHOST=enp7s2\nVID={}" > /etc/net/ifaces/vlan{}/options'

cat /etc/net/ifaces/vlan813/options

echo '192.168.100.1/27' > /etc/net/ifaces/vlan113/ipv4address

echo '192.168.200.1/24' > /etc/net/ifaces/vlan213/ipv4address

echo '192.168.99.1/29' > /etc/net/ifaces/vlan813/ipv4address

-------включение маршрутизации---------

vim /etc/net/sysctl.conf

systemctl restart network

ip -c --br a
=========================






===ИНТЕРФЕЙСЫ HQ-SRV===

echo 'TYPE=eth' > /etc/net/ifaces/enp7s1/options

echo '192.168.100.2/27' > /etc/net/ifaces/enp7s1/ipv4address

echo 'default via 192.168.100.1' > /etc/net/ifaces/enp7s1/ipv4route

echo 'nameserver 8.8.8.8' > /etc/net/ifaces/enp7s1/resolv.conf

systemctl restart network

ip -c --br a
=========================








Задание №2
Выполняется на ISP

vim /etc/net/sysctl.conf
net.ipv4.ip_forward = 1

systemctl restart network

apt-get update && apt-get install nftables nano -y

nano /etc/nftables/nftables.nft

==
#!/usr/sbin/nft -f

flush ruleset

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat;
        oifname "enp7s1" masquerade
    }
}
==

systemctl enable --now nftables
systemctl start nftables
nft flush ruleset
nft -f /etc/nftables/nftables.nft


nft list ruleset
Команда должна вывести текст из предыдущего действия.

sysctl net.ipv4.ip_forward

ping -c4 ya.ru 

==========HQ-SRV==========

useradd -u 2013 sshuser

echo "sshuser:P@ssw0rd" | chpasswd

usermod -aG wheel sshuser

echo "WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/sshuser

su -l sshuser

sudo id

exit



==========BR-SRV==========

useradd -u 2013 sshuser

echo "sshuser:P@ssw0rd" | chpasswd

usermod -aG wheel sshuser

echo "WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/sshuser

su -l sshuser

sudo id

exit

==========HQ-RTR==========

useradd net_admin

echo "net_admin:P@ssw0rd" | chpasswd

usermod -aG wheel net_admin

apt-get update && apt-get install sudo -y

echo "WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/net_admin

su -l net_admin

sudo id

exit

==========BR-RTR==========

useradd net_admin

echo "net_admin:P@ssw0rd" | chpasswd

usermod -aG wheel net_admin

apt-get update && apt-get install sudo

echo "WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/net_admin

su -l net_admin

sudo id

=====HQ-SRV=====

vim /etc/openssh/sshd_config

Port 2013
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/openssh/ssh_banner


echo "Authorized access only" | tee /etc/openssh/ssh_banner

Перезагружаем SSH-сервис:

systemctl restart sshd

Проверка:
С машины HQ-RTR подключаемся по SSH к машине HQ-SRV
ssh -p 2013 sshuser@192.168.100.2

exit

=====BR-SRV=====

vim /etc/openssh/sshd_config

Port 2013
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/openssh/ssh_banner


echo "Authorized access only" | tee /etc/openssh/ssh_banner

Перезагружаем SSH-сервис:

systemctl restart sshd

Проверка:
С машины BR-RTR подключаемся по SSH к машине BR-SRV
ssh -p 2013 sshuser@192.168.0.2

exit

===BR-RTR===

vim /etc/net/ifaces/gre1/options

TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.60.2
TUNREMOTE=172.16.50.2
TUNTTL=64
TUNOPTIONS='ttl 64'

echo "10.10.10.2/30" > /etc/net/ifaces/gre1/ipv4address
systemctl restart network

ip -br -c a

===HQ-RTR===

vim /etc/net/ifaces/gre1/options

TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.50.2
TUNREMOTE=172.16.60.2
TUNTTL=64
TUNOPTIONS='ttl 64'

echo "10.10.10.1/30" > /etc/net/ifaces/gre1/ipv4address
systemctl restart network

ip -br -c a

ping 10.10.10.2 -c 3

===HQ-RTR===
apt-get update && apt-get install frr -y

sed -i 's/ospfd=no/ospfd=yes/' /etc/frr/daemons ; grep ospf /etc/frr/daemons

vim /etc/frr/frr.conf

interface gre
 no ip ospf passive
exit
!
interface gre1
 ip ospf area 0
 ip ospf authentication
 ip ospf authentication-key P@ssw0rd
 no ip ospf passive
exit
!
interface vlan113
 ip ospf area 0
exit
!
interface vlan213
 ip ospf area 0
exit
!
interface vlan813
 ip ospf area 0
exit
!
router ospf
 passive-interface default
exit


systemctl enable --now frr

systemctl restart network




===BR-RTR===
apt-get update && apt-get install frr -y

sed -i 's/ospfd=no/ospfd=yes/' /etc/frr/daemons ; grep ospf /etc/frr/daemons

vim /etc/frr/frr.conf


interface gre
 no ip ospf passive
exit
!
interface gre1
 ip ospf area 0
 ip ospf authentication
 ip ospf authentication-key P@ssw0rd
 no ip ospf passive
exit
!
interface enp7s2
 ip ospf area 0
exit
!
router ospf
 passive-interface default
exit


systemctl enable --now frr

systemctl restart network

===HQ-RTR + BR-RTR===

apt-get install nftables nano -y

nano /etc/nftables/nftables.nft

#!/usr/sbin/nft -f
flush ruleset

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat;
        oifname "enp7s1" masquerade
    }
}

systemctl enable --now nftables

systemctl restart nftables.service

===HQ-RTR===

apt-get install dnsmasq -y

--------смена DNS-----------

rm -f /etc/net/ifaces/enp7s1/resolv.conf

echo $'search au-team.irpo\nnameserver 192.168.100.2' > /etc/net/ifaces/vlan113/resolv.conf

--------DHCP------------

sed -i 's/AUTO_LOCAL_RESOLVER=yes/AUTO_LOCAL_RESOLVER=no/' /etc/sysconfig/dnsmasq ; grep AUTO_LOCAL_RESOLVER /etc/sysconfig/dnsmasq

nano /etc/dnsmasq.conf

port=0
interface=vlan213
listen-address=192.168.200.1
dhcp-authoritative
dhcp-range=interface:vlan213,192.168.200.2,192.168.200.2,255.255.255.240,6h
dhcp-option=3,192.168.200.1
dhcp-option=6,192.168.100.2
leasefile-ro





systemctl enable --now dnsmasq ; ss -lun | grep 67

systemctl restart network

cat /etc/resolv.conf
===HQ-SRV===

apt-get update && apt-get install bind bind-utils nano -y

echo $'search au-team.irpo\nnameserver 127.0.0.1' > /etc/net/ifaces/enp7s1/resolv.conf

rndc-confgen -a -c /etc/bind/rndc.key


nano /etc/bind/options.conf

options {
    listen-on { 127.0.0.1; 192.168.100.2; };
    forwarders { 77.88.8.7; 77.88.8.3; };
    recursion yes;
    allow-recursion { any; };
    allow-query { any; };
    dnssec-validation no;
    directory "/etc/bind/zone";
    dump-file "/var/run/named/named_dump.db";
    statistics-file "/var/run/named/named.stats";
    pid-file "/var/run/named/named.pid";
};

logging {
    category default { default_syslog; };
};

zone "au-team.irpo" {
    type master;
    file "au-team.irpo";
};

zone "168.192.in-addr.arpa" {
    type master;
    file "168.192.in-addr.arpa";
};





==========================================
cp -r /etc/bind/zone/127.in-addr.arpa /etc/bind/zone/au-team.irpo

nano /etc/bind/zone/au-team.irpo

$TTL 1D
@ IN SOA au-team.irpo. root.au-team.irpo. (
	2025020600
    12H
    1H
    1W
    1H
)
@       IN NS    hq-srv.au-team.irpo.

hq-rtr  IN A     192.168.100.1
hq-srv  IN A     192.168.100.2
hq-cli  IN A     192.168.200.2
br-rtr  IN A     192.168.0.1
br-srv  IN A     192.168.0.2
docker  IN A     172.16.50.1
web     IN A     172.16.60.1






==========================================
cp -r /etc/bind/zone/127.in-addr.arpa /etc/bind/zone/168.192.in-addr.arpa

nano /etc/bind/zone/168.192.in-addr.arpa

$TTL 1D
@ IN SOA au-team.irpo. root.au-team.irpo. (
    2025020600
    12H
    1H
    1W
    1H
)
@       IN NS    au-team.irpo.

1.100   IN PTR   hq-rtr.au-team.irpo.
2.100   IN PTR   hq-srv.au-team.irpo.
2.200   IN PTR   hq-cli.au-team.irpo.





==========================================

chown :named /etc/bind/zone/au-team.irpo /etc/bind/zone/168.192.in-addr.arpa

systemctl enable --now bind

service network restart

systemctl restart bind.service

host br-rtr

host -t PTR 192.168.100.2



apt-get update && apt-get install tzdata -y

timedatectl set-timezone Asia/Novosibirsk

timedatectl


Стенд Модуля 2 установленный из скрипта имеет недостаток,
он проявляется при выполнении Задания №5

Вам необходимо самостоятельно включить SSH доступ на машинах
HQ-RTR
BR-RTR
HQ-SRV
HQ-CLI

Для этого на машинах выполняем следующие действия

Редактируем файл sshd_config

vim /etc/openssh/sshd_config

Добавляем одну строчку в самое начало
Port 2013

systemctl enable --now sshd
=========BR-SRV=========

------------SambaDC------------
echo "nameserver 192.168.1.10" >> /etc/net/ifaces/enp7s1/resolv.conf

systemctl restart network

cat /etc/resolv.conf

ping ya.ru -c 2

apt-get update && apt-get install task-samba-dc -y

rm -f /etc/samba/smb.conf

rm -rf {/var/lib/samba, /var/cache/samba}

mkdir -p /var/lib/samba/sysvol

samba-tool domain provision

"Везде соглашаемся со значениями по умолчанию (Жмём Enter)"
"Пароль задаём: P@ssw0rd"

mv /etc/krb5.conf /etc/krb5.conf.back

cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

systemctl enable --now samba

systemctl status samba

samba-tool domain info 127.0.0.1

================================
------------DNS------------

samba-tool dns add br-srv.au-team.irpo au-team.irpo hq-srv A 192.168.1.10 -U Administrator

samba-tool dns add br-srv.au-team.irpo au-team.irpo hq-rtr A 192.168.1.1 -U Administrator

samba-tool dns add br-srv.au-team.irpo au-team.irpo br-rtr A 192.168.3.1 -U Administrator

samba-tool dns add br-srv.au-team.irpo au-team.irpo web.au-team.irpo A 172.16.1.1 -U Administrator

samba-tool dns add br-srv.au-team.irpo au-team.irpo docker.au-team.irpo A 172.16.2.1 -U Administrator

samba-tool dns query br-srv.au-team.irpo au-team.irpo @ ALL -U administrator

sed -i 's/nameserver 192.168.1.10/nameserver 127.0.0.1/' /etc/net/ifaces/enp7s1/resolv.conf 

systemctl restart network

cat /etc/resolv.conf

kinit administrator@AU-TEAM.IRPO

================================
------------Users------------

samba-tool group add hq

for i in {1..5}; do samba-tool user add hquser$i P@ssw0rd; done

for i in {1..5}; do samba-tool group addmembers hq hquser$i; done

samba-tool group listmembers hq

================================

=========HQ-RTR=========
sed -i 's/192.168.1.10/192.168.3.10/' /etc/dnsmasq.conf; systemctl restart dnsmasq

cat /etc/dnsmasq.conf

=========HQ-CLI=========
!!!Работаем в графической оболочке HQ-CLI (под user)!!!

на HQ-CLI перезагружаем сеть, проверяем DNS, вводим в домен

acc

Пользователи (Аутентификация) -> Галочка (Домен Active Directory) -> Применить -> Пароль P@ssw0rd
Если настроено всё верно, появится окно (Добро пожаловать в домен AU-TEAM.IRPO)
Перезагружаем машину

Меню -> Выйти -> Перезагрузить

Пользователь: hquser1
Пароль: P@ssw0rd

!!!Работаем в консольной оболочке HQ-CLI (под root)!!!

control libnss-role 
(Должна выдать enabled)

roleadd hq wheel

echo "WHEEL_USERS ALL=(ALL:ALL) /bin/cat, /bin/grep, /usr/bin/id" >> /etc/sudoers

tail /etc/sudoers

!!!Работаем в графической оболочке HQ-CLI (под user)!!!

Перезагружаем машину

Меню -> Выйти -> Перезагрузить

Пользователь: hquser1
Пароль: P@ssw0rd

Открываем терминал и проверяем команду

sudo id
*вводим пароль*
sudo cat /etc/resolv.conf
*вводим пароль*
sudo grep
*вводим пароль*
=====HQ-SRV=====

==================RAID==================
lsblk

parted /dev/sdb
	mklabel msdos
	mkpart primary 1MiB 100%
	set 1 raid on
	print
	select /dev/sdc
	mklabel msdos
	mkpart primary 1MiB 100%
	set 1 raid on
	print
	select /dev/sdd
	mklabel msdos
	mkpart primary 1MiB 100%
	set 1 raid on
	print
	quit

mdadm --create /dev/md3 --level=5 --raid-devices=3 /dev/sdb1 /dev/sdc1 /dev/sdd1

echo "DEVICE partitions" > /etc/mdadm.conf

mdadm --detail --scan --verbose >> /etc/mdadm.conf

mkfs.ext4 /dev/md3

mkdir -p /raid

cp /etc/fstab /etc/fstab.bak

echo "/dev/md3 /raid ext4 defaults 0 0" >> /etc/fstab

mount -a

df -Th

==================NFS==================
apt-get update && apt-get install nfs-server nfs-utils -y

mkdir /raid/nfs

chmod 777 /raid/nfs

cp /etc/exports /etc/exports.back

echo "/raid/nfs 192.168.2.0/27(rw,no_subtree_check,no_root_squash)" >> /etc/exports

systemctl enable --now nfs-server

==================HQ-CLI==================
!!!Работаем в консольной оболочке HQ-CLI (под root)!!!

mkdir /mnt/nfs

chmod -R 777 /mnt/nfs

showmount -e hq-srv

cp /etc/fstab /etc/fstab.back

echo "192.168.1.10:/raid/nfs /mnt/nfs nfs rw,soft,_netdev 0 0 " >> /etc/fstab

mount -av

df -T

---создать файл на HQ-CLI---

touch /mnt/nfs/test_file

ls -l /mnt/nfs/test_file

---посмотреть на HQ-SRV---

ls -l /raid/nfs

=====ISP=====

control chrony server

sed -i 's/pool pool.ntp.org iburst/pool pool.ntp.org iburst prefer minstratum 4/' /etc/chrony.conf | grep pool /etc/chrony.conf

sed -i 's/\#local stratum 10/local stratum 8/' /etc/chrony.conf | grep "local stratum" /etc/chrony.conf

systemctl restart chronyd
===============================



===Далее команда выполняется на машинах HQ-SRV, BR-RTR и BR-SRV===

sed -i 's/pool pool.ntp.org iburst/server 172.16.1.1 iburst/' /etc/chrony.conf && systemctl restart chronyd

---Через 10 секунд проверяем---

chronyc sources
===============================




-==На HQ-CLI выполняем другую команду===

echo "server 172.16.1.1 iburst" >> /etc/chrony.conf && systemctl restart chronyd

---Через 10 секунд проверяем---

chronyc sources


---Вывод команды (chronyc sources) должен быть приблизительно следующим---

MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* 172.16.1.1                    5   6    17     1  +1438ns[  +43us] +/-   38ms
===BR-SRV===

apt-get install ansible sshpass -y

cp -r /etc/ansible/ansible.cfg /etc/ansible/ansible.cfg.back

rm -rf /etc/ansible/ansible.cfg

vim /etc/ansible/ansible.cfg

[defaults]

host_key_checking = False
interpreter_python=/usr/bin/python3
inventory       = /etc/ansible/hosts

=====================================

vim /etc/ansible/hosts

HQ-SRV ansible_user=user ansible_password=resu ansible_port=2013
HQ-RTR ansible_user=net_admin ansible_password=P@ssw0rd ansible_port=2013
BR-RTR ansible_user=net_admin ansible_password=P@ssw0rd ansible_port=2013
HQ-CLI ansible_user=user ansible_password=resu ansible_port=2013

=====================================

Проверить состояние службы SSH на хостах
(все должны ответить "pong")

ansible all -m ping
===BR-SRV===

apt-get install docker-engine docker-compose-v2 -y

systemctl enable --now docker.service

mount -o loop /dev/sr0 /mnt/ -v

ls -l /mnt/docker/

cat /mnt/docker/readme.txt

docker load < /mnt/docker/site_latest.tar

docker load < /mnt/docker/mariadb_latest.tar

docker image ls


===docker-compose.yml===

cat << EOF > docker-compose.yml
services:
  database:
    container_name: db
    image: mariadb:latest
    restart: always
    ports: 
      - "3306:3306"
    environment:
      MARIADB_DATABASE: testdb3
      MARIADB_USER: test3c
      MARIADB_PASSWORD: P@ssw0rd
      MARIADB_ROOT_PASSWORD: P@ssw0rd
    volumes:
      - db_data:/var/lib/mysql
      
  app:
    container_name: site
    image: site:latest
    restart: always
    ports: 
      - "8083:8000"
    environment: 
      DB_HOST: database
      DB_PORT: 3306
      DB_NAME: testdb3
      DB_USER: test3c
      DB_PASS: P@ssw0rd
      DB_TYPE: maria
    depends_on: 
      - database
volumes:
  db_data:
EOF

==============
docker compose config

docker compose up -d

docker ps

ss -ltnp4 | grep 8083

Переходим на HQ-CLI, заходим по 192.168.3.10:8083

Пометка: По заданию мы должны попадать на сайт через домен
docker.au-team.irpo, но мы пока этого сделать не можем,
т.к. у нас не настроен реверс-прокси** на ISP (nginx)
**Выполняется в задании №9

Cоздаем запись

docker rm -f $(docker ps -qa)

Проверяем, что сайт перестал работать

docker compose up -d

Проверяем, что сайт поднялся и наша запись осталась
===HQ-SRV===

mount -o loop /dev/sr0 /mnt/ -v

apt-get install lamp-server -y

cp /mnt/web/index.php /var/www/html

cp /mnt/web/logo.png /var/www/html

systemctl enable --now mariadb
======================================

mariadb -e "CREATE DATABASE webdb;"

mariadb -e "

CREATE USER 'web3'@'localhost' IDENTIFIED BY 'P@ssw0rd';

GRANT ALL PRIVILEGES ON webdb.* TO 'web3'@'localhost';

"

mariadb webdb < /mnt/web/dump.sql

mariadb -e "USE webdb; SHOW TABLES;"
=====================================

vim /var/www/html/index.php

$servername = "localhost";

$username = "web3";

$password = "P@ssw0rd";

$dbname = "webdb";

systemctl enable --now httpd2.service
============================================

Переходим на HQ-CLI, заходим по 192.168.1.10

Пометка: По заданию мы должны попадать на сайт через домен
web.au-team.irpo + аутенфикация по паролю,
но мы пока этого сделать не можем,
т.к. у нас не настроен реверс-прокси** на ISP (nginx)
**Выполняется в задании №9

===HQ-RTR===

nft add chain nat prerouting { type nat hook prerouting priority dstnat \; }

nft add rule nat prerouting iif "enp7s1" tcp dport 2013 dnat to 192.168.1.10

nft add rule nat prerouting iif "enp7s1" tcp dport 8083 dnat to 192.168.1.10:80

nft list ruleset

nft list ruleset > /etc/nftables/nftables.nft

systemctl restart nftables

nft list ruleset
================================

===BR-RTR===

nft add chain nat prerouting { type nat hook prerouting priority dstnat \; }

nft add rule nat prerouting iif "enp7s1" tcp dport { 8083, 2013 } dnat to 192.168.3.10

nft list ruleset

nft list ruleset > /etc/nftables/nftables.nft

systemctl restart nftables

nft list ruleset
===ISP===

apt-get update && apt-get install nginx -y

vim /etc/nginx/sites-available.d/r-proxy.conf

====
server {
listen 80;
server_name web.au-team.irpo;
location / {
proxy_pass http://172.16.1.10:8083;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
auth_basic "Restricted Access";
auth_basic_user_file /etc/nginx/.htpasswd;
}
}
server {
listen 80;
server_name docker.au-team.irpo;
location / {
proxy_pass http://172.16.2.10:8083;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
}
}
====

ln -s /etc/nginx/sites-available.d/r-proxy.conf /etc/nginx/sites-enabled.d/

nginx -t

systemctl enable --now nginx

systemctl status nginx

=======================================

apt-get install apache2-htpasswd -y

htpasswd -c /etc/nginx/.htpasswd Kazimir
# Вводим пароль P@ssw0rd (2 раза)

cat /etc/nginx/.htpasswd

nginx -t

systemctl restart nginx


=======================================
Переходим на HQ-CLI, заходим по http://web.au-team.irpo/
Логин: Kazimir
Пароль: P@ssw0rd

Дальше заходим по http://docker.au-team.irpo/

Если всё открывается, значит задание выполнено верно.

===HQ-CLI===

apt-get update && apt-get install yandex-browser-stable -y

• Установку браузера отметьте в отчёте.
