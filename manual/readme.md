# A1 Виртуальные машины и коммутация
## Настройка имен хостов

### Windows (Server & Desktop)

Вариант 1. Графические утилиты
Вариант 2. PowerShell (от администратора):

    > Rename-Computer -NewName <name> | shutdown /r /t 0

### Linux

    # hostnamectl set-hostname <name> && reboot

### CSR

    (config)# hostname <name>

## Настройка адресов

### Windows (Server & Desktop)

Вариант 1. Графические утилиты
Вариант 2. PowerShell (от администратора):
    
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
### Вариант 1 (GRE over IPsec)

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

    (config)# crypto ipsec transform-set TS esp-aes esp-sha256-hmac
    (cfg-crypto-trans)# mode tunnel
    (config)# crypto ipsec profile VPN
    (ipsec-profile)# set transform-set TS

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

### Вариант 2 (IPsec)

Будет чуть позже

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


[На главную](../README.md)
