# Lab 12 Основные протоколы сети интернет
### Топология сети
<image src="scheme.png">

Задачи лабораторной работы:  

1. Настроите NAT(PAT) на R14 и R15. Трансляция должна осуществляться в адрес автономной системы AS1001.
2. Настроите NAT(PAT) на R18. Трансляция должна осуществляться в пул из 5 адресов автономной системы AS2042.
3. Настроите статический NAT для R20.
4. Настроите NAT так, чтобы R19 был доступен с любого узла для удаленного управления.
5. Настроите для IPv4 DHCP сервер в офисе Москва на маршрутизаторах R12 и R13. VPC1 и VPC7 должны получать сетевые настройки по DHCP.
6. Настроите NTP сервер на R12 и R13. Все устройства в офисе Москва должны синхронизировать время с R12 и R13.

### 1 Настроите NAT(PAT) на R14 и R15. Трансляция должна осуществляться в адрес автономной системы AS1001
Ниже приведены настройки для R15, для R14 аналогичные.
Сначала конфигурируем внешние и внутренние интерфейсы:
```
interface Ethernet0/2
 ip nat outside

R15(config)#int e0/0
R15(config-if)#ip nat inside

R15(config)#int e1/0
R15(config-if)#ip nat inside
```
Потом создаем ACL, разрешающий выход во внешнюю сеть.
Тк у нас сети источников 192.168.0.0/24, и 192.168.1.0/24, то
агрегатом будет сеть 192.168.0.0/23. Ее мы укажем в ACL:
```
R15(config)#access-list 11 permit 192.168.0.0 0.0.1.255
```
Далее конфигурируем сам РАТ:
```
R15(config)#ip nat inside source list 11 interface e0/2 overload
```
Проверяем. Пингуем VPS1, VPS7, расположенные в AS 1001 с R24.
```
R24#ping 192.168.0.4
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.0.4, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/3 ms
R24#ping 192.168.1.4
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.4, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```
И смотрим трансляции адресов/портов
```
R15#show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
icmp 12.0.0.2:0        192.168.0.4:0      12.0.1.2:0         12.0.1.2:0
icmp 12.0.0.2:1        192.168.1.4:1      12.0.1.2:1         12.0.1.2:1
```

### 2 Настроите NAT(PAT) на R18. Трансляция должна осуществляться в пул из 5 адресов автономной системы AS2042
Сначала анонсируем стык в BGP на R24, тк в адрес стыка на R18 будут транслироваться адреса источников.
```
R24(config)#router bgp 2222
R24(config-router)#network 14.0.4.0 mask 255.255.255.0
```
То же самое делаем для другого линка:
```
R24(config)#router bgp 2222
R24(config-router)#network 14.0.5.0 mask 255.255.255.0
```
Тк у нас сети источников 192.168.8.0/24, и 192.168.9.0/24, то
агрегатом будет сеть 192.168.8.0/23. Ее мы укажем в ACL:
```
access-list 11 permit 192.168.8.0 0.0.1.255
```
Создадим 2 пула (по одному для каждого линка) по 5 адресов из стыковочной сети:
```
ip nat pool POOL18 14.0.4.3 14.0.4.7 netmask 255.255.255.0
ip nat pool POOL189 14.0.5.3 14.0.5.7 netmask 255.255.255.0
```
Далее настраиваем РАТ:
```
ip nat inside source list 11 pool POOL18 overload
ip nat inside source list 11 pool POOL189 overload
```
Запускаем пинг из разных сетей и проверяем статистику:
```
R18#sh ip nat statistics
Total active translations: 12 (0 static, 12 dynamic; 12 extended)
Peak translations: 37, occurred 00:06:49 ago
Outside interfaces:
  Ethernet0/2
Inside interfaces:
  Ethernet0/0, Ethernet0/1
Hits: 215  Misses: 0
CEF Translated packets: 215, CEF Punted packets: 0
Expired translations: 121
Dynamic mappings:
-- Inside Source
[Id: 2] access-list 11 pool POOL18 refcount 12
 pool POOL18: netmask 255.255.255.0
        start 14.0.4.3 end 14.0.4.7
        type generic, total addresses 5, allocated 1 (20%), misses 0

Total doors: 0
Appl doors: 0
Normal doors: 0
Queued Packets: 0

R18#sh ip nat stat
Total active translations: 60 (0 static, 60 dynamic; 60 extended)
Peak translations: 61, occurred 00:29:14 ago
Outside interfaces:
  Ethernet0/2, Ethernet0/3
Inside interfaces:
  Ethernet0/0, Ethernet0/1
Hits: 1297  Misses: 0
CEF Translated packets: 1297, CEF Punted packets: 0
Expired translations: 614
Dynamic mappings:
-- Inside Source
[Id: 2] access-list 11 pool POOL189 refcount 60
 pool POOL189: netmask 255.255.255.0
        start 14.0.5.3 end 14.0.5.7
        type generic, total addresses 5, allocated 1 (20%), misses 0

Total doors: 0
Appl doors: 0
Normal doors: 0
Queued Packets: 0
```
Фрагмент таблицы трансляций:
```
R18#sh ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
icmp 14.0.5.3:11644    192.168.9.11:11644 19.19.19.19:11644  19.19.19.19:11644
icmp 14.0.5.3:11900    192.168.9.11:11900 19.19.19.19:11900  19.19.19.19:11900
icmp 14.0.5.3:12156    192.168.9.11:12156 19.19.19.19:12156  19.19.19.19:12156
icmp 14.0.5.3:12412    192.168.9.11:12412 19.19.19.19:12412  19.19.19.19:12412
icmp 14.0.5.3:12668    192.168.9.11:12668 19.19.19.19:12668  19.19.19.19:12668
icmp 14.0.5.3:12924    192.168.9.11:12924 19.19.19.19:12924  19.19.19.19:12924
icmp 14.0.5.3:13180    192.168.9.11:13180 19.19.19.19:13180  19.19.19.19:13180
icmp 14.0.5.3:13436    192.168.9.11:13436 19.19.19.19:13436  19.19.19.19:13436
icmp 14.0.5.3:13692    192.168.9.11:13692 19.19.19.19:13692  19.19.19.19:13692
icmp 14.0.5.3:13948    192.168.9.11:13948 19.19.19.19:13948  19.19.19.19:13948
icmp 14.0.5.3:14204    192.168.9.11:14204 19.19.19.19:14204  19.19.19.19:14204
icmp 14.0.5.3:14460    192.168.9.11:14460 19.19.19.19:14460  19.19.19.19:14460
icmp 14.0.5.3:14716    192.168.9.11:14716 19.19.19.19:14716  19.19.19.19:14716
icmp 14.0.5.3:14972    192.168.9.11:14972 19.19.19.19:14972  19.19.19.19:14972
icmp 14.0.5.3:15228    192.168.9.11:15228 19.19.19.19:15228  19.19.19.19:15228
icmp 14.0.5.3:15484    192.168.9.11:15484 19.19.19.19:15484  19.19.19.19:15484
```

### 3. Настроите статический NAT для R20
Чтобы пакеты, отправленные с R20, натировались на R15, на R15 настраиваем статический NAT:
```
ip nat inside source static 20.20.20.20 15.15.15.15
```

### 4. Настроите NAT так, чтобы R19 был доступен с любого узла для удаленного управления
Здесь мы должны применить проброс портов.
Сначала настроим доступ до R19 по ssh.
Для этого сгенерируем ключи RSA длиной 768 бит (для ssh v2) и разрешим подключение к терминалу.
```
R19(config)#crypto key generate rsa
R19(config)#ip ssh version 2
R19(config)#line vty 0 4
R19(config-line)#transport input ssh
```
На R15 проанонсируем стыковочную сеть 12.0.0.0/30, тк на адрес R15 из этой сети будут подключаться клиенты.
Далее на R15 пробросим порты.
```
ip nat inside source static tcp 19.19.19.19 22 12.0.0.2 2222 extendable
```
Теперь пакеты, пришедшие на 12.0.0.2 и порт 2222 будут перенаправляться на 19.19.19.19 порт 22.
Проверим доступность по ssh с VPC офиса СПБ (АС 2042):
```
VPCS>  ping 12.0.0.2 -P 6 -p 2222
Connect   2222@12.0.0.2 seq=1 ttl=248 time=2.147 ms
SendData  2222@12.0.0.2 seq=1 ttl=248 time=1.067 ms
Close     2222@12.0.0.2 timeout(9.623ms)
```
Таблица трансляций (фрагмент) на R15 теперь выглядит так:
```
R15#sh ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
tcp 12.0.0.2:2222      19.19.19.19:22     14.0.5.3:24537     14.0.5.3:24537
```

### 5. Настроите для IPv4 DHCP сервер в офисе Москва на маршрутизаторах R12 и R13. VPC1 и VPC7 должны получать сетевые настройки по DHCP
Исключаем из пула адреса, которые не должны выдаваться:
```
ip dhcp excluded-address 192.168.0.252
ip dhcp excluded-address 192.168.0.253
ip dhcp excluded-address 192.168.0.254
ip dhcp excluded-address 192.168.0.1
ip dhcp excluded-address 192.168.0.2
ip dhcp excluded-address 192.168.0.3
ip dhcp excluded-address 192.168.1.252
ip dhcp excluded-address 192.168.1.253
ip dhcp excluded-address 192.168.1.254
ip dhcp excluded-address 192.168.1.1
ip dhcp excluded-address 192.168.1.2
ip dhcp excluded-address 192.168.1.3
```
Настраиваем пулы адресов, а также шлюз по умолчанию:
```
ip dhcp pool VLAN10_POOL
 network 192.168.0.0 255.255.255.0
 default-router 192.168.0.254

ip dhcp pool VLAN20_POOL
 network 192.168.1.0 255.255.255.0
 default-router 192.168.1.254
```
Результат настройки:
```
VPCS> ip dhcp
DORA IP 192.168.0.4/24 GW 192.168.0.254
```

### 6. Настроите NTP сервер на R12 и R13. Все устройства в офисе Москва должны синхронизировать время с R12 и R13
Настройки приведены для R12 (для R13 аналогично).
Сначала задаем стратум, равный 3, далее синхронизируем хардварное и софтварное время.
```
ntp master 3
ntp update-calendar
```
На ntp клиентах задаем адреса серверов ntp:
```
ntp server 12.12.12.12
ntp server 13.13.13.13
```
Результат синхронизации:
```
R14#sh ntp s
Clock is synchronized, stratum 9, reference is 12.12.12.12
nominal freq is 250.0000 Hz, actual freq is 250.0000 Hz, precision is 2**10
ntp uptime is 254400 (1/100 of seconds), resolution is 4000
reference time is E94EBDB4.A3D70C00 (20:08:20.640 UTC Sun Jan 14 2024)
clock offset is 0.0000 msec, root delay is 0.00 msec
root dispersion is 75.14 msec, peer dispersion is 69.69 msec
loopfilter state is 'CTRL' (Normal Controlled Loop), drift is 0.000000000 s/s
system poll interval is 128, last update was 108 sec ago.
```

