# A1 Виртуальные машины и коммутация
## Настройка имен хостов

### Windows (Server & Desktop)

**Вариант 1.** Графические утилиты

**Вариант 2.** PowerShell (от администратора):

    > Rename-Computer -NewName <name> | shutdown /r /t 0

### Linux

    # hostnamectl set-hostname <name> && reboot

### CSR

    (config)# hostname <name>

## Настройка адресов

### Windows (Server & Desktop)

**Вариант 1.** Графические утилиты

**Вариант 2.** PowerShell (от администратора):
    
    > Get-NetAdapter    # Ищем нужный адаптер и его индекс (столбец ifindex)
    > New-NetIPAddress -IPAddress <ip> -DefaultGateway <gw> -PrefixLength <pl> -InterfaceIndex <ifindex>

Например:

    > New-NetIPAddress -IPAddress 192.168.100.200 -DefaultGateway 192.168.100.1 -PrefixLength 24 -InterfaceIndex 4

### Linux (Debian 11)

**Вариант 1 (рекомендуемый)**

Редактируем /etc/network/interfaces:

    auto <ifname>
    iface <ifname> inet static
      address <ip>
      netmask <mask>
      gateway <gw>
      dns-nameservers <dns-ip>
    
Перезапускаем сервис networking

    # systemctl restart networking

\<ifname\> можно посмотреть в выводе команды ip a

**Вариант 2 (действует до перезагрузки)**

    # ip address add <ip/prefix> dev <ifname>
    # ip route add 0.0.0.0/0 via <gw>

**На ISP (обязательно)**
    
    # echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
    # sysctl -p

### CSR
    (config)# interface <ifname>
    (config-if)# ip address <ip> <mask>
    (config-if)# no shutdown

**На RTR-L и RTR-R также надо настройть шлюз последней надежды**

    (config)# ip route 0.0.0.0 0.0.0.0 <gw_ip>

# B1 Сетевая связность
## Настройка NAT (RTR-L, RTR-R)

### RTR-L
   
    (config)# access-list 1 permit 192.168.100.0 0.0.0.255
    (config)# ip nat inside source list 1 interface g1 overload
    (config)# interface g1
    (config-if)# ip nat outside
    (config-if)# interface g2
    (config-if)# ip nat inside

### RTR-R
   
    (config)# access-list 1 permit 172.16.100.0 0.0.0.255
    (config)# ip nat inside source list 1 interface g1 overload
    (config)# interface g1
    (config-if)# ip nat outside
    (config-if)# interface g2
    (config-if)# ip nat inside

## Настройка защищенного туннеля

##### Настройка IPsec

**RTR-L**

**Phase 1**

    (config)# crypto isakmp policy 1
    (config-isakmp)# ecryption aes 128
    (config-isakmp)# hash sha256
    (config-isakmp)# authentication pre-share
    (config-isakmp)# group 5
    (config)# crypto isakmp key 12345 address 5.5.5.100

**Phase 2**

    (config)# crypto ipsec transform-set TS esp-aes esp-sha256-hmac
    (cfg-crypto-trans)# mode tunnel
    (config)# crypto ipsec profile VPN
    (ipsec-profile)# set transform-set TS

**RTR-R**

**Phase 1**

    (config)# crypto isakmp policy 1
    (config-isakmp)# ecryption aes 128
    (config-isakmp)# hash sha256
    (config-isakmp)# authentication pre-share
    (config-isakmp)# group 5
    (config)# crypto isakmp key 12345 address 4.4.4.100

**Phase 2**

Абсолютно идентично RTR-L -> Phase 2

##### Настройка GRE

**RTR-L**

    (config)# interface Tunnel1
    (config-if)# ip address 10.10.10.1 255.255.255.252
    (config-if)# tunnel source g1
    (config-if)# tunnel destination 5.5.5.100
    (config-if)# tunnel protection ipsec profile VPN

**RTR-R**

    (config)# interface Tunnel1
    (config-if)# ip address 10.10.10.2 255.255.255.252
    (config-if)# tunnel source g1
    (config-if)# tunnel destination 4.4.4.100
    (config-if)# tunnel protection ipsec profile VPN


## Настройка правильной маршрутизации

Можно статические машруты, но проще динамику (OSPF или EIGRP на выбор)

##### RTR-L

    (config)# router ospf 1
    (config-router)# router-id 4.4.4.100
    (config-router)# network 192.168.100.0 0.0.0.255 area 1
    (config-router)# network 10.10.10.0 0.0.0.3 area 0
    (config-router)# passive-interface g2

##### RTR-R

    (config)# router ospf 1
    (config-router)# router-id 5.5.5.100
    (config-router)# network 172.16.100.0 0.0.0.255 area 2
    (config-router)# network 10.10.10.0 0.0.0.3 area 0
    (config-router)# passive-interface g2

## Настройка ACL на RTR-L

    (config)# ip access-list extended SERVICES
    (config-ext-nacl)# permit gre any any
    (config-ext-nacl)# permit esp any any
    (config-ext-nacl)# permit icmp any any
    (config-ext-nacl)# permit ospf any any
    (config-ext-nacl)# permit tcp any host 4.4.4.100 eq 53 www 443 2222
    (config-ext-nacl)# permit udp any host 4.4.4.100 eq 53
    (config-ext-nacl)# permit tcp any eq 53 host 4.4.4.100
    (config-ext-nacl)# permit udp any eq 53 host 4.4.4.100
    (config-ext-nacl)# permit udp any eq 500 any eq 500
    (config)# interface g1
    (config-if)# ip access-group SERVICES in

## Настройка ACL на RTR-R

    (config)# ip access-list extended SERVICES
    (config-ext-nacl)# permit gre any any
    (config-ext-nacl)# permit esp any any
    (config-ext-nacl)# permit icmp any any
    (config-ext-nacl)# permit ospf any any
    (config-ext-nacl)# permit tcp any host 5.5.5.100 eq www 443 2244
    (config-ext-nacl)# permit udp any eq 500 any eq 500
    (config)# interface g1
    (config-if)# ip access-group SERVICES in

## Настройка ssh для LEFT

**На WEB-L должен быть установлен OpenSSH (в т.ч. если это WindowsServer)**

    (config)# ip nat inside source static tcp 192.168.100.100 22 4.4.4.100 2222 extendable

## Настройка ssh для RIGHT

**На WEB-R должен быть установлен OpenSSH (в т.ч. если это WindowsServer)**

    (config)# ip nat inside source static tcp 172.16.100.100 22 5.5.5.100 2244 extendable


# C1 Инфраструктурные службы

## Настройка DNS первого слоя

### Настройка на ISP (Debian 11)

#### Предварительные требования

- Должен быть установлен пакет bind9

#### Настройка

0. Если ISP подключен к интернету, то правим /etc/bind/named.conf.options

        allow-recursion { any; };
    
На SRV, если он на Linux также надо добавить (или изменить)
        
        dsssec-validation no;

1. Содержимое /etc/bind/named.conf.local

        zone "demo.wsr" {
            type master;
            file "/etc/bind/db.demo.wsr";
            allow-query { any; };
            allow-transfer { 4.4.4.100; };
        };

2. Для упрощения копируем /etc/bind/db.empty в /etc/bind/db.demo.wsr
3. Содержимое /etc/bind/db.demo.wsr

        $TTL    604800  ; не трогаем
        @   IN  SOA demo.wsr.   root.demo.wsr.  (
                    2           ; не трогаем
                    604800      ; не трогаем
                    86400       ; не трогаем
                    2419200     ; не трогаем
                    604800      ; не трогаем
        )
        ;
        @   IN  NS  isp
        int IN  NS  srv.int
        
        srv.int IN  A   4.4.4.100
        isp IN  A   3.3.3.1
        www IN  A   4.4.4.100
        www IN  A   5.5.5.100
        internet    IN  CNAME   isp

4. Перезапускаем сервис

        systemctl restart named
    

## Настройка DNS второго слоя

### Настройка на SRV (Windows Server)

##### Установка роли DNS

**Вариант 1 (графический)**

Диспетчер серверов -> Управление -> Добавить роли и компоненты

В мастере на этапе "Роли сервера" выбрать "DNS-сервер"

**Вариант 2 (posh, для умных)**

    > Install-WindowsFeature -Name DNS -IncludeManagementTools

##### НАстройка DNS

**Вариант 1 (графический)**

Всё через Средсва -> DNS

**Вариант 2 (posh, для умных)**

Создание зоны прямого просмотра

    > Add-DnsServerPrimaryZone -Name "int.demo.wsr" -ZoneFile "int.demo.wsr.dns"

Создание зоны обратного просмотра для 192.168.100.0/24

    > Add-DnsServerPrimaryZone -NetworkId "192.168.100.0/24" -ZoneFile "168.192.dns"

Создание зоны обратного просмотра для 172.16.100.0/24

    > Add-DnsServerPrimaryZone -NetworkId "172.16.100.0/24" -ZoneFile "100.16.172.dns"

Добавляем сервер пересылки

    > Add-DnsServerForwarder -IPAddress 3.3.3.1

Разрешаем рекурсивные запросы

    > Set-DnsServerRecursion -Enable $true


##### Добавление записей

    > Add-DnsServerResourceRecordA -IPv4Address 192.168.100.200 -Name srv -ZoneName int.demo.wsr -CreatePtr
    > Add-DnsServerResourceRecordA -IPv4Address 192.168.100.100 -Name web-l -ZoneName int.demo.wsr -CreatePtr
    > Add-DnsServerResourceRecordA -IPv4Address 192.168.100.254 -Name rtr-l -ZoneName int.demo.wsr -CreatePtr
    > Add-DnsServerResourceRecordA -IPv4Address 172.16.100.100 -Name web-r -ZoneName int.demo.wsr -CreatePtr
    > Add-DnsServerResourceRecordA -IPv4Address 172.16.100.254 -Name rtr-r -ZoneName int.demo.wsr -CreatePtr
    > Add-DnsServerResourceRecordCname -Name ntp -HostNameAlias srv.int.demo.wsr -ZoneName
    > Add-DnsServerResourceRecordCname -Name dns -HostNameAlias srv.int.demo.wsr -ZoneName

##### Донастройки

1. На всех внутренних машинах указать ip SRV в качестве dns

Если это Linux (быстрый способ, если стоит cloud-init, то работает только до перезагрузки. По хорошему - надо через файлы конфигурации интерфейсов):

    # echo "nameserver 192.168.100.200" > /etc/resolv.conf

Если Windows Server

    > Set-DNSClientServerAddress –InterfaceIndex 4 –ServerAddresses 192.168.100.200

2. Прокидываем порт на RTR-L

        (config)# ip nat inside source static tcp 192.168.100.200 53 4.4.4.100 53 extendable
        (config)# ip nat inside source static udp 192.168.100.200 53 4.4.4.100 53 extendable

## Настройка синхронизации времени (первый уровень)

Настройка происходит на ISP

**Если ISP - Linux - должен быть установлен пакет chrony**

Редактируем /etc/chrony/chrony.conf

- Удаляем все записи `peer`, `pool`, `server`
- Добавляем записи
  - server 127.0.0.1 iburst trust
  - local stratum 4
  - allow 4.4.4.100
  - allow 3.3.3.10
- Перезапускаем сервис (systemctl restart chronyd)
- Проверяем, что стратум верный (chronyc tracking)

Настройка CLI для синхронизации с ISP

**Вариант 1 (графика)**

Панель управления -> Часы и регион -> Дата и время -> Время по интернету -> Изменить параметры. Устанавливаем IP ISP -> Обновить сейчас -> OK

**Вариант 2 (posh, для умных)**

    > Start-Service w32time
    > w32tm /config /syncfromflags:manual /manualpeerlist:"3.3.3.1" //перед кавычками с IP НЕ ДОЛЖНО БЫТЬ ПРОБЕЛАw32tm
    > Restart-Service w32time
    > w32tm /resync

## Настройка синхронизации времени (второй уровень)

**На rtr-r добавить правило в ACL**

    (config)# ip access-list extended SERVICES
    (config-ext-nacl)# permit udp any eq ntp host 4.4.4.100

**На SRV (если windows)**

    > Start-Service w32time
    > w32tm /config /syncfromflags:manual /manualpeerlist:"4.4.4.1"
    > w32tm /config /reliable:yes
    > Restart-Service w32time
    > w32tm /resync

**На srv (если linux)**

Добавляем записи в /etc/chrony/chrony.conf
  - server 4.4.4.1 iburst trust
  - allow 192.168.100.0/24
  - allow 172.16.100.0/24
  
Перезапускаем chronyd

## Настройка клиентов (внтуренних машин)

### На WinServer

Графика (описано выше для CLI) или

    > Start-Service w32time
    > w32tm /config /syncfromflags:manual /manualpeerlist:"192.168.100.200" //IP SRV
    > Restart-Service w32time
    > w32tm /resync

### На Linux

Добавляем запись в /etc/chrony/chrony.conf
  
- server 192.168.100.200 iburst
- Перезапускаем chronyd

### На Cisco

    (config)# ntp server 192.168.100.200

## Настройка SMB на SRV

### Настройка на SRV (Windows Server)

1. Настраиваем raid1

**Вариант 1 (графика)**

Выполнить -> mmc -> Ctrl+M -> Управление дисками

Переводим диски в online (ПКМ по диску -> В сети, Инициализировать -> GPT)

Создаем зеркало: ПКМ по пространству диска -> Создать зеркальный том -> Всё по подсказкам мастера (нужно будет воткнуть 2 диска)

**Вариант 2 (diskpart)**

    > diskpart
    > list disk // проверяем наличие дисков и их номера
    > select disk 1
    > online disk
    > attributes disk clear readonly
    > convert gpt
    > convert dynamic

То же самое повторить с disk 2

    > create volume mirror disk=1,2
    > format fs=ntfs quick
    > assign letter=R

Готово

2. Создание SMB шары

Устанавливаем роль файлового сервера и диспетчер ресурсов

**Вариант 1 (графика)**

ДС -> Управление -> Добавить роли и компоненты -> Роли сервера -> Файловые службы...: файловый сервер, Диспетчер ресурсов

**Вариант 2 (posh)**

    > Install-WindowsFeature -Name FS-Resource-Manager -IncludeManagementTools // ФС подцепит сам

Создаем шару

**Вариант 1 (графика)**

ДС -> Файловые службы и хранилища -> Общие ресурсы -> Клик по сцылке, Дальше по мастеру. На этапе "Общий ресурс" выбираем диск R. На этапе разрешения кликаем "Настройка разрешений"

- на вкладке "Отправить" устанавливаем для "Все" полный доступ
- на вкладке "Разрешения" добавляем для "Все" полный доступ
- Отключаем наследование


**Вариант 2 (posh)**
        
    > New-SmbShare -Name share -Path R:\shares -FullAccess "все" \\При EN локализации Everything


### Настройка на SRV (Linux)

1. Настраиваем raid1

Должен быть установлен mdadm и/или lvm2

**Вариант 1 (mdadm)** 

    # mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/vdb /dev/vdc

Создаем раздел

    # fdisk /dev/md0
    command (m for help): n
    Дальше enter до победного
    command (m for help): w

Создаем файловую систему

    # mkfs.ext4 /dev/md0p1

Монтируем ФС

    # mkdir /mnt/storage
    # mount /dev/md0p1 /mnt/storage
    # chmod -R a=rwx /mnt/storage

Правим /etc/fstab (чтоб маунт после перезагрузки не отвалился)

    /dev/md0p1 /mnt/storage ext4 defaults 0 0

**Вариант 2(lvm)**

Создаем физические тома LVM

    # pvcreate /dev/vdb /dev/vdc

Создаем группу томов

    # vgcreate DS /dev/vdb /dev/vdc

Создаем логический том типа зеркало

    # lvcreate --type mirror --name lv-share -l 100%FREE DS

Создаем ФС

    # mkfs.ext4 /dev/DS/lv-share

Монтируем

    # mount /dev/DS/lv-share /mnt/storage

Не забываем про chmod и /etc/fstab

2. Создание SMB шары

Должны быть установлены пакеты samba и smbclient

Редактируем файл /etc/samba/smb.conf
Удаляем всё, добавляем это:

    [global]
        workgroup = WORKGROUP
        security = user
        map to guest = bad user
        wins support = no
        dns proxy = no

    [share]
        path = /mnt/storage
        guest ok = yes
        force user = nobody
        browsable = yes
        writable = yes

## Монтирование шары на WEB-L, WEB-R

Создаем каталог /opt/share

монтируем:

    # mount -t cifs -o username=root,password=toor //192.168.100.200/share /opt/share

Пробуем с клиента создать каталог. Проверяем на сервере, что он появился


## Установка и настройка центра сертификации

### на SRV (windows server)

Установка центра сертификиции

**Вариант 1 (графика)**

Через добавление ролей и компонентов -> Службы сертификатов ... -> Центр сертификации

После установки в верху ДС появится восклицательный знак, где попросит настроить центр

В мастере:

- Службы ролей: центр сертификации
- Вариант установки: автономный
- Тип ЦС: Корневой
- Закрытый ключ: создать
- Шифрование: по дефолту
- Имя ЦС:
  - Общее имя: по дефолту
  - Суффикс: O=DEMO.WSR,C=RU
- Срок действия: любой, не менее 500 дней
- БД сертификатов:
  - БД: C:\CertDB
  - Журнал: C:\CertLog
  
Устанавливаем

**Вариант 2 (posh)**

    > Add-WindowsFeature ADCS-cert-authority -IncludeManagementTools
    > Add-WindowsFeature RSAT-ADCS-mgmt -IncludeManagementTools
    > Install-AdcsCertificationAuthority `
        -CACommonName MyORG-CA `
        -CAType EnterpriseRootCA `
        -CryptoProviderName "RSA#Microsoft Software Key Storage Provider" `
        -KeyLength 2048 `
        -HashAlgorithmName SHA256 `
        -ValidityPeriod Years `
        -ValidityPeriodUnits 5 `
        -DatabaseDirectory c:\CertDB `
        -LogDirectory c:\CertLog `
        -CADistinguishedNameSuffix "O=DEMO.WSR,C=RU"


[На главную](../index.md)
