# Lab 05 Policy based routig & IP SLA

### Топология сети
<image src="scheme.png">

Задачи лабораторной работы:  
Настроить политику маршрутизации в офисе Чокурдах  
Распределить трафик между 2 линками  

1. Настроите политику маршрутизации для сетей офиса.  
2. Распределите трафик между двумя линками с провайдером.  
3. Настроите отслеживание линка через технологию IP SLA.(только для IPv4)  
4. Настройте для офиса Лабытнанги маршрут по-умолчанию.

### PBR  

На R28 настроен ACL, который матчит тафик до лупбека R27 (27.27.27.27/32) и обратно.
```
ip access-list extended vlan30_acl
 permit icmp 192.168.30.0 0.0.0.255 host 27.27.27.27
 permit icmp host 27.27.27.27 192.168.30.0 0.0.0.255
```

Далее создаем route map с изменением адреса следующего хопа:
```
route-map vlan30_map permit 10
 match ip address vlan30_acl
 set ip next-hop 15.0.0.2
```
И привязываем route map к интерфейсу:
```
interface Ethernet0/2.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
 ip policy route-map vlan30_map
```
Теперь трафик из vlan 30 перенаправляется на R26, а потом на R25.
На R26 прописаны следующие маршруты:
```
ip route 27.27.27.27 255.255.255.255 13.0.2.2
ip route 192.168.30.0 255.255.255.0 15.0.0.1
ip route 192.168.31.0 255.255.255.0 15.0.0.1
```

На R25, в свою очередь, также создан ACL и route-map:
```
ip access-list extended vlan30_acl
 permit icmp 192.168.30.0 0.0.0.255 host 27.27.27.27
 permit icmp host 27.27.27.27 192.168.30.0 0.0.0.255

route-map vlan30_map permit 10
 match ip address vlan30_acl
 set ip next-hop 13.0.2.1

interface Ethernet0/1
 ip address 16.0.0.2 255.255.255.252
 ip policy route-map vlan30_map
```

А также прописан маршрут до лупбека R27 (27.27.27.27/32):
```
ip route 27.27.27.27 255.255.255.255 16.0.0.1
```

На R27 прописан маршрут по умолчанию:
```
ip route 0.0.0.0 0.0.0.0 16.0.0.2
```
Если с VPC30 (те VLAN30) запустить пинг до 27.27.27.27, то пинг пройдет:
```
VPCS> ping 27.27.27.27 -c 1000
84 bytes from 27.27.27.27 icmp_seq=1 ttl=252 time=3.312 ms
84 bytes from 27.27.27.27 icmp_seq=2 ttl=252 time=3.038 ms
```

Но трафик этого влана будет проходить через интерфейс e0/2 роутера R25 (см скриншот ниже):

<image src="dump.jpg">

Трафик VLAN31 будет проходить через интерфейс e0/3 роутера R25.

### IP SLA 
Пусть основной линк будет e0/1 R28 <-> e0/3 R25, его доступность будем отслеживать при помощи IP SLA.  
Резервный линк e0/0 R28 <-> e0/1 R26, на него будет перенаправляться трафик в случае недоступности основного линка.

На роутере R28 создаем sla, настраиваем его:  
```
ip sla 1
 icmp-echo 15.0.1.2 source-ip 15.0.1.1
 frequency 5
ip sla schedule 1 life forever start-time now
```
Далее настраиваем отслеживание доступности линка:  
```
track 1 ip sla 1 reachability
 delay down 10 up 10
```

Создаем два маршрута до лупбека R27 с разными метриками, на основной маршрут вешаем трекинг. Данный маршрут будет удален из таблицы маршрутизации в случае недоступности основного линка:
```
ip route 27.27.27.27 255.255.255.255 15.0.1.2 50 track 1
ip route 27.27.27.27 255.255.255.255 15.0.0.2 100
```

На роутере R25 настраиваем то же самое, только резервным next-hop'ом будет выступать интерфейс e0/2 R26.  
Переключение на резервный линк:  
```
84 bytes from 27.27.27.27 icmp_seq=41 ttl=253 time=2.581 ms
84 bytes from 27.27.27.27 icmp_seq=42 ttl=253 time=2.621 ms
27.27.27.27 icmp_seq=43 timeout
27.27.27.27 icmp_seq=44 timeout
27.27.27.27 icmp_seq=45 timeout
27.27.27.27 icmp_seq=46 timeout
27.27.27.27 icmp_seq=47 timeout
27.27.27.27 icmp_seq=48 timeout
27.27.27.27 icmp_seq=49 timeout
27.27.27.27 icmp_seq=50 timeout
27.27.27.27 icmp_seq=51 timeout
27.27.27.27 icmp_seq=52 timeout
84 bytes from 27.27.27.27 icmp_seq=53 ttl=252 time=3.059 ms
84 bytes from 27.27.27.27 icmp_seq=54 ttl=252 time=3.933 ms
```
Статистика по SLA:
```
R28(config)#do show ip sla statistics
IPSLAs Latest Operation Statistics

IPSLA operation id: 1
        Latest RTT: 2 milliseconds
Latest operation start time: 13:33:12 UTC Mon Oct 23 2023
Latest operation return code: OK
Number of successes: 402
Number of failures: 19
Operation time to live: Forever
```








