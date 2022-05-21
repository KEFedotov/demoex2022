# A1 Виртуальные машины и коммутация
## O1 hostname

Проверка на всех хостах, если соответствует Таблице 1 на всех -> **A1.O1 PASSED**

В любом другом случае -> **A1.O1 FAILED**

### Linux

    # hostname

### Windows

    > get-computerinfo -property csname

### CSR

    # show running-config | include hostname

## O2 Адресация

Проверка на всех хостах всех интерфейсов, если соответствует таблице 1 -> **A1.O2 PASSED**

В любом другом случае -> **A1.O2 FAILED**

### Linux

    # ip address show <iface> | grep 'inet '

### Windows

    > get-netipaddress | select <ip>

### CSR

    # show ip interface brief | include <ip>

# B1 Сетевая связность
## O1 Запрет прямого попадания внутреннего трафика во внешние сети

Проверка на ISP

### Linux

Запуск **tcpdump (должен быть установлен)** на интерфейсе, смотрящем на RTR

### Windows Server

Запуск **pktmon** на интерфейсе, смотрящем на RTR

Запуск пинга с внутренних машин (SRV, WEB-L, WEB-R) на соответствующий интерфейс ISP -> анализируем перехваченный трафик

### Интерпретация 

Пинг хотя бы с одной машины успешен **И** в заголовках отсутствуют внутренние адреса (192.168.100.0/24, 172.16.100.0/24) -> **B1.O1 PASSED**

В любом другом случае -> **B1.O1 FAILED**

## O2 NAT

Запуск пинга с внутренних машин (SRV, WEB-L, WEB-R) на соответствующий интерфейс ISP

### RTR - Cisco

На соответствующем RTR

    # show ip nat translations

### Интерпретация

Пинг хотя бы с одной машины успешен **И** в выводе преобразования есть запись: внешний IP RTR - IP внутренней машины -> **B1.O2 PASSED**

В любом другом случае -> **B1.O2 FAILED**

## O3 Наличие туннеля между платформами

### RTR - Cisco

С RTR-L запускаем traceroute на внутренний IP RTR-R

    # traceroute <ip>

### Интерпретация

Traceroute успешен **И** не содержит IP-адресов ISP -> **B1.O3 PASSED**

В любом другом случае -> **B1.O3 FAILED**

## O4 Туннель между платформами защищен

На ISP запускаем tcpdump/pktmon на интерфейсе, смотрящем на RTR-L

С RTR-L запускаем ping на внутренний IP RTR-R

    # ping <ip>

### Интерпретация

Если ping успешен **И** в выводе tcpdump/pktmon нет строки 'ICMP echo' -> **B1.O4 PASSED**

В любом другом случае -> **B1.O4 FAILED**

## O5 Трафик во внешние сети проходит без использования туннеля

С любой внутренней машины запускаем трассировку CLI

### Linux

    # traceroute <cli-ip>

### Windows

    > tracert <cli-ip>

### Интепретация

Если трассировка успешна **И** в выводе присутствует IP-адрес ISP -> **B1.O5 PASSED**

В любом другом случае -> **B1.O5 FAILED**

## O6 Трафик во внешние сети проходит без использования туннеля

С любой внутренней машины LEFT запускаем трассировку WEB-R

### Linux

    # traceroute <web-r-ip>

### Windows

    > tracert <web-r-ip>

### Интепретация

Если трассировка успешна **И** в выводе отсутствует любой из IP-адресов ISP -> **B1.O6 PASSED**

В любом другом случае -> **B1.O6 FAILED**

## O7 Трафик, идущий по туннелю не транслируется


### RTR - Cisco

На RTR-L очищаем таблицу NAT

    # clear ip nat translation *

С любой внутренней машины LEFT запускаем ping внутреннего интерфейса RTR-R

На RTR-L проверяем таблицу NAT

    # show ip nat translations

### Интерпретация

Если ping успешен **И** в выводе таблицы нет преобразования внутреннего IP машины во внешний RTR-L -> **B1.07 PASSED**

В любом другом случае -> **B1.O7 FAILED**

## O8 Контроль входящего трафика на RTR-L

### RTR - Cisco

Проверяем входящий ACL на внешнем интерфейсе

    # show ip access-lists interface <ifname> in

1. Ищем следующие верные сопоставления:

- permit gre ...
- permit icmp ...
- permit esp ...
- permit tcp ... domain www 443 2222 (могут быть отдельными ACE)
- permit udp ... domain

2. Ищем костыль:

- permit ip any any

### Интерпретация

Если есть входящий ACL И записи п.1 найдены все И запись п.2 не найдена -> **B1.O8 PASSED**

В любом другом случае -> **B1.O8 FAILED**

## O9 Контроль входящего трафика на RTR-L

### RTR - Cisco

Проверяем входящий ACL на внешнем интерфейсе

    # show ip access-lists interface <ifname> in

1. Ищем следующие верные сопоставления:

- permit gre ...
- permit icmp ...
- permit esp ...
- permit tcp ... www 443 2222 (могут быть отдельными ACE)

1. Ищем костыль:

- permit ip any any

### Интерпретация

Если есть входящий ACL И записи п.1 найдены все И запись п.2 не найдена -> **B1.O9 PASSED**

В любом другом случае -> **B1.O9 FAILED**

## O10 Настройка служб ssh региона LEFT

Пробуем по ssh с CLI подключиться к внешнему IP RTR-L по порту 2222 и запустить команду вывода IP

    > ssh root@4.4.4.100 -p 2222 ip address show

### Интерпретация

Если в выводе внутренний IP машины WEB-L -> **B1.10 PASSED**

В любом другом случае -> **B1.10 FAILED**

## O11 Настройка служб ssh региона RIGHT

Пробуем по ssh с CLI подключиться к внешнему IP RTR-R по порту 2244 и запустить команду вывода IP

    > ssh root@5.5.5.100 -p 2244 ip address show

### Интерпретация

Если в выводе внутренний IP машины WEB-R -> **B1.11 PASSED**

В любом другом случае -> **B1.11 FAILED**


# C1 Инфраструктурные службы

Во всех аспектах, связанных с DNS, если проверка путём резолвинга запускается с WinServer - используем Resolve-DnsName _name_ -DnsOnly, если с Linux - dig _name_. Струкутра того, что ищем в выводе - почти идентична

**На Linux должен быть установлен dnsutils**

## O1 DNS первого уровня настроен в соответствии с заданием

С ISP запускаем проверку машин зоны 

    # dig isp.demo.wsr
    # dig www.demo.wsr
    # dig internet.demo.wsr

Структура ответа:

    ;; ANSWER SECTION:
    isp.demo.wsr.   ... IN  A   3.3.3.1
    
    ;; ANSWER SECTION:
    www.demo.wsr.   ... IN  A   4.4.4.100
    www.demo.wsr.   ... IN  A   5.5.5.100
    
    ;; ANSWER SECTION:
    internet.demo.wsr.   ... IN  CNAME   isp.demo.wsr.
    isp.demo.wsr.   ... IN  A   3.3.3.1

### Интерпретация

Если ВСЕ три ответа соответствуют -> **C1.O1 PASSED**

В любом другом случае -> **C1.O1 FAILED**

## O2 DNS второго уровня настроен в соответствии с заданием

С SRV запускаем проверку машин зоны

    > Resolve-DnsName -Name srv.int.demo.wsr -DnsOnly
    > Resolve-DnsName -Name web-l.int.demo.wsr -DnsOnly
    > Resolve-DnsName -Name web-r.int.demo.wsr -DnsOnly
    > Resolve-DnsName -Name rtr-l.int.demo.wsr -DnsOnly
    > Resolve-DnsName -Name rtr-r.int.demo.wsr -DnsOnly
    > Resolve-DnsName -Name dns.int.demo.wsr -DnsOnly
    > Resolve-DnsName -Name ntp.int.demo.wsr -DnsOnly

Структура ответа

    srv.int.demo.wsr    A   ... 192.168.100.200
    web-l.int.demo.wsr    A   ... 192.168.100.100
    web-r.int.demo.wsr    A   ... 172.16.100.100
    rtr-l.int.demo.wsr    A   ... 192.168.100.254
    rtr-r.int.demo.wsr    A   ... 172.16.100.254
    dns.int.demo.wsr    CNAME   ... srv.int.demo.wsr
    ntp.int.demo.wsr    CNAME   ... srv.int.demo.wsr

### Интерпретация

Если ВСЕ ответы соответствуют -> **C1.O2 PASSED**

В любом другом случае -> **C1.O2 FAILED**

## O3 DNS второго уровня обслуживает обратные зоны

Прверка на SRV

    > Resolve-DnsName -Name 192.168.100.100 -DnsOnly
    > Resolve-DnsName -Name 192.168.100.200 -DnsOnly
    > Resolve-DnsName -Name 192.168.100.254 -DnsOnly
    > Resolve-DnsName -Name 172.16.100.100 -DnsOnly
    > Resolve-DnsName -Name 172.16.100.254 -DnsOnly

Структура вывода
    
    100.100.168.192.in-addr.arpa    PTR ... web-l.int.demo.wsr
    200.100.168.192.in-addr.arpa    PTR ... srv.int.demo.wsr
    254.100.168.192.in-addr.arpa    PTR ... rtr-l.int.demo.wsr
    100.100.16.172.in-addr.arpa    PTR ... web-r.int.demo.wsr
    254.100.16.172.in-addr.arpa    PTR ... rtr-r.int.demo.wsr

### Интерпретация

Если ВСЕ ответы соответствуют -> **C1.O3 PASSED**

В любом другом случае -> **C1.O3 FAILED**

## O4 DNS-сервер ISP делегирует зону int.demo.wsr на SRV

Проверка с ISP

    # dig int.demo.wsr
    
Структура вывода

    ;; AUTHORITY SECTION:
    int.demo.wsr.   ... IN  NS   srv.int.demo.wsr
    ...
    ;; ADDITIONAL SECTION:
    srv.int.demo.wsr.   ... IN  A   4.4.4.100 ; Адрес RTR-L!!!

### Интерпретация

Если ответ соответствует -> **C1.O4 PASSED**

В любом другом случае -> **C1.O4 FAILED**

## O5 CLI использует DNS-сервер ISP по умолчанию

Проверка на CLI

    > Get-DnsClientServerAddress

Структура вывода

    Ethernet    ... IPv4    {3.3.3.1}   //IP ISP

### Интерпретация

Если ответ соответствует -> **C1.O5 PASSED**

В любом другом случае -> **C1.O5 FAILED**

## O6 SRV обслуживает рекурсивные запросы от внутренних машин

Проверка с WEB-L (или WEB-R)

    #dig isp.demo.wsr

Структура ответа

    ;; ANSWER SECTION
    isp.demo.wsr    ... A   3.3.3.1

### Интерпретация

Если ответ соответствует -> **C1.O6 PASSED**

В любом другом случае -> **C1.O6 FAILED**

## O7 Внутренние хосты используют DNS-службу SRV

Проверка с WEB-L (или WEB-R)

    # cat /etc/resolv.conf
    # cat /etc/hosts

Структура ответа

В первой команде должны увидеть только:

    # nameserver 192.168.100.200
    
В выводе 2 команды должны увидеть дефолтный файл hosts (не должен содержать такие имена как srv.int.demo.wsr, isp.demo.wsr и т.д.)

### Интерпретация

Если оба отчета соответствуют -> **C1.O7 PASSED**

В любом другом случае -> **C1.O7 FAILED**

## O8 Первый уровень системы синхронизации времени

Проверка на ISP

**Если ISP - Linux**

    # cronyc tracking

Ищем:

- Stratum - 4
- Leap status: normal

**Если ISP - WS**

    # w32tm /query /status

Ищем:

- Страта: 4
- Источник: 3.3.3.1 # тут любой из адресов ISP, включая localhost
  
### Интерпретация

Если оба ответа соответствуют -> **C1.O8 PASSED**

В любом другом случае -> **C1.O8 FAILED**

## O9 ISP дополнительные настройки NTP

### Допускается подключение только от RTR-L

Смотрим /etc/chrony/chrony.conf

- должно присутствовать allow 4.4.4.100 # Внешний IP rtr-l
- Должны отсутствовать allow 192.168.100.0/24 и 172.16.100.0/24 (внтренние сети)

### ISP считает собственный источник времени доверенным

Смотрим /etc/chrony/chrony.conf

- Должна быть запись типа `server <ip | name> **trust** ...


### Интерпретация

Если все ответы соответствуют -> **C1.O9 PASSED**

В любом другом случае -> **C1.O9 FAILED**

### Собственный источник доверенный


[На главную](../README.md)
