# Lab 01 Intervlan Routing

### Топология сети
<image src="scheme.png">

Задачи лабораторной работы:  
1. Провести базовую конфигурацию оборудования  
2. Создать VLANы и настроить порты свитчей  
3. Настроить транк между свитчами  
4. Настроить на роутере межвлановую маршрутизацию  
5. Проверить работу межвлановой маршрутизации  

### Исходные данные:  
Свитч Cisco IOS - 2 Pcs  
Роутер Cisco IOS - 1 Pc  
VPC - 2 Pcs  

Таблица адресов  

|Device|	Interface |	IP Address   |	Subnet  Mask |	Default Gateway |
|------|--------------|-----------   |--------       |------------------|
|R1	   | Gi0/1.3   |	192.168.3.1 | 255.255.255.0	| N/A |
|R1	    |Gi0/1.4   |	192.168.4.1 |	255.255.255.0 |	N/A|
|R1	 |Gi0/1.8 |	N/A|	N/A |	N/A |
|S1	|VLAN 3|	192.168.3.11|	255.255.255.0|	192.168.3.1|
|S2|	VLAN 3 |	192.168.3.12|	255.255.255.0|	192.168.3.1|
|PC-A|	NIC	|192.168.3.3|	255.255.255.0|	192.168.3.1|
|PC-B|	NIC|	192.168.4.3|	255.255.255.0|	192.168.4.1|

Таблица VLAN  

|VLAN|	Name|	Interface Assigned|
|----|------|--------------------|
|3	|MGMT|	S1 VLAN 3
|3	|MGMT|	S2 VLAN 3
|4|	Operations|	S2 Gi0/2
|7|	ParkingLot	|Оставшиеся неиспользуемые порты S1 S2 
|8|	Native|	N/A

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
  
На другом свитче и роутере бзовые настройки аналогичны.    
Сетевые настройки на PC-A: **ip 192.168.3.3/24 192.168.3.1**  
Сетевые настройки на PC-B: **ip 192.168.4.3/24 192.168.4.1**

## Создание и настройка vlan  

Создаем vlanы на S1   

vlan 3    
name MGMT     

vlan 4  
name Operations   

vlan 7  
name ParkingLot  

vlan 8  
name NATIVE  

Для свитча S2 vlanы создаются аналогичными командами  

Настраиваем access port на S1  

int Gi0/2  
switchport mode access  
switchport access vlan 3   

Настраиваем access port на S2  

int Gi0/2  
switchport mode access  
switchport access vlan 4  

## Настройка trunk на S1  
int Gi0/0  
switchport trunk encapsulation dot1q  
switchport mode trunk  
switchport trunk allowed vlan 3,4,8  
switchport trunk native vlan 8  

## Настройка trunk на S2  
int Gi0/0  
switchport trunk encapsulation dot1q  
switchport mode trunk  
switchport trunk allowed vlan 3,4,8  
switchport trunk native vlan 8  

int Gi0/0  
switchport trunk encapsulation dot1q  
switchport mode trunk  
switchport trunk allowed vlan 3,4,8  
switchport trunk native vlan 8  

## Настройка адресации на vlan  
interface vlan 3  
ip address 192.168.3.11 255.255.255.0 (для S1)  
ip address 192.168.3.12 255.255.255.0 (для S2)  

## Настройки вланов для S1  

<image src="S1_vlans.png">

## Настройки вланов для S2  

<image src="S1_vlans.png">  

## Настройка sub-интерфейсов роутера  

int Gi0/1.3  
encapsulation dot1Q 3  
ip address 192.168.3.1 255.255.255.0  

int Gi0/1.4  
encapsulation dot1Q 4  
ip address 192.168.4.1 255.255.255.0  

Маршрутизация включается командой: **ip routing**  

## Проверка работы маршрутизации

Для проверки работы межвлановой маршрутизации с РС-А, находящегося во vlan 3, посылалась команда **ping** до РС-В,  
находящегося во vlan 4. Пинг проходит, результаты представлены ниже:  

<image src="ping.png">  

## Полезные ссылки

[Конфигурация свитча S1](./S1.md)  

[Конфигурация свитча S2](./S2.md)

[Конфигурация роутера R1](./R1.md)

[Файл лабораторной работы](./lab01.uml)  














