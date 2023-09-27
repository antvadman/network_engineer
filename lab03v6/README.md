# Lab 03 DHCPv6

### Топология сети
<image src="scheme.png">

Задачи лабораторной работы:  
1. Построить сеть и произвести базовые конфигурации устройств  
2. Проверить получение ipc6 адреса SRV1 по SLAAC  
3. Настроить Stateless DHCPv6 на роутере R1  
4. Настроить Stateful DHCPv6 на роутере R1  
5. Настроить DHCPv6 Relay на роутере R2  

### Исходные данные:  
Свитч Cisco IOS - 2 Pcs  
Роутер Cisco IOS - 2 Pcs  
Windows Server - 2 Pcs  

### Таблица адресов  

|Device|Interface |IPv6 address|
|------|--------------|-----------|
|R1	   | Gi0/0 |2001:db8:acad:2::1/64  |
|R1	   | Gi0/0 |fe80::1  |
|R1	   | Gi0/0 |2001:db8:acad:1::1/64  |
|R1	   | Gi0/0 |fe80::1  |
|R2	   | Gi0/0 |2001:db8:acad:2::2/64  |
|R2	   | Gi0/0 |fe80::2  |
|R2	   | Gi0/0 |2001:db8:acad:3::1/64  |
|R2	   | Gi0/0 |fe80::1  |
|SRV1	   | NIC |DHCP  |
|SRV2	   | NIC |DHCP  |

### Базовая конфигурация на примере роутера R1

Назначить имя устройства: **hostname R1** (для других сетевых устройств аналогично)  
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
ipv6 route ::/0 2001:DB8:ACAD:2::2  
ipv6 route ::/0 2001:DB8:ACAD:2::1  
```
## Проверка корректности маршрутов  
Пинг с роутера R1 интерфейса Gi0/1 роутера R2:  
```
R1#ping 2001:db8:acad:3::1  
Type escape sequence to abort.  
Sending 5, 100-byte ICMP Echos to 2001:DB8:ACAD:3::1, timeout is 2 seconds:  
!!!!!  
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/4/7 ms  
```
## Проверка получения ipv6 адреса по SLAAC (SRV1)   

<image src="srv1_slaac.png">

## Настройка Stateless DHCP на R1  
```
R1(config)# ipv6 dhcp pool R1-STATELESS   
R1(config-dhcp)# dns-server 2001:db8:acad::254  
R1(config-dhcp)# domain-name STATELESS.com  
R1(config)# interface gi0/1  
R1(config-if)# ipv6 nd other-config-flag  
R1(config-if)# ipv6 dhcp server R1-STATELESS  
```

## Проверка получения ipv6 адреса по DHCP (stateless)  

<image src="stateless.png">  

## Настройка Stateful DHCP на R1  
```
R1(config)# ipv6 dhcp pool R2-STATEFUL  
R1(config-dhcp)# address prefix 2001:db8:acad:3:aaa::/80  
R1(config-dhcp)# dns-server 2001:db8:acad::254  
R1(config-dhcp)# domain-name STATEFUL.com  
R1(config)# interface gi0/0  
R1(config-if)# ipv6 dhcp server R2-STATEFUL  
```

## Проверка получения ipv6 адреса по SLAAC (SRV2)  

<image src="srv2_slaac.png">  

## Настройка DHCP Relay на R2 и проверка адреса на SRV2  
```
R2(config)# interface gi0/1  
R2(config-if)# ipv6 nd managed-config-flag  
R2(config-if)# ipv6 dhcp relay destination 2001:db8:acad:2::1 gi0/0  
```
<image src="srv2_stateful.png">  

## Полезные ссылки

[Файл лабораторной работы](./lab03v6.unl)  














