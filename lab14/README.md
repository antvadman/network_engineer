# Lab 14 TLS Сертификаты


Задачи лабораторной работы:  

1. Настроите GRE поверх IPSec между офисами Москва и С.-Петербург.
2. Настроите DMVPN поверх IPSec между Москва и Чокурдах, Лабытнанги.
Дополнительно: Для IPSec использовать CA и сертификаты.

### 1 Настроите GRE поверх IPSec между офисами Москва и С.-Петербург.

Чотбы пакеты, предназначенные для шифрования, попадали на туннельный интерфейс и не натировались, необходимо исключить их из ACL натирования. Пакеты, идущие в другие сети, натируются.
```
#MSK
access-list 111 deny   icmp 192.168.0.0 0.0.1.255 192.168.8.0 0.0.1.255
access-list 111 permit ip 192.168.0.0 0.0.1.255 any

#SPB
access-list 111 deny   icmp 192.168.8.0 0.0.1.255 192.168.0.0 0.0.1.255
access-list 111 permit ip 192.168.0.0 0.0.1.255 any
```
Тк мы используем сертификаты для аутентификации, то необходимо на каком-нибудь из роутеров развернуть СА.
В лабе будем использовать для этого R21.
Задаем имя домена, включаем доступ по http для скачивания сертов и генерируем самоподписной сертификат СА.
Ниже приведен фрагмент сертификата.
```
ip domain-name otus.ru
ip http server
crypto key generate rsa general-keys label CA exportable modulus 2048
crypto pki server CA

crypto pki certificate chain CA
 certificate ca 01
  308202F8 308201E0 A0030201 02020101 300D0609 2A864886 F70D0101 04050030
  0D310B30 09060355 04031302 4341301E 170D3234 30323132 31313532 31345A17
  0D323730 32313131 31353231 345A300D 310B3009 06035504 03130243 41308201
 ...
 ...
        quit

```

Далее на клиентах R15 R21 мы также должны сгенерировать ключевую пару, на ее основе создать серт и подписать его ключем СА. 
Для этого мы настраиваем доверенный центр (он же СА) и направляем запрос на сертификат СА и подпись своего сертификата. Ниже преведен конфиг R15, для остальных роутеров аналогично.
```
crypto pki trustpoint VPN
 enrollment url http://12.0.0.1:80
 ip-address 12.0.0.2
 subject-name CN=R15,OU=VPN,O=Otus,C=RU
 revocation-check none
 rsakeypair VPN
```
На R21 мы видим запрос:
```
R21#sho crypto pki server CA requests
Enrollment Request Database:

Subordinate CA certificate requests:
ReqID  State      Fingerprint                      SubjectName
--------------------------------------------------------------

RA certificate requests:
ReqID  State      Fingerprint                      SubjectName
--------------------------------------------------------------

Router certificates requests:
ReqID  State      Fingerprint                      SubjectName
--------------------------------------------------------------
1      pending    0BBA155CDAD3831F23171F3991D1C596 hostname=R15+ipaddress=12.0.0.2,cn=R15,ou=VPN,o=Otus,c=RU
```
Подписываем сертификат и отправляем его на R15. Ntgthm e R15 два сертификата: подписанный свой и серт СА.
```
R15#sho crypto pki certificates
Certificate
  Status: Available
  Certificate Serial Number (hex): 04
  Certificate Usage: General Purpose
  Issuer:
    cn=CA
  Subject:
    Name: R15
    IP Address: 12.0.0.2
    hostname=R15+ipaddress=12.0.0.2
    cn=R15
    ou=VPN
    o=Otus
    c=RU
  Validity Date:
    start date: 19:41:47 UTC Feb 17 2024
    end   date: 19:41:47 UTC Feb 16 2025
  Associated Trustpoints: VPN

CA Certificate
  Status: Available
  Certificate Serial Number (hex): 01
  Certificate Usage: Signature
  Issuer:
    cn=CA
  Subject:
    cn=CA
  Validity Date:
    start date: 14:52:14 UTC Feb 12 2024
    end   date: 14:52:14 UTC Feb 11 2027
  Associated Trustpoints: VPN


R18#sho crypto pki certificates
Certificate
  Status: Available
  Certificate Serial Number (hex): 05
  Certificate Usage: General Purpose
  Issuer:
    cn=CA
  Subject:
    Name: R18
    IP Address: 14.0.4.1
    hostname=R18+ipaddress=14.0.4.1
    cn=R18
    ou=VPN
    o=Otus
    c=RU
  Validity Date:
    start date: 16:48:02 UTC Feb 17 2024
    end   date: 16:48:02 UTC Feb 16 2025
  Associated Trustpoints: VPN

CA Certificate
  Status: Available
  Certificate Serial Number (hex): 01
  Certificate Usage: Signature
  Issuer:
    cn=CA
  Subject:
    cn=CA
  Validity Date:
    start date: 11:52:14 UTC Feb 12 2024
    end   date: 11:52:14 UTC Feb 11 2027
  Associated Trustpoints: VPN
```
Теперь настраиваем туннель IPSec:
```
crypto isakmp policy 10
 encr aes 256
 group 2

crypto ipsec transform-set GRE-TS esp-aes 256 esp-sha-hmac
 mode transport

crypto ipsec profile GRE-PROF
 set transform-set GRE-TS
```
Применяем профиль ipsec на туннельном интерфейсе:
```
interface Tunnel0
 ip address 10.10.10.15 255.255.255.0
 ip tcp adjust-mss 1360
 tunnel source 12.0.0.2
 tunnel destination 14.0.4.1
 tunnel protection ipsec profile GRE-PROF
```
Пингуем из сети МСК станции сети в СПБ:
```
VPCS> ping 192.168.8.11
84 bytes from 192.168.8.11 icmp_seq=1 ttl=59 time=2.939 ms
```
На дампе трафика содержание пакетов зашифровано, протокол инкапсуляции ESP.
<image src="1.png">

### 2 Настроите DMVPN поверх IPSec между Москва и Чокурдах, Лабытнанги.

Аналогично 1 пункту, на роутерах генерируем ключевые пары, создаем запросы на серты.
Подписываем серты в СА и отправляем их вместе с самопдписным сертом СА.
Единственная разница - мы применяем профиль IPSec на туннельном интерфейсе DMVPN.
Пример R15:
```
interface Tunnel100
 ip address 100.100.100.15 255.255.255.0
 no ip redirects
 ip nhrp network-id 100
 tunnel source Ethernet0/2
 tunnel mode gre multipoint
 tunnel protection ipsec profile GRE-PROF
```
Для проверки пробуем пропинговть R28 с R27.
На скриншоте видно, что сначала строится канал с хабом, потом спока со споком.
<image src="2.png">
Далее обмен трафиком происходит спок-спок без хаба.
<image src="3.png">

[Лаба](./lab14.zip)
