# shiny-octo-eureka
linux
Имя компьютера
Имя хоста хранится в файле /etc/hostname. В файле также может храниться доменное имя системы, если таковое имеется.
Просмотреть имя компьютера можно, выполнив команду: hostname
Изменение имени компьютера: # hostnamectl set-hostname vm.company.prof

Сетевые карты
Список доступных сетевых карт: $ lspci | grep -i 'net'

Посмотреть IP: ip addr show или ip a
Шлюз по умолчанию: ip route show или ip r

Поменять интерфейсы: nmcli

Управление пользователями
Добавление нового пользователя # useradd -p пароль имя_пользователя

**Добавление пользователя в sudoers**
Заходим под рутом: su -
sudo vi /etc/sudoers
sshuser ALL=(ALL:ALL) NOPASSWD:ALL или echo "sshuser ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
usermod -aG wheel sshuser
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
ssh-keygen -t rsa -b 2048 -f srv_ssh_key
mkdir .ssh
mv srv_ssh_key* .ssh/
Конфиг для автоматического подключения .ssh/config:

Host srv-hq
        HostName 10.0.10.2
        User sshuser
        IdentityFile .ssh/srv_ssh_key
        Port 2023
Host srv-br
        HostName 10.0.20.2
        User sshuser
        IdentityFile .ssh/srv_ssh_key
        Port 2023
chmod 600 .ssh/config
Копирование ключа на удаленный сервер:

ssh-copy-id -i .ssh/srv_ssh_key.pub sshuser@10.0.10.2
ssh-copy-id -i .ssh/srv_ssh_key.pub sshuser@10.0.20.2
На сервере /etc/ssh/sshd_config:

AllowUsers sshuser
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
AuthorizedKeysFile .ssh/authorized_keys
Port 2023
systemctl restart sshd
Подключение:

ssh srv-hq

Permission denied (publickey).
Попробуйте войти только с паролем:
Код:
$ ssh vivek@server1.cyberciti.biz -o PubkeyAuthentication=no
Permission denied (publickey).
