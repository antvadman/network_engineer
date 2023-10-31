# Lab 06 OSPF

### Топология сети
<image src="scheme.png">

Задачи лабораторной работы:  

1. Маршрутизаторы R14-R15 находятся в зоне 0 - backbone.
2. Маршрутизаторы R12-R13 находятся в зоне 10. Дополнительно к маршрутам должны получать маршрут по умолчанию.
3. Маршрутизатор R19 находится в зоне 101 и получает только маршрут по умолчанию.
4. Маршрутизатор R20 находится в зоне 102 и получает все маршруты, кроме маршрутов до сетей зоны 101.

### Маршрутизаторы R14-R15  
Для помещения роутеров в бэкбон, мы включаем процесс OSPF, присваиваем роутеру идентификатор, аннонсируем сети в области:  
Ниже приведен пример для R14. Для R15 настройки аналогичные.  
```
router ospf 1
router-id 14.14.14.14
redistribute connected subnets
network 10.0.0.0 0.0.0.7 area 0
network 10.0.1.0 0.0.0.7 area 0
```
### Маршрутизаторы R12-R13
Часть интерфейсов R12-R13 будет помещена в бэкбон, другие, согласно условию задачи, помещаются в зону 10.  
```
network 192.168.0.0 0.0.0.255 area 10
network 192.168.1.0 0.0.0.255 area 10
```
Для получения маршрута по умолчанию на роутерах R14-R15 выполняется команда default-information originate always.  
Теперь роутеры R12-R13 получат 2 маршрута по умолчанию и будут балансировать трафик (ECMP).  
Вывод таблицы маршрутизации R12:
```
O*E2  0.0.0.0/0 [110/1] via 10.0.1.4, 00:03:08, Ethernet0/3
                [110/1] via 10.0.0.2, 00:05:17, Ethernet0/2
      10.0.0.0/8 is variably subnetted, 6 subnets, 3 masks
O IA     10.0.2.0/30 [110/20] via 10.0.0.2, 00:14:35, Ethernet0/2
O IA     10.0.3.0/30 [110/20] via 10.0.1.4, 00:14:35, Ethernet0/3
      11.0.0.0/30 is subnetted, 1 subnets
O E2     11.0.0.0 [110/20] via 10.0.0.2, 00:14:35, Ethernet0/2
      12.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
O E2     12.0.0.0/30 [110/20] via 10.0.1.4, 00:14:35, Ethernet0/3
      13.0.0.0/32 is subnetted, 1 subnets
O E2     13.13.13.13 [110/20] via 192.168.1.253, 00:14:35, Ethernet0/1.20
                     [110/20] via 192.168.0.253, 00:14:35, Ethernet0/0.10
      14.0.0.0/32 is subnetted, 1 subnets
O E2     14.14.14.14 [110/20] via 10.0.0.2, 00:14:35, Ethernet0/2
      15.0.0.0/32 is subnetted, 1 subnets
O E2     15.15.15.15 [110/20] via 10.0.1.4, 00:14:35, Ethernet0/3
      20.0.0.0/32 is subnetted, 1 subnets
O E2     20.20.20.20 [110/20] via 10.0.1.4, 00:14:35, Ethernet0/3
```

### Маршрутизатор R19
Помещаем роутер R19 в зону 101   
```
network 10.0.2.0 0.0.0.3 area 101
```
Делаем зону Stub  
```
area 101 stub
```
На R14 выполняем команду  
```
area 101 stub no-summary
```
Теперь таблица маршрутизации R19 выглядит так  
```
O*IA  0.0.0.0/0 [110/11] via 10.0.2.2, 1d06h, Ethernet0/0
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.0.2.0/30 is directly connected, Ethernet0/0
L        10.0.2.1/32 is directly connected, Ethernet0/0
      19.0.0.0/32 is subnetted, 1 subnets
C        19.19.19.19 is directly connected, Loopback0
```
### Маршрутизатор R20
Помещаем роутер R20 в зону 102   
```
network 10.0.3.0 0.0.0.3 area 102
```
Здесь будем фильтровать маршруты.  
Создаем ACL, которым будем матчить сети, маршруты до которых мы будем отфильтровывать.  
```
access-list 1 deny   10.0.2.0 0.0.0.3
access-list 1 permit any
```
Далее создаем distribute list, в котором будем ссылаться на ACL  
```
distribute-list 1 in
```
Так выгладела ТМ до фильтрации, пинг в зону 101 проходит  
```
      10.0.0.0/8 is variably subnetted, 5 subnets, 3 masks
O IA     10.0.0.0/29 [110/20] via 10.0.3.2, 00:00:08, Ethernet0/0
O IA     10.0.1.0/29 [110/20] via 10.0.3.2, 00:00:08, Ethernet0/0
O IA     10.0.2.0/30 [110/40] via 10.0.3.2, 00:00:08, Ethernet0/0
C        10.0.3.0/30 is directly connected, Ethernet0/0

R20#ping 10.0.2.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.2.1, timeout is 2 seconds:
!!!!!
```
ТМ после фильтрации, пинг в зону 101 пропал  
```
      10.0.0.0/8 is variably subnetted, 4 subnets, 3 masks
O IA     10.0.0.0/29 [110/20] via 10.0.3.2, 00:00:11, Ethernet0/0
O IA     10.0.1.0/29 [110/20] via 10.0.3.2, 00:00:11, Ethernet0/0
C        10.0.3.0/30 is directly connected, Ethernet0/0

R20#ping 10.0.2.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.2.1, timeout is 2 seconds:
.....
```


