Модуль 1. Настройка сетевой инфраструктуры
В этом модуле необходимо настроить базовую связность, маршрутизацию, VLAN, безопасный доступ и DNS.

Шаг 1. Расчет IP-адресации (п. 1 задания)
Используя условия задания (RFC1918) и ограничения по количеству хостов, рассчитаем сети.

VLAN 100 (HQ-SRV): не более 32 адресов → маска /27 (255.255.255.224)

Сеть: 192.168.10.0/27

Шлюз (HQ-RTR): 192.168.10.1

Сервер (HQ-SRV): 192.168.10.2

VLAN 200 (HQ-CLI): не менее 16 адресов → маска /27 (хватает на 30)

Сеть: 192.168.10.32/27

Шлюз (HQ-RTR): 192.168.10.33

Клиент (HQ-CLI): 192.168.10.34 (по DHCP)

VLAN 999 (Management): не более 8 адресов → маска /29 (255.255.255.248)

Сеть: 192.168.10.64/29

Адрес для управления HQ-RTR: 192.168.10.65

Сеть BR-SRV: не более 16 адресов → маска /28 (255.255.255.240)

Сеть: 192.168.20.0/28

Шлюз (BR-RTR): 192.168.20.1

Сервер (BR-SRV): 192.168.20.2

Таблица 2 (Заполненная):

Имя устройства	IP-адрес	Шлюз по умолчанию
HQ-RTR	192.168.10.1 (VLAN100) / 192.168.10.65 (VLAN999)	172.16.1.2 (в сторону ISP)
BR-RTR	192.168.20.1	172.16.2.2 (в сторону ISP)
HQ-SRV	192.168.10.2/27	192.168.10.1
HQ-CLI	DHCP (из пула 192.168.10.34-63)	192.168.10.33
BR-SRV	192.168.20.2/28	192.168.20.1
Шаг 2. Базовая настройка и доступ в интернет (п. 2)
Конфигурация на маршрутизаторе ISP (логика команд для Linux/EcoRouter):

bash
# Настройка интерфейса к провайдеру (получение адреса по DHCP)
ip addr add dhcp dev eth0
ip link set eth0 up

# Настройка интерфейсов к офисам
ip addr add 172.16.1.1/28 dev eth1
ip link set eth1 up
ip addr add 172.16.2.1/28 dev eth2
ip link set eth2 up

# Включение IP-форвардинга (маршрутизации)
sysctl -w net.ipv4.ip_forward=1

# Настройка NAT (динамическая трансляция портов) для сетей офисов
iptables -t nat -A POSTROUTING -s 192.168.10.0/24 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.20.0/24 -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth2 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth2 -o eth0 -j ACCEPT
Шаг 3. Создание локальных учетных записей (п. 3)
На серверах (HQ-SRV и BR-SRV):

bash
# Создание пользователя sshuser с UID 2026
useradd -u 2026 -m sshuser
echo "sshuser:P@ssw0rd" | chpasswd

# Настройка sudo без пароля
echo "sshuser ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
На маршрутизаторах (HQ-RTR и BR-RTR):

bash
useradd -m net_admin
echo "net_admin:P@ssw0rd" | chpasswd
# Для Linux: дать sudo без пароля, для EcoRouter: дать права admin
Шаг 4. Настройка коммутации и VLAN (п. 4)
В сегменте HQ используется технология «Router-on-a-stick» (один порт на маршрутизатор).

Настройка на HQ-RTR (создание сабинтерфейсов):

bash
# Физический интерфейс eth0 смотрит в сторону коммутатора HQ
ip link add link eth0 name eth0.100 type vlan id 100
ip addr add 192.168.10.1/27 dev eth0.100
ip link set eth0.100 up

ip link add link eth0 name eth0.200 type vlan id 200
ip addr add 192.168.10.33/27 dev eth0.200
ip link set eth0.200 up

ip link add link eth0 name eth0.999 type vlan id 999
ip addr add 192.168.10.65/29 dev eth0.999
ip link set eth0.999 up
В отчёт необходимо внести: схему подключения, список VLAN с именами и ID, конфигурацию сабинтерфейсов.

Шаг 5. Настройка безопасного SSH (п. 5)
На серверах HQ-SRV и BR-SRV (файл /etc/ssh/sshd_config):

bash
# Меняем порт
Port 2026
# Запрещаем вход для root
PermitRootLogin no
# Разрешаем только конкретному пользователю
AllowUsers sshuser
# Ограничиваем попытки ввода пароля
MaxAuthTries 2
# Баннер перед входом
Banner /etc/ssh/ssh-banner
Создаём баннер:

bash
echo "Authorized access only" > /etc/ssh/ssh-banner
systemctl restart sshd
Шаг 6. IP туннель и динамическая маршрутизация (п. 6, 7)
Создание GRE-туннеля между HQ-RTR и BR-RTR:

bash
# На HQ-RTR
ip tunnel add gre1 mode gre remote <IP_BR_RTR_в_сети_172.16.2.2> local 172.16.1.1
ip addr add 10.10.10.1/30 dev gre1
ip link set gre1 up

# На BR-RTR
ip tunnel add gre1 mode gre remote 172.16.1.1 local 172.16.2.2
ip addr add 10.10.10.2/30 dev gre1
ip link set gre1 up
Настройка OSPF (Link-state протокол) через туннель:

bash
# На HQ-RTR
# Запускаем процесс OSPF (например, используя quagga/frr или bird)
# Разрешаем OSPF только на интерфейсе туннеля
# Парольная защита: authentication message-digest
Отчёт должен содержать: тип туннеля, IP адреса туннеля, конфигурацию OSPF (router id, сети, ключ аутентификации).

Шаг 7. Настройка DHCP для HQ-CLI (п. 9)
Настройка DHCP-сервера на HQ-RTR (пример для dhcpd.conf):

bash
subnet 192.168.10.32 netmask 255.255.255.224 {
    range 192.168.10.34 192.168.10.63;
    option routers 192.168.10.33;          # Шлюз
    option domain-name-servers 192.168.10.2; # DNS - HQ-SRV
    option domain-name "au-team.irpo";
    # Исключаем адрес маршрутизатора
    deny unknown-clients;
}
Шаг 8. Настройка DNS (п. 10)
На сервере HQ-SRV (настройка BIND или аналоги):

Создаём зоны:

forward zone au-team.irpo

reverse zone для сетей

Записи согласно Таблице 3:

bind
$ORIGIN au-team.irpo.
$TTL 1D
@       IN SOA  hq-srv admin.au-team.irpo. (1 1D 1H 1W 3H )
        IN NS   hq-srv

; A и PTR записи
hq-rtr  IN A    192.168.10.1
        IN PTR  hq-rtr.au-team.irpo.
br-rtr  IN A    192.168.20.1
hq-srv  IN A    192.168.10.2
        IN PTR  hq-srv.au-team.irpo.
hq-cli  IN A    192.168.10.34
        IN PTR  hq-cli.au-team.irpo.
br-srv  IN A    192.168.20.2
docker  IN A    172.16.1.1   ; Интерфейс ISP к HQ
web     IN A    172.16.2.1   ; Интерфейс ISP к BR
Модуль 2. Организация сетевого администрирования
В этом модуле работаем с уже предварительно настроенным стендом. Задачи: домен, файловое хранилище, веб-сервера, Ansible, Docker.

Шаг 1. Контроллер домена Samba DC (п. 1)
На сервере BR-SRV (действия):

bash
# Установка Samba DC
apt-get install samba dc
# Подготовка Realm (домена)
samba-tool domain provision --realm=AU-TEAM.IRPO --domain=AU-TEAM --adminpass='P@ssw0rd' --server-role=dc
# Настройка Kerberos и запуск
Создание пользователей и ввод в домен:

bash
# Создание 5 пользователей hquser1..5
for i in {1..5}; do
    samba-tool user create hquser$i P@ssw0rd
done
# Создание группы
samba-tool group add hq
# Добавление пользователей в группу
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5

# Ввод машины HQ-CLI в домен (выполняется на самой HQ-CLI)
realm join --user=administrator au-team.irpo
Настройка sudo: через файл /etc/sudoers.d/hq_group:
%hq ALL=(ALL) /bin/cat, /bin/grep, /usr/bin/id

Шаг 2. Файловое хранилище RAID 0 (п. 2)
На сервере HQ-SRV (диски /dev/sdb, /dev/sdc):

bash
# Создание RAID 0
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
# Создание файловой системы
mkfs.ext4 /dev/md0
# Автомонтирование в /etc/fstab
echo "/dev/md0 /raid ext4 defaults 0 0" >> /etc/fstab
mount -a
Шаг 3. NFS-сервер и автоматическое монтирование (п. 3)
На HQ-SRV (сервер):

bash
mkdir -p /raid/nfs
# Настройка экспорта в /etc/exports
echo "/raid/nfs 192.168.10.32/27(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -a
systemctl start nfs-server
На HQ-CLI (клиент):

bash
# Установка autofs
# Настройка /etc/auto.master: /mnt /etc/auto.nfs
# Создание /etc/auto.nfs: nfs -fstype=nfs4 192.168.10.2:/raid/nfs
systemctl restart autofs
# Доступ по пути /mnt/nfs
Шаг 4. Настройка Ansible (п. 5)
На сервере BR-SRV:

bash
# Создание инвентаря /etc/ansible/hosts
[headquarter]
HQ-SRV ansible_host=192.168.10.2
HQ-CLI ansible_host=192.168.10.34
HQ-RTR ansible_host=192.168.10.1

[branch]
BR-RTR ansible_host=192.168.20.1

[all:vars]
ansible_user=sshuser
ansible_ssh_private_key_file=/home/sshuser/.ssh/id_rsa

# Проверка связи
ansible all -m ping
Шаг 5. Развертывание веб-приложений (п. 6, 7)
Docker на BR-SRV:

bash
# Загрузка образов из Additional.iso
docker load < site_latest.tar
docker load < mariadb_latest.tar
# Создание docker-compose.yml
docker-compose.yml:

yaml
version: '3'
services:
  db:
    image: mariadb_latest
    environment:
      MYSQL_ROOT_PASSWORD: P@ssw0rd
      MYSQL_DATABASE: testdb
      MYSQL_USER: testc
      MYSQL_PASSWORD: P@ssw0rd
  testapp:
    image: site_latest
    ports:
      - "8080:8080"
    depends_on:
      - db
Apache/PHP на HQ-SRV:

bash
# Копируем файлы из web/ в /var/www/html/
cp index.php images/ /var/www/html/
# Импорт БД
mysql -u root < dump.sql
mysql -e "CREATE USER 'webc'@'localhost' IDENTIFIED BY 'P@ssw0rd';"
mysql -e "GRANT ALL PRIVILEGES ON webdb.* TO 'webc'@'localhost';"
# В index.php меняем пароль на P@ssw0rd и БД на webdb
systemctl start httpd
Шаг 6. Статическая трансляция портов (п. 8)
На маршрутизаторах (пример для iptables):

bash
# На HQ-RTR: проброс 2026 порта на HQ-SRV
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 2026 -j DNAT --to-destination 192.168.10.2:2026
iptables -A FORWARD -p tcp -d 192.168.10.2 --dport 2026 -j ACCEPT

# Проброс 8080 на веб HQ-SRV
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 -j DNAT --to-destination 192.168.10.2:80
iptables -A FORWARD -p tcp -d 192.168.10.2 --dport 80 -j ACCEPT
Шаг 7. Обратный прокси на ISP (п. 9)
Настройка nginx на маршрутизаторе ISP:

nginx
server {
    listen 80;
    server_name web.au-team.irpo;
    location / {
        proxy_pass http://172.16.1.1:8080;  # Порт, проброшенный на HQ-RTR
    }
}
server {
    listen 80;
    server_name docker.au-team.irpo;
    location / {
        proxy_pass http://172.16.2.1:8080;  # Порт, проброшенный на BR-RTR
    }
}
Шаг 8. Установка Яндекс Браузера (п. 11)
На HQ-CLI:

bash
# Скачивание .rpm пакета с официального сайта (или копирование из ISO)
# Установка
sudo rpm -i yandex-browser-stable-current.rpm
# ИЛИ через apt, если добавлен репозиторий
