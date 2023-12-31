# Lab 04 Архитектура сети

### Топология сети
<image src="scheme.png">

Задачи лабораторной работы:  
1. Разработаете и задокументируете адресное пространство для лабораторного стенда.  
2. Настроите ip адреса на каждом активном порту.    
3. Настроите каждый VPC в каждом офисе в своем VLAN.  
4. Настроите VLAN/Loopbackup interface управления для сетевых устройств.  
5. Настроите сети офисов так, чтобы не возникало broadcast штормов, а использование линков было максимально оптимизировано. 

## Москва  

На коммутаторах SW4 и SW5 настроены vlan 10,20,666 (MGMT), а также VRRP.  
Линки, соединяющие коммутаторы, агрегированы (протокол LACP). 
На них настроен trunk, по которому передается трафик vlan'ов 10, 20 и 666:
```
interface Port-channel1
 switchport trunk allowed vlan 10,20,666
 switchport trunk encapsulation dot1q
 switchport mode trunk
```
На коммутаторах настроена балансировка по vlan путем назначения приоритетов на SVI интерфейсах:  

#### SW4  
```
interface Vlan10
 ip address 192.168.0.2 255.255.255.0 
 vrrp 10 ip 192.168.0.1  
 vrrp 10 priority 120  
!  
interface Vlan20  
ip address 192.168.1.2 255.255.255.0  
vrrp 20 ip 192.168.1.1  
vrrp 20 priority 99       
```
#### SW5  
```
interface Vlan10  
 ip address 192.168.0.2 255.255.255.0  
 vrrp 10 ip 192.168.0.1  
 vrrp 10 priority 99  
!
interface Vlan20  
 ip address 192.168.1.2 255.255.255.0  
 vrrp 20 ip 192.168.1.1  
 vrrp 20 priority 120
 ```
 
На маршрутизаторах R12 и R13 также настроен VRRP.  
Здесь для балансировки по VLAN на каждом виртуальном линке поднято по одному сабинтерфейсу.  
Таким образом трафик каждого VLAN будет проходить по своему одному виртуальному линку:по e0/0.10 будет ходить трафик VLAN 10, а по e0/1.20 трафик VLAN 20.


#### R12
 ```
 interface Ethernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.0.252 255.255.255.0
 vrrp 100 ip 192.168.0.254
 vrrp 100 priority 120

interface Ethernet0/1.20
 encapsulation dot1Q 20
 ip address 192.168.1.252 255.255.255.0
 no vrrp 100 preempt
 vrrp 200 ip 192.168.1.254
 vrrp 200 priority 99
 ```
 
#### R13
``` 
interface Ethernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.0.253 255.255.255.0
 vrrp 100 ip 192.168.0.254
 vrrp 100 priority 99

interface Ethernet0/1.20
 encapsulation dot1Q 20
 ip address 192.168.1.253 255.255.255.0
 vrrp 200 ip 192.168.1.254
 vrrp 200 priority 120
``` 


#### Таблица адресации для офиса Москва
|     Хост     |  Адрес                 | Интерфейс         |
|--------------|------------------------|-------------------| 
|VPC1          |192.168.0.10/24         | vlan 10           |
|VPC7          |192.168.1.20/24         | vlan 20           |
|SW4+SW5 VRRP  |192.168.0.1/24          | vlan 10            |
|SW4+SW5 VRRP  |192.168.1.1/24          | vlan 20            |
|SW3           |172.16.0.3/24          | vlan 666           |
|SW2           |172.16.0.2/24          | vlan 666           |
|SW4           |172.16.0.4/24          | vlan 666           |
|SW5           |172.16.0.5/24          | vlan 666           |
|R12+R13 VRRP  |192.168.0.254/24          | e0/0.10            |
|R12+R13 VRRP  |192.168.1.254/24          | e0/1.20            |
|R12           |192.168.0.252/24          | e0/0.10            |
|R12           |192.168.1.252/24          | e0/1.20            |
|R12           |10.0.0.1/29          |e0/2             |
|R12           |10.0.1.1/29          |e0/3             |
|R12           |12.12.12.12/32          |  loopback 0           |
|R13           |192.168.0.253/24          | e0/0.10            |
|R13           |192.168.1.253/24          | e0/1.20            |
|R13           |10.0.0.3/29          |e0/2             |
|R13           | 10.0.1.3/29         |e0/3             |
|R13           |13.13.13.13/32          |   loopback 0          |
|R14           |10.0.0.2/29          |e0/0             |
|R14           |14.14.14.14/32          |loopback 0             |
|R14           |10.0.1.2/29        |e0/1             |
|R14           |10.0.2.2/30        |e0/3             |
|R14           |11.0.0.2/30        |e0/2             |
|R15           |10.0.0.4/29          |e0/0             |
|R15           |15.15.15.15/32          |loopback 0             |
|R15           |10.0.1.4/29        |e0/1             |
|R15           |10.0.3.2/30        |e0/3             |
|R15           |12.0.0.2/30        |e0/2             |

## Санкт-Петербург 

На свитчах SW9, SW10 настроены VLAN 8 и 9 соответственно. Порты e0/0 и e0/1 на обоих коммутаторах агрегированы и на них настроены транки.
```
interface Port-channel1
 switchport trunk allowed vlan 8,9
 switchport trunk encapsulation dot1q
 switchport mode trunk
```
Транки также настроены на интерфейсах e0/3 и e1/0.

На роутерах R16 и R17 настроен VRRP с возможностью балансировки трафика по VLAN (на каждом роутере созданы 2 VRRP группы с противоположными приоритетами).

#### R17
```
interface GigabitEthernet0/0.8
 encapsulation dot1Q 8
 ip address 192.168.8.3 255.255.255.0
 vrrp 100 ip 192.168.8.1
 vrrp 100 priority 120
!
interface GigabitEthernet0/2.9
 encapsulation dot1Q 9
 ip address 192.168.9.3 255.255.255.0
 vrrp 200 ip 192.168.9.1
 vrrp 200 priority 99
```
#### R16
```
interface GigabitEthernet0/0.9
 encapsulation dot1Q 9
 ip address 192.168.9.2 255.255.255.0
 vrrp 200 ip 192.168.9.1
 vrrp 200 priority 120
!
interface GigabitEthernet0/2.8
 encapsulation dot1Q 8
 ip address 192.168.8.2 255.255.255.0
 vrrp 100 ip 192.168.8.1
 vrrp 100 priority 99
```
Для межвлановой маршрутизации на обоих роутерах выполнена команда ip routing.  


#### Таблица адресации для офиса СПБ
|     Хост     |  Адрес                 | Интерфейс         |
|--------------|------------------------|-------------------| 
|VPC8          |192.168.8.8/24         | vlan 8           |
|VPC7          |192.168.9.9/24         | vlan 9           |
|SW9           |192.168.8.254.24          | vlan 8           |
|SW10           |192.168.9.254/24         | vlan 9           |
|R17           |17.17.17.17/32          | loopback 0           |
|R17           |192.168.8.3/24          | e0/0           |
|R17           |192.168.9.3/24          | e0/2           |
|R17           |14.0.2.2/30          | e0/1           |
|R16           |16.16.16.16/32          | loopback 0           |
|R16           |192.168.9.2/24          | e0/0           |
|R16           |192.168.8.2/24          | e0/2           |
|R16           |14.0.3.2/30          | e0/1           |
|R16           |14.0.6.2/30          | e0/3           |
|R16 + R17           |192.168.8.1/24          | vrrp 100           |
|R16 + R17           |192.168.9.1/24          | vrrp 200           |
|R32           |32.32.32.32/32          | loopback 0           |
|R32           |14.0.6.1/30          | e0/0           |
|R18           |18.18.18.18/32          | loopback 0           |
|R18           |14.0.2.1/30          | e0/1           |
|R18           |14.0.3.1/30          | e0/0           |
|R18           |14.0.4.1/30          | e0/2           |
|R18           |14.0.5.1/30          | e0/3           

## Чокурдах
Классический "Роутер-на-палочке". На SW29 настроены VLAN 30,31. На роутере настроены соответствующие сабинтерфейсы, межвлановая маршрутизация.

#### Таблица адресации для офиса Чокурдах
|     Хост     |  Адрес                 | Интерфейс         |
|--------------|------------------------|-------------------| 
|R28          |28.28.28.28/32         | loopback 0           |
|R28          |192.168.30.1/24         | e0/2.30           |
|R28          |192.168.31.1/24         | e0/2.31           |
|R28          |15.0.0.1/30         | e0/0           |
|R28          |15.0.1.1/30         | e0/1           |
|VPC30          |192.168.30.30/24         | vlan 30           |
|VPC31          |192.168.31.31/24         | vlan 31           |

#### Таблица адресации для офиса Лабытнаги
|     Хост     |  Адрес                 | Интерфейс         |
|--------------|------------------------|-------------------| 
|R27          |27.27.27.27/32         | loopback 0           |
|R27          |16.0.0.1/30         | vlan 9           |

#### Таблица адресации для офиса Киторн
|     Хост     |  Адрес                 | Интерфейс         |
|--------------|------------------------|-------------------| 
|R22          |22.22.22.22/32         | loopback 0           |
|R22          |11.0.0.1/30         | e0/0           |
|R22          |11.0.1.1/30         | e0/1           |
|R22          |11.0.2.1/30         | e0/2           |

#### Таблица адресации для офиса Ламас
|     Хост     |  Адрес                 | Интерфейс         |
|--------------|------------------------|-------------------| 
|R21          |21.21.21.21/32         | loopback 0           |
|R21          |12.0.0.1/30         | e0/0           |
|R21          |11.0.1.2/30         | e0/1           |
|R21          |12.0.1.1/30         | e0/2           |

#### Таблица адресации для офиса Триада
|     Хост     |  Адрес                 | Интерфейс         |
|--------------|------------------------|-------------------| 
|R23          |23.23.23.23/32         | loopback 0           |
|R23          |11.0.2.2/30         | e0/0           |
|R23          |13.0.0.1/30         | e0/1           |
|R23          |13.0.1.1/30         | e0/2           |
|R24          |24.24.24.24/32         | loopback 0           |
|R24          |13.0.1.2/30         | e0/2           |
|R24          |12.0.1.2/30         | e0/0           |
|R24          |13.0.3.2/30         | e0/1           |
|R24          |14.0.4.2/30         | e0/3           |
|R25          |25.25.25.25/32         | loopback 0           |
|R25          |13.0.0.2/30         | e0/0           |
|R25          |13.0.2.2/30         | e0/2           |
|R25          |16.0.0.2/30         | e0/1           |
|R26          |26.26.26.26/32         | loopback 0           |
|R26          |13.0.2.1/30         | e0/2           |
|R26          |13.0.3.1/30         | e0/0           |




