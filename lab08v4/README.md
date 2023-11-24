# Lab 08 EIGRP IPv4

### Топология сети
<image src="scheme.png">

Задачи лабораторной работы:  

1. В офисе С.-Петербург настроить EIGRP.
2. R32 получает только маршрут по умолчанию.
3. R16-17 анонсируют только суммарные префиксы.
4. Использовать EIGRP named-mode для настройки сети.

### Маршрутизаторы R16-R17  
Включаем процесс EIGRP, присваиваем процессу Virtual Instance Name, указываем роутеру, что будем работать с адресами IPv4.    
Ниже приведен пример для R16. Для R17 настройки аналогичные.  
```
router eigrp AS2042
 address-family ipv4 unicast autonomous-system 2042
```
Теперь указываем анонсируемые сети:  
```
  topology base
  exit-af-topology
  network 14.0.3.0 0.0.0.3
  network 14.0.6.0 0.0.0.3
  network 192.168.8.0
  network 192.168.9.0
  eigrp router-id 16.16.16.16
```
Для роутера R18 мы передаем суммарный маршрут через интерфейс.  
192.168.8.0/24 + 192.168.9.0/24 = 192.168.8.0/23  
Эту сеть и будем передавать R18.  
```
  af-interface Ethernet0/1
   summary-address 192.168.8.0 255.255.254.0
  exit-af-interface
```
Аналогично с R17 передается суммарный маршрут на R18.  
Теперь таблица маршрутизации R18 выглядет так (оставлены маршруты, полученные по EIGRP):  
```
D        14.0.6.0/30 [90/1536000] via 14.0.3.2, 19:01:45, Ethernet0/0
      18.0.0.0/32 is subnetted, 1 subnets
D        32.32.32.32 [90/1536640] via 14.0.3.2, 18:59:46, Ethernet0/0
D     192.168.8.0/23 [90/1536000] via 14.0.3.2, 19:16:35, Ethernet0/0
                     [90/1536000] via 14.0.2.2, 19:16:35, Ethernet0/1
```
Пингуем с VPC лупбэк R18:  
```
VPCS> ping 18.18.18.18
84 bytes from 18.18.18.18 icmp_seq=1 ttl=254 time=1.523 ms
84 bytes from 18.18.18.18 icmp_seq=2 ttl=254 time=2.918 ms
```

Далее передаем с R16 маршрут по умолчанию на R32:  
```
 af-interface Ethernet0/3
   summary-address 0.0.0.0 0.0.0.0
  exit-af-interface
```
Теперь таблица маршрутизации R32 выглядет так:  
```
D*    0.0.0.0/0 [90/1536000] via 14.0.6.2, 18:51:49, Ethernet0/0
      14.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        14.0.6.0/30 is directly connected, Ethernet0/0
L        14.0.6.1/32 is directly connected, Ethernet0/0
      32.0.0.0/32 is subnetted, 1 subnets
C        32.32.32.32 is directly connected, Loopback0
```
Пингуем с VPC лупбэк R32:  
```
VPCS> ping 32.32.32.32
84 bytes from 32.32.32.32 icmp_seq=1 ttl=254 time=2.304 ms
84 bytes from 32.32.32.32 icmp_seq=2 ttl=254 time=3.014 ms
```












