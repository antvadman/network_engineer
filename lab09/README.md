# Lab 09 BGP

### Топология сети
<image src="scheme.png">

Задачи лабораторной работы:  

1. Настроите eBGP между офисом Москва и двумя провайдерами - Киторн и Ламас.
2. Настроите eBGP между провайдерами Киторн и Ламас.
3. Настроите eBGP между Ламас и Триада.
4. Настроите eBGP между офисом С.-Петербург и провайдером Триада.
5. Организуете IP доступность между пограничным роутерами офисами Москва и С.-Петербург.

### 1.1 Москва - Киторн  
Настройка eBGP между R14 и R22  
(Для других пиров BGP настройки аналогичные. Будут приводиться только конфиги)
Запускаем процесс BGP, при этом указываем номер своей АС, настраиваем соседа.  
Для этого указываем адрес соседа и номер соседней АС.  
R14 (Здесь мы дополнительно анонсируем лупбэк 14.14.14.14/32)
```
router bgp 1001
 network 14.14.14.14 mask 255.255.255.255
 neighbor 11.0.0.1 remote-as 101
```
R22  
```
router bgp 101
 neighbor 11.0.0.2 remote-as 1001
 neighbor 11.0.1.2 remote-as 301
```

### 1.2 Москва - Ламас  
Настройка eBGP между R15 и R21
R15  
```
router bgp 1001
 network 15.15.15.15 mask 255.255.255.255
 neighbor 12.0.0.1 remote-as 301
```
R21  
```
router bgp 301
 neighbor 11.0.1.1 remote-as 101
 neighbor 12.0.0.2 remote-as 1001
 neighbor 12.0.1.2 remote-as 2222
```

### 2 Киторн - Ламас
eBGP между Киторн и Ламас (R21-R22 соответственно) настроен в предыдущем пункте.  
```
neighbor 11.0.1.1 remote-as 101
neighbor 11.0.1.2 remote-as 301
```

### 3 Ламас - Триада 
Настройка eBGP между R21 и R24
R21  
```
router bgp 301
 neighbor 11.0.1.1 remote-as 101
 neighbor 12.0.0.2 remote-as 1001
 neighbor 12.0.1.2 remote-as 2222
```
R24
```
router bgp 2222
 bgp log-neighbor-changes
 neighbor 12.0.1.1 remote-as 301
 neighbor 13.0.3.1 remote-as 2222
 neighbor 14.0.4.1 remote-as 2042
```

### 4 С. Петербург - Триада
R18 поднимает соседство с R24, R26 и анонсирует свой лупбэк (18.18.18.18/32).  
R18
```
router bgp 2042
 network 18.18.18.18 mask 255.255.255.255
 neighbor 14.0.4.2 remote-as 2222
 neighbor 14.0.5.2 remote-as 2222
```
R24
```
router bgp 2222
 network 24.24.24.24 mask 255.255.255.255
 neighbor 12.0.1.1 remote-as 301
 neighbor 13.0.3.1 remote-as 2222
 neighbor 14.0.4.1 remote-as 2042
```
R26
```
router bgp 2222
 neighbor 13.0.3.2 remote-as 2222
 neighbor 14.0.5.1 remote-as 2042
```

### 5 С. Петербург - Москва
С R18 пропингуем лупбэки московских роутеров R14, R15. Источкиком при этом укажем лупбэк R18:
```
R18#ping 15.15.15.15 source 18.18.18.18
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 15.15.15.15, timeout is 2 seconds:
Packet sent with a source address of 18.18.18.18
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
R18#ping 14.14.14.14 source 18.18.18.18
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 14.14.14.14, timeout is 2 seconds:
Packet sent with a source address of 18.18.18.18
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/3 ms
```


















