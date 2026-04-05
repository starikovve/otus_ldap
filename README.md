# otus_ldap
LDAP. Централизованная авторизация и аутентификация 


# Тема домашнего задания:
Настройка LDAP-сервера и подключение LDAP-клиентов.
Цель работы:
Научиться устанавливать и настраивать LDAP-сервер на базе FreeIPA, а также автоматизировать подключение LDAP-клиентов с помощью Ansible.
Задачи работы:  
1 Установить и настроить FreeIPA-сервер.  
2 Проверить работоспособность LDAP-сервиса.  
3 Подготовить LDAP-клиента для подключения к серверу.  
4 Написать Ansible playbook для автоматической конфигурации клиента.  
5 Проверить успешность подключения клиента к FreeIPA.
# Краткое описание работы:
В ходе выполнения домашнего задания необходимо развернуть LDAP-сервер на основе FreeIPA и настроть его для работы с клиентскими машинами. Затем требуется создать Ansible playbook, который позволит автоматически выполнять настройку LDAP-клиента и подключение его к серверу. После этого необходимо убедиться, что клиент успешно зарегистрирован и может взаимодействовать с сервером.
Ожидаемый результат:
После выполнения работы должен быть установлен и настроен FreeIPA-сервер, а также создан Ansible playbook, обеспечивающий автоматическую настройку LDAP-клиента.

Создадим Vagrantfile, в котором будут указаны параметры наших ВМ:

```

Vagrant.configure("2") do |config|
    # Указываем ОС, версию, количество ядер и ОЗУ
    config.vm.box = "generic/centos8"
    config.vm.box_version = "4.3.2"
 
    config.vm.provider :virtualbox do |v|
      v.memory = 2048
      v.cpus = 1
    end
  
    # Указываем имена хостов и их IP-адреса
    boxes = [
      { :name => "ipa.otus.lan",
        :ip => "192.168.57.10",
      },
      { :name => "client1.otus.lan",
        :ip => "192.168.57.11",
      },
      { :name => "client2.otus.lan",
        :ip => "192.168.57.12",
      }
    ]
    # Цикл запуска виртуальных машин
    boxes.each do |opts|
      config.vm.define opts[:name] do |config|
        config.vm.hostname = opts[:name]
        config.vm.network "private_network", ip: opts[:ip]
      end
    end
  end
```

Будут созданы 3 виртуальных машины с ОС CentOS 8 Stream. Каждая ВМ будет иметь по 2ГБ ОЗУ и по одному ядру CPU. 

# Установка FreeIPA сервера

Подключимся к нему по SSH с помощью команды: 

```
vagrant ssh ipa.otus.lan
```
 перейдём в root-пользователя: sudo -i 

<img width="491" height="114" alt="image" src="https://github.com/user-attachments/assets/aa2078b4-106d-4f38-b53f-8ebb290524fc" />



Начнем настройку FreeIPA-сервера: 

- Установим часовой пояс: timedatectl set-timezone Europe/Moscow
- Установим утилиту chrony: yum install -y chrony
- Запустим chrony и добавим его в автозагрузку: systemctl enable chronyd —now
- Если требуется, поменяем имя нашего сервера: hostnamectl set-hostname <имя сервера>
- В нашей лабораторной работе данного действия не требуется, так как уже указаны корректные имена в Vagrantfile
- Выключим Firewall: systemctl stop firewalld
- Отключаем автозапуск Firewalld: systemctl disable firewalld
- Остановим Selinux: setenforce 0
- Поменяем в файле /etc/selinux/config, параметр Selinux на disabled

<img width="890" height="151" alt="image" src="https://github.com/user-attachments/assets/ebb6eb5f-d0c2-4040-a307-1d160faed86a" />


```
vi /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of these three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

<img width="876" height="566" alt="image" src="https://github.com/user-attachments/assets/b46a60e6-95fb-4b68-bf8d-80a8ad7292df" />


Для дальнейшей настройки FreeIPA нам потребуется, чтобы DNS-сервер хранил запись о нашем LDAP-сервере. В рамках данной лабораторной работы мы не будем настраивать отдельный DNS-сервер и просто добавим запись в файл /etc/hosts

```
vi /etc/hosts

127.0.0.1   localhost localhost.localdomain 
127.0.1.1 ipa.otus.lan ipa
192.168.57.10 ipa.otus.lan ipa
```
<img width="932" height="561" alt="image" src="https://github.com/user-attachments/assets/c4b08e87-3172-4d29-8b37-6bbcc307bbae" />


- Установим модуль DL1: yum install -y @idm:DL1
- Установим FreeIPA-сервер: yum install -y ipa-server

Полсе запуска выдал ошибку и требует ipv6 на lo

<img width="1318" height="237" alt="image" src="https://github.com/user-attachments/assets/153de77e-4712-4c1f-997e-ae49d2d48062" />

```
ip addr show lo
```
<img width="1003" height="115" alt="image" src="https://github.com/user-attachments/assets/16339a3e-dd57-4b9a-bd1d-a1a6dbc330c9" />

Включаем ipv6 на lo
```
sysctl -w net.ipv6.conf.lo.disable_ipv6=0
```
<img width="1074" height="196" alt="image" src="https://github.com/user-attachments/assets/ba70fa91-f358-4a79-a0de-745ef4587aab" />


- Запустим скрипт установки: ipa-server-install
Далее, нам потребуется указать параметры нашего LDAP-сервера, после ввода каждого параметра нажимаем Enter, если нас устраивает параметр, указанный в квадратных скобках, то можно сразу нажимать Enter:
```
Do you want to configure integrated DNS (BIND)? [no]: no
Server host name [ipa.otus.lan]: <Нажимаем Enter>
Please confirm the domain name [otus.lan]: <Нажимем Enter>
Please provide a realm name [OTUS.LAN]: <Нажимаем Enter>
Directory Manager password: <Указываем пароль минимум 8 символов>
Password (confirm): <Дублируем указанный пароль>
IPA admin password: <Указываем пароль минимум 8 символов>
Password (confirm): <Дублируем указанный пароль>
NetBIOS domain name [OTUS]: <Нажимаем Enter>
Do you want to configure chrony with NTP server or pool address? [no]: no
The IPA Master Server will be configured with:
Hostname:       ipa.otus.lan
IP address(es): 192.168.57.10
Domain name:    otus.lan
Realm name:     OTUS.LAN

The CA will be configured with:
Subject DN:   CN=Certificate Authority,O=OTUS.LAN
Subject base: O=OTUS.LAN
Chaining:     self-signed
Проверяем параметры, если всё устраивает, то нажимаем yes
Continue to configure the system with these values? [no]: yes

```
Далее начнется процесс установки. Процесс установки занимает примерно 10-15 минут (иногда время может быть другим). Если мастер успешно выполнит настройку FreeIPA то в конце мы получим сообщение: 
The ipa-server-install command was successful

<img width="1091" height="511" alt="image" src="https://github.com/user-attachments/assets/7453c2fd-4dad-4b17-9d9b-399a4c0eac67" />


При вводе параметров установки мы вводили 2 пароля:

- Directory Manager password — это пароль администратора сервера каталогов, У этого пользователя есть полный доступ к каталогу.
- IPA admin password — пароль от пользователя FreeIPA admin


После успешной установки FreeIPA, проверим, что сервер Kerberos может выдать нам билет: 

```
[root@ipa ~]# kinit admin
Password for admin@OTUS.LAN:  #Указываем Directory Manager password
[root@ipa ~]# klist           #Запросим список билетов Kerberos
Ticket cache: KCM:0
Default principal: admin@OTUS.LAN
```
```
kinit admin
klist
```
<img width="1037" height="211" alt="image" src="https://github.com/user-attachments/assets/07485313-baf0-4a8c-8cdd-5b8f5537ebc1" />

Для удаление полученного билета воспользуемся командой: kdestroy

Мы можем зайти в Web-интерфейс нашего FreeIPA-сервера, для этого на нашей хостой машине нужно прописать следующую строку в файле Hosts:
192.168.57.10 ipa.otus.lan

<img width="759" height="146" alt="image" src="https://github.com/user-attachments/assets/80caee73-462b-4ef2-8665-03480cd8228c" />


Откроется окно управления FreeIPA-сервером. В имени пользователя укажем admin, в пароле укажем наш IPA admin password и нажмём войти. 


<img width="2558" height="514" alt="image" src="https://github.com/user-attachments/assets/ac265b89-f3ae-4e7d-8e8c-fd93b9008099" />


На этом установка и настройка FreeIPA-сервера завершена.

# Ansible playbook для конфигурации клиента


Настройка клиента похожа на настройку сервера. На хосте также нужно:
- Настроить синхронизацию времени и часовой пояс
- Настроить (или отключить) firewall
- Настроить (или отключить) SElinux
  
В файле hosts должна быть указана запись с FreeIPA-сервером и хостом

Хостов, которые требуется добавить к серверу может быть много, для упрощения нашей работы выполним настройки с помощью Ansible:

В каталоге с нашей лабораторной работой создадим каталог Ansible: mkdir ansible
В каталоге ansible создадим файл hosts со следующими параметрами:

```
[clients]
client1.otus.lan ansible_host=192.168.57.11 ansible_user=vagrant ansible_ssh_private_key_file=/home/vlad/.ssh/ldap/client11/private_key
client2.otus.lan ansible_host=192.168.57.12 ansible_user=vagrant ansible_ssh_private_key_file=/home/vlad/.ssh/ldap/client12/private_key
```
Файл содержит группу clients в которой прописаны 2 хоста: 
client1.otus.lan
client2.otus.lan
Также указаны и ip-адреса, имя пользователя от которого будет логин и ssh-ключ.

Далее создадим файл provision.yml в котором непосредственно будет выполняться настройка клиентов: 

```
- name: Base set up
  hosts: clients
  #Выполнять действия от root-пользователя
  become: yes
  tasks:
  #Установка текстового редактора Vim и chrony
  - name: install softs on CentOS
    yum:
      name:
        - vim
        - chrony
      state: present
      update_cache: true
  
  #Отключение firewalld и удаление его из автозагрузки
  - name: disable firewalld
    service:
      name: firewalld
      state: stopped
      enabled: false
  
  #Отключение SElinux из автозагрузки
  #Будет применено после перезагрузки
  - name: disable SElinux
    selinux:
      state: disabled
  
  #Отключение SElinux до перезагрузки
  - name: disable SElinux now
    shell: setenforce 0

  #Установка временной зоны Европа/Москва    
  - name: Set up timezone
    timezone:
      name: "Europe/Moscow"
  
  #Запуск службы Chrony, добавление её в автозагрузку
  - name: enable chrony
    service:
      name: chronyd
      state: restarted
      enabled: true
  
  #Копирование файла /etc/hosts c правами root:root 0644
  - name: change /etc/hosts
    template:
      src: hosts.j2
      dest: /etc/hosts
      owner: root
      group: root
      mode: 0644
  
  #Установка клиента Freeipa
  - name: install module ipa-client
    yum:
      name:
        - freeipa-client
      state: present
      update_cache: true
  
  #Запуск скрипта добавления
 хоста к серверу
  - name: add host to ipa-server
    shell: echo -e "yes\nyes" | ipa-client-install --mkhomedir --domain=OTUS.LAN --server=ipa.otus.lan --no-ntp -p admin -w 1Q2w3e4r5
```

<img width="1502" height="188" alt="image" src="https://github.com/user-attachments/assets/745ef1f8-3d73-402b-8071-c650f3e24b48" />

Почти все модули нам уже знакомы, давайте подробнее остановимся на последней команде echo -e "yes\nyes" | ipa-client-install --mkhomedir --domain=OTUS.LAN --server=ipa.otus.lan --no-ntp -p admin -w 1Q2w3e4r5


При добавлении хоста к домену мы можем просто ввести команду ipa-client-install и следовать мастеру подключения к FreeIPA-серверу (как было в первом пункте).

```
Однако команда позволяет нам сразу задать требуемые нам параметры:
--domain — имя домена
--server — имя FreeIPA-сервера
--no-ntp — не настраивать дополнительно ntp (мы уже настроили chrony)
-p — имя админа домена
-w — пароль администратора домена (IPA password)
--mkhomedir — создать директории пользователей при их первом логине
```

Если мы сразу укажем все параметры, то можем добавить эту команду в Ansible и автоматизировать процесс добавления хостов в домен. 

Альтернативным вариантом мы можем найти на GitHub отдельные модули по подключениею хостов к FreeIPA-сервер. 

После подключения хостов к FreeIPA-сервер нужно проверить, что мы можем получить билет от Kerberos сервера: kinit admin
Если подключение выполнено правильно, то мы сможем получить билет, после ввода пароля. 

<img width="805" height="202" alt="image" src="https://github.com/user-attachments/assets/55d3217c-6924-459f-9aeb-38a3baf17e51" />


Давайте проверим работу LDAP, для этого на сервере FreeIPA создадим пользователя и попробуем залогиниться к клиенту:

- Авторизируемся на сервере: kinit admin
- Создадим пользователя otus-user
  ```
  ipa user-add otus-user --first=Otus --last=User --password
  ```
<img width="851" height="470" alt="image" src="https://github.com/user-attachments/assets/2697fc76-21a5-42b7-a518-b35f2ff76ba5" />

На хосте client1 или client2 выполним команду kinit otus-user

<img width="1010" height="117" alt="image" src="https://github.com/user-attachments/assets/98c2f8cb-41a4-498c-b97d-bc2d70e7a9ea" />

<img width="661" height="140" alt="image" src="https://github.com/user-attachments/assets/f20a3d0a-34de-4911-bbc4-45065f91fa2d" />


Система запросит у нас пароль и попросит ввести новый пароль. 

На этом процесс добавления хостов к FreeIPA-серверу завершен.
