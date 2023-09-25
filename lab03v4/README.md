# Lab 03 DHCPv4

### Топология сети
<image src="scheme.png">

Задачи лабораторной работы:  
1. Построить сеть и произвести базовые конфигурации устройств  
2. Настроить два DHCPv4 сервера на роутере R1  
3. Настроить DHCPv4 Relay на роутере R2   

### Исходные данные:  
Свитч Cisco IOS - 2 Pcs  
Роутер Cisco IOS - 2 Pcs  
Virtual PC - 2 Pcs  

### Таблица VLAN  

|VLAN|	Name |Interface assigned	   |
|------|--------------|-----------   |
|1	   | NA |	S2:Gi0/0  |
|100	    |Clients   |S1:Gi0/0 |
|200	 |MGMT |S1: VLAN200 |
|999	 |ParkingLot |	All switched off unused interfaces |
|1000	 |Native |NA |

### Таблица адресов  

|Device|Interface |IPv4 address|Subnet mask|Default gateway|
|------|--------------|-----------|---------|--------|
|R1	   | Gi0/0 |10.0.0.1  |255.255.255.252|NA|
|R1	    |Gi0/1   |NA|NA|NA|
|R1	 |Gi0/1.100 |192.168.1.0|255.255.255.192|NA|
|R1	 |Gi0/1.200 |192.168.1.64|255.255.255.224|NA|
|R1	 |Gi0/1.1000 |NA|NA|NA|
|R2	   | Gi0/0 |10.0.0.2  |255.255.255.252|NA|
|R2	 |Gi0/1 |192.168.1.97|255.255.255.240|NA|
|S1	 |Vlan200 |192.168.1.66|255.255.255.224|192.168.1.65|
|S2	 |Vlan1 |192.168.1.98|255.255.255.240|192.168.1.97|
|VPCA	 |NIC |DHCPv4|DHCPv4|DHCPv4|
|VPCB	 |NIC |DHCPv4|DHCPv4|DHCPv4|

### Базовая конфигурация на примере свитча S1

Назначить имя устройства: **hostname S1** (для других сетевых устройств аналогично)  
Отключить DNS lookup: **no ip domain-lookup** (для других сетевых устройств аналогично)  
Назначить cisco логином и паролем для настроек: **username cisco privilege 15 password cisco**  
Назначить cisco логином и паролем для терминала: 
**line vty 0 4**  
**login local**  
Назначить cisco логином и паролем для консоли:  
**line con 0**  
**login local**  
Установить корректное время: **clock timezone PST 3 0**  
Шифровать пароли: **service password-encryption**   
Создать предупреждающий баннер:   
**banner motd**  
**For authorized persons only**  
Назначить class паролем на enable: **enable secret class**  
Настроить logging synchronous для консольного канала:
**line con 0**
**logging synch**
**line vty 0 4**
**logging synch**

## Настройка статических маршрутов на роутерах (R1 и R2 соответственно)  
```
ip route 0.0.0.0 0.0.0.0 10.0.0.2  
ip route 0.0.0.0 0.0.0.0 10.0.0.1  
```
## Проверка корректности маршрутов
Пинг с роутера R1 интерфейса Gi0/1 роутера R2:  
```
R1#ping 192.168.1.97
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.97, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 5/6/12 ms
```

## Настройка VLAN  
S1  
```
interface Vlan200  
ip address 192.168.1.66 255.255.255.224  
ip default-gateway 192.168.1.65  
```
S2
```
interface Vlan1  
ip address 192.168.1.98 255.255.255.240  
ip default-gateway 192.168.1.97  
```
## Настройка trunk  
S1  
```
interface GigabitEthernet0/1  
 switchport trunk allowed vlan 100,200,1000  
 switchport trunk encapsulation dot1q  
 switchport trunk native vlan 1000  
 switchport mode trunk  
```
Интерфейс Gi0/1 находится во VLAN 1 до настройки транка, тк все порты по умолчанию находятся в этом влане.  

Если в данный момент оба VPC были бы подключены к DHCP и находились во 100 влане, то получили бы адреса 192.168.1.1 и 192.168.1.2.  

## Настройка DHCP пулов на R1  
```
ip dhcp excluded-address 192.168.1.1 192.168.1.5  
ip dhcp excluded-address 192.168.1.65 192.168.1.69  
ip dhcp pool R1_clients_pool  
 network 192.168.1.0 255.255.255.192  
 domain-name ccna-lab.com  
 default-router 192.168.1.1  
 lease 2 12 30  

ip dhcp pool R1_MGMT_pool  
 network 192.168.1.64 255.255.255.224  
 domain-name ccna-lab.com  
 default-router 192.168.1.65  
 lease 2 12 30  
```

## Проверка DHCP пулов на R1  
```
R1#show ip dhcp pool
Pool R1_clients_pool :  
 Utilization mark (high/low)    : 100 / 0  
 Subnet size (first/next)       : 0 / 0  
 Total addresses                : 62  
 Leased addresses               : 1  
 Pending event                  : none  
 1 subnet is currently in the pool :  
 Current index        IP address range                    Leased addresses  
 192.168.1.6          192.168.1.1      - 192.168.1.62      1  
R1#show ip dhcp binding  
Bindings from all pools not associated with VRF:  
IP address          Client-ID/              Lease expiration        Type  
                    Hardware address/  
                    User name  
192.168.1.6         0100.5079.6668.05       Sep 27 2023 07:31 AM    Automatic  
```
## Получение адреса на VPCA  
```
VPCS> ip dhcp  
DORA IP 192.168.1.6/26 GW 192.168.1.1  
```

## Настройка DHCP Relay на R2  
Сначала создадим пул адресов на R1, которые будет получать VPCB.  
```
ip dhcp excluded-address 192.168.1.97 192.168.1.101  
ip dhcp pool R2_client_lan  
 network 192.168.1.96 255.255.255.240  
 domain-name ccna-lab.com  
 default-router 192.168.1.97  
 lease 2 12 30  
```
Настроим DHCP Relay на Gi0/1 R2  
```
interface GigabitEthernet0/1  
 ip address 192.168.1.97 255.255.255.240  
 ip helper-address 10.0.0.1  
```
Проверим выдачу R1 адресов для VPCB:  
```
R1#show ip dhcp binding  
Bindings from all pools not associated with VRF:  
IP address          Client-ID/              Lease expiration        Type  
                    Hardware address/  
                    User name  
192.168.1.6         0100.5079.6668.05       Sep 28 2023 08:41 AM    Automatic  
192.168.1.102       0100.5079.6668.06       Sep 28 2023 08:41 AM    Automatic  
```

## Получение адреса на VPCB  
```
VPCS> ip dhcp  
DORA IP 192.168.1.102/28 GW 192.168.1.97  
```
## Полезные ссылки

[Файл лабораторной работы](./lab03v4.unl)  














