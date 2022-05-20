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

Редактируем /etc/network/interfaces.d/setup или /etc/network/interfaces (если setup нет). Настройка интерфейса одинаковая:

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

        forwarders {
            4.4.4.100;
        };
        allow-recursion { none; };

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


[На главную](../index.md)
