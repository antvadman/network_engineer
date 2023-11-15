# Lab 07 ISIS IPv4

### Топология сети
<image src="scheme.png">

Задачи лабораторной работы:  

1. Настроите IS-IS в ISP Триада.
2. R23 и R25 находятся в зоне 2222.
3. R24 находится в зоне 24.
4. R26 находится в зоне 26.

### Маршрутизаторы R23-R25  
Включаем процесс ISIS, присваиваем роутеру Network Entity Title (NET), включаем ISIS на интерфейсе, подключенном к сети, которую мы хотим анонсировать в ISIS.    
Ниже приведен пример для R23. Для R25 настройки аналогичные.  
```
router isis
 net 49.2222.0000.0000.0023.00
interface Loopback0
 ip address 23.23.23.23 255.255.255.255
 ip router isis
interface Ethernet0/1
 ip address 13.0.0.1 255.255.255.252
 ip router isis
interface Ethernet0/2
 ip address 13.0.1.1 255.255.255.252
 ip router isis
```
### Маршрутизаторы R24-R26
Присваиваем роутеру NET с зоной 24. Аналогично предыдущим настройкам включаем ISIS на интерфейсах:    
```
router isis
 net 49.0024.0000.0000.0024.00
interface Ethernet0/1
 ip address 13.0.3.2 255.255.255.252
 ip router isis
 duplex auto
!
interface Ethernet0/2
 ip address 13.0.1.2 255.255.255.252
 ip router isis
 duplex auto
 
 interface Loopback0
 ip address 24.24.24.24 255.255.255.255
 ip router isis
```
Для R26 изменится только область и интерфейсы:  
```
router isis
 net 49.00226.0000.0000.00026.00
 
interface Ethernet0/0
 ip address 13.0.3.1 255.255.255.252
 ip router isis
 duplex auto

interface Ethernet0/2
 ip address 13.0.2.1 255.255.255.252
 ip router isis
 duplex auto

interface Loopback0
 ip address 26.26.26.26 255.255.255.255
 ip router isis
```
В результате роутеры R23,24,25,26 будут знать как добраться друг до друга и анонсируемых ими сетей.  
Так выглядит таблица маршрутизатора R26:  
```
R26#show ip route isis
Gateway of last resort is not set
      13.0.0.0/8 is variably subnetted, 6 subnets, 2 masks
i L2     13.0.0.0/30 [115/20] via 13.0.2.2, 21:01:40, Ethernet0/2
i L2     13.0.1.0/30 [115/20] via 13.0.3.2, 21:01:20, Ethernet0/0
      23.0.0.0/32 is subnetted, 1 subnets
i L2     23.23.23.23 [115/30] via 13.0.3.2, 21:01:20, Ethernet0/0
                     [115/30] via 13.0.2.2, 21:01:20, Ethernet0/2
      24.0.0.0/32 is subnetted, 1 subnets
i L2     24.24.24.24 [115/20] via 13.0.3.2, 21:01:20, Ethernet0/0
      25.0.0.0/32 is subnetted, 1 subnets
i L2     25.25.25.25 [115/20] via 13.0.2.2, 21:01:40, Ethernet0/2
```
Проверим доступность лупбэка R23 с R26:
```
R26#ping 23.23.23.23
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 23.23.23.23, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```









