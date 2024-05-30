# shiny-octo-eureka

**Изменение имени компьютера**: # hostnamectl set-hostname vm.company.prof

Посмотреть IP: ip addr show или ip a

Шлюз по умолчанию: ip route show или ip r

Поменять интерфейсы: nmcli or nmtui

Управление пользователями
Добавление нового пользователя # useradd -p пароль имя_пользователя

**Добавление пользователя в sudoers**
Заходим под рутом: su -

sudo vi /etc/sudoers

sshuser ALL=(ALL:ALL) NOPASSWD:ALL или echo "sshuser ALL=(ALL:ALL) NOPASSWD: ALL" >> /etc/sudoers

usermod -aG wheel sshuser

**SSH ПОДКЛЮЧЕНИЕ И НАСТРОЙКА**

На серверах, к которым подключаемся:

mkdir /home/sshuser

chown sshuser:sshuser /home/sshuser

cd /home/sshuser

mkdir -p .ssh/

chmod 700 .ssh

touch .ssh/authorized_keys

chmod 600 .ssh/authorized_keys

chown sshuser:sshuser .ssh/authorized_keys

На машине с которого подключаемся к серверам:

ssh-keygen

ssh-copy-id root@5.159.102.80

На сервере /etc/ssh/sshd_config:
AllowUsers sshuser
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
AuthorizedKeysFile .ssh/authorized_keys

systemctl restart sshd

Подключение:ssh srv-hq

**НАСТРОЙКА КОММУТАЦИИ**
 SW-HQ
**Временное назначение ip-адреса (смотрящего в сторону rtr-hq):**

ip link add link ens18 name ens18.300 type vlan id 300

ip link set dev ens18.300 up

ip addr add 10.0.10.66/27 dev ens18.300

ip route add 0.0.0.0/0 via 10.0.10.65

echo nameserver 8.8.8.8 > /etc/resolv.conf

Обновление пакетов и установка openvswitch:

apt-get update && apt-get install -y openvswitch
Автозагрузка:

systemctl enable --now openvswitch
ens18 - rtr-hq
ens19 - cicd-hq - vlan200 ens20 - srv-hq - vlan100 ens21 - cli-hq - vlan200

Создаем каталоги для ens19,ens20,ens21:

mkdir /etc/net/ifaces/ens{19,20,21}
Для моста:

mkdir /etc/net/ifaces/ovs0
Management интерфейс:

mkdir /etc/net/ifaces/mgmt
Не удалять настройки openvswitch:

sed -i "s/OVS_REMOVE=yes/OVS_REMOVE=no/g" /etc/net/ifaces/default/options
Мост /etc/net/ifaces/ovs0/options:

mgmt /etc/net/ifaces/mgmt/options:


Поднимаем интерфейсы:

cp /etc/net/ifaces/ens18/options /etc/net/ifaces/ens19/options

cp /etc/net/ifaces/ens18/options /etc/net/ifaces/ens20/options

cp /etc/net/ifaces/ens18/options /etc/net/ifaces/ens21/options

Назначаем Ip, default gateway на mgmt:

echo 10.0.10.66/27 > /etc/net/ifaces/mgmt/ipv4address
echo default via 10.0.10.65 > /etc/net/ifaces/mgmt/ipv4route

Перезапуск network:

systemctl restart network
Проверка:

ip -c --br a
ovs-vsctl show
ens18 - rtr-hq делаем trunk и пропускаем VLANs:

ovs-vsctl set port ens18 trunk=100,200,300
ens19 - tag=200

ovs-vsctl set port ens19 tag=200
ens20 - tag=100:

ovs-vsctl set port ens20 tag=100
ens21 - tag=200

ovs-vsctl set port ens21 tag=200
Включаем инкапсулирование пакетов по 802.1q:
modprobe 8021q

**NAT
**
Rtr-hq:

config

security zone public

exit

int te1/0/2

security-zone public

exit

object-group network COMPANY

ip address-range 10.0.10.1-10.0.10.254

exit

object-group network WAN

ip address-range 11.11.11.11

exit

nat source

pool WAN

ip address-range 11.11.11.11

exit

ruleset SNAT

to zone public

rule 1

match source-address COMPANY

action source-nat pool WAN

enable

exit

exit

commit

confirm

rtr-br:

config

security zone public

exit

int te1/0/2

security-zone public

exit

object-group network COMPANY

ip address-range 10.0.20.1-10.0.20.254

exit

object-group network WAN

ip address-range 22.22.22.22

exit

nat source

pool WAN

ip address-range 22.22.22.22

exit

ruleset SNAT

to zone public

rule 1

match source-address COMPANY

action source-nat pool WAN

enable

exit

exit

commit

confirm

RTR-HQ
DNS-сервер - srv-hq

configure terminal

ip dhcp-server pool COMPANY-HQ

network 10.0.10.32/27

default-lease-time 3:00:00

address-range 10.0.10.33-10.0.10.62

excluded-address-range 10.0.10.33   

default-router 10.0.10.33 

dns-server 10.0.10.2 (srv-HQ)

domain-name company.prof

exit

Включение DHCP-сервера:
ip dhcp-server

**ANSIBLE**
apt-get install -y ansible sshpass
Создаем файл: nano /etc/ansible/inventory.yml
Networking:
  hosts:
    rtr-hq:
    
      ansible_host: 10.0.10.1
      
      ansible_user: admin
      
      ansible_password: P@ssw0rd
      
      ansible_port: 22
      
    rtr-br:
    
      ansible_host: 10.0.20.1
      
      ansible_user: admin
      
      ansible_password: P@ssw0rd
      
      ansible_port: 22
      
Servers:
  hosts:
    srv-hq:
      ansible_host: 10.0.10.2
      
      ansible_ssh_private_key_file: ~/.ssh/srv_ssh_key
      
      ansible_user: sshuser
      
      ansible_port: 2023
    srv-br:
    
      ansible_host: 10.0.20.2
      
      ansible_ssh_private_key_file: ~/.ssh/srv_ssh_key
      
      ansible_user: sshuser
      
      ansible_port: 22
      
Clients:
  hosts:
  
    cli-hq:
    
      ansible_host: 10.0.10.34
      
      ansible_user: root
      
      ansible_password: P@ssw0rd
      
      ansible_port: 22
      
    cli-br:
      ansible_host: 10.0.20.34
      
      ansible_user: root
      
      ansible_password: P@ssw0rd
      
      ansible_port: 22

**FREEIPA**
Демон энтропии haveged:
apt-get update && apt-get install -y haveged

Автозагрузка:
systemctl enable --now haveged

FreeIPA:
apt-get install -y freeipa-server

Интерактивная установка:
ipa-server-install

Получаем билет kerberos:
kinit admin

Цикл для создания 30 пользователей User1-User30, пароль P@ssw0rd, срок действия пароля до 2025:

for i in {1..30};do 
	echo "P@ssw0rd" | ipa user-add user$i --first=User --last=$i --password;
	ipa user-mod user$i --setattr=krbPasswordExpiration=20251225011529Z;
done
Создадим группы group1, group2, group3:

for i in {1..3}; do
	ipa group-add group$i;
done
Добавим юзеров в группы

group1:

for i in {1..10}; do
	ipa group-add-member group1 --users=user$i;
done
group2:

for i in {11..20}; do
	ipa group-add-member group2 --users=user$i;
done
group3:

for i in {21..30}; do
	ipa group-add-member group3 --users=user$i;
done
Устанавливаем сертификат FreeIPA:

cp /etc/ipa/ca.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust extract

CLI-HQ
Установка:

apt-get update && apt-get install -y freeipa-client zip
Инициализация клиента:

ipa-client-install --mkhomedir
