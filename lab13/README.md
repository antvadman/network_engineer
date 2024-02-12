# Lab 13 VPN


Задачи лабораторной работы:  

1. Настроите GRE между офисами Москва и С.-Петербург.
2. Настроите DMVMN между Москва и Чокурдах, Лабытнанги.

### 1 Настроите GRE между офисами Москва и С.-Петербург.

Настроим туннельные интерфейсы на роутерах R15 и R18 соответственно:
```
R15
interface Tunnel0
 ip address 10.10.10.15 255.255.255.0
 ip tcp adjust-mss 1360
 tunnel source 12.0.0.2
 tunnel destination 14.0.4.1
end

R18
interface Tunnel0
 ip address 10.10.10.18 255.255.255.0
 ip tcp adjust-mss 1360
 tunnel source 14.0.4.1
 tunnel destination 12.0.0.2
end
```
Проверим связность:
```
R18#ping 10.10.10.15
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.10.15, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```
Связность роутеров через туннелированные интерфейсы есть, однако, чтобы сети за роутерами видели друг друга,
необходимо добавить маршруты через туннелированные интерфейсы.
Для R15:
```
S     192.168.8.0/24 [1/0] via 10.10.10.18
S     192.168.9.0/24 [1/0] via 10.10.10.18

```
Для R18:
```
S     192.168.0.0/24 [1/0] via 10.10.10.15
S     192.168.1.0/24 [1/0] via 10.10.10.15

```
### 2 Настроите DMVMN между Москва и Чокурдах, Лабытнанги.

Сконфигурируем туннельный интерфейс для hub:
```
interface Tunnel100
 ip address 100.100.100.15 255.255.255.0
 no ip redirects
 ip nhrp network-id 100
 tunnel source Ethernet0/2
 tunnel mode gre multipoint
end
```
Теперь настраиваем spoke R27 и R28 соответственно:
```
R27
interface Tunnel100
 ip address 100.100.100.27 255.255.255.0
 no ip redirects
 ip nhrp map 100.100.100.15 12.0.0.2
 ip nhrp map multicast 12.0.0.2
 ip nhrp network-id 100
 ip nhrp nhs 100.100.100.15
 tunnel source Ethernet0/0
 tunnel mode gre multipoint

R28
interface Tunnel100
 ip address 100.100.100.28 255.255.255.0
 no ip redirects
 ip nhrp map 100.100.100.15 12.0.0.2
 ip nhrp map multicast 12.0.0.2
 ip nhrp network-id 100
 ip nhrp nhs 100.100.100.15
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
```

Проверяем доступность туннельных интерфейсов hub-to-spoke и spoke-to-spoke:
```
R28#ping 100.100.100.15
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 100.100.100.15, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms

R28#ping 100.100.100.27
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 100.100.100.27, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```
DMVPN на hub:
```
R15#sh ip nhrp
100.100.100.27/32 via 100.100.100.27
   Tunnel100 created 00:06:49, expire 00:09:50
   Type: dynamic, Flags: registered nhop
   NBMA address: 16.0.0.1
100.100.100.28/32 via 100.100.100.28
   Tunnel100 created 00:04:29, expire 00:08:50
   Type: dynamic, Flags: registered nhop
   NBMA address: 15.0.0.1


Interface: Tunnel100, IPv4 NHRP Details
Type:Hub, NHRP Peers:2,

 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1 16.0.0.1         100.100.100.27    UP 00:07:18     D
     1 15.0.0.1         100.100.100.28    UP 00:04:58     D
```

