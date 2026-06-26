
![[cache/Pasted image 20260626154645.png]]

ISP 
```
hostnamectl hostname isp.au-team.irpo
```

HQ-RTR 
```
hostnamectl hostname hq-rtr.au-team.irpo
```

HQ-SRV 
```
hostnamectl hostname hq-srv.au-team.irpo
```

BR-RTR 
```
hostnamectl hostname br-rtr.au-team.irpo
```

BR-SRV 
```
hostnamectl hostname br-srv.au-team.irpo
```

HQ-CLI 
```
hostnamectl hostname hq-cli.au-team.irpo
```

ISP  
ens34: 172.16.1.1 /28 
ens35: 172.16.2.1 /28   

HQ-RTR 
ens33: 
	172.16.1.2/28
	172.16.1.1
     8.8.8.8
ens34.vlan100: 192.168.100.1/27
ens34.vlan200: 192.168.200.1/27 
ens34.vlan999: 192.168.99.1/29 

HQ-SRV  
ens33.vlan100:
	192.168.100.10/27
	192.168.100.1

BR-RTR 
ens33:
	172.16.2.2/28
	172.16.2.1
	 8.8.8.8
ens34: 172.30.100.1/27

BR-SRV 
ens33:
	172.30.100.10/27 
	172.30.100.1

HQ-CLI
ens33 ничо не делаем
ens33.200 просто создаем его, его именно выключаем и включаем когда dhcp будет приходить

DHCP // для начала надо чтобы у HQ_RTR был интернет, чтобы скачать dhcp пакет. HQ-RTR dhcp это сервер

на них троих ISP/BR-RTR/HQ-RTR пишем
```
echo “net.ipv4.ip_forward=1” >> /etc/sysctl.conf
sysctl -p
```

### Установка и запуск iptables-services НА ISP 

```
dnf install iptables-services -y 
systemctl enable --now iptables 
```
### Очистка правил

```
iptables -F 
```
### Разрешить форвардинг для обеих сетей (HQ и BR)__________

```
iptables -A FORWARD -s 172.16.0.0/16 -j ACCEPT 
iptables -A FORWARD -d 172.16.0.0/16 -j ACCEPT 
```
### NAT (маскарадинг) для обеих сетей через внешний интерфейс ens33

```
iptables -t nat -A POSTROUTING -o ens33 -s 172.16.0.0/16 -j MASQUERADE 
```

  
## Отключение firewalld

```
systemctl stop firewalld 
systemctl disable firewalld 
```

### Сохранение правил

```
iptables-save > /etc/sysconfig/iptables 
```

ПРОВЕРЯЕМ ПИНГИ НА 8.8.8.8 С HQ-RTR и BR-RTR
  
Настройте протокол динамической конфигурации хостов для сети в сторону HQ-CLI

HQ-RTR
```
dnf install dhcp-server
nano /etc/dhcp/dhcpd.conf
subnet 192.168.200.0 netmask 255.255.255.224 { 
	range 192.168.200.2 192.168.200.30; 
	option routers 192.168.200.1; 
	option broadcast-address 192.168.200.31; 
	option domain-name-servers 192.168.100.10; 
	option domain-name “au-team.irpo”;
}
systemctl enable --now dhcpd
dhcpd
```


Получаем адрес на HQ-CLI путём отключения и включения интерфейса ens33.vlan200. ПРОВЕРЯЕМ НА HQ-RTR, ЧТО ЕСТЬ ЗАПИСЬ В ФАЙЛЕ, УКАЗЫВАЮЩАЯ 
НА ПОЛУЧЕНИЕ АДРЕС КЛИЕНТОМ: 

```
cat /var/lib/dhcpd/dhcpd.leases
```


---

## SSH

HQ-SRV и BR-SRV

```
useradd -m -U -s /bin/bash -u 2026 sshuser 
passwd sshuser 
P@ssw0rd 
P@ssw0rd 
echo “sshuser ALL=(ALL) NOPASSWD: ALL” >> /etc/sudoers
```


Создаём баннер 
```
echo “Authorized access only” > /etc/ssh/banner.txt
```

Настраиваем SSH 
```
nano /etc/ssh/sshd_config 

Port 2026 
AllowUsers sshuser 
MaxAuthTries 2 
Banner /etc/ssh/banner.txt 

```

Сохраняем и выходим

Разрешаем подключение по порту 2026
```
semanage port -a -t ssh_port_t -p tcp 2026 
```

Перезапускаем ssh 
```
systemctl restart sshd 
```

(ЕСЛИ SSH не работает , # Проверить весь файл на ошибки
```
sshd -t -f /etc/ssh/sshd_config
```

### Проверка подключения по SSH

BR-RTR
```
ssh -l sshuser 172.30.100.10 -p 2026
```

HQ-RTR
```
ssh -l sshuser 192.168.100.10 -p 2026
```

---

## СОЗДАЕМ ПРОСТО ТАК ПОЛЬЗОВАТЕЛЕЙ БЕЗ НАСТРОЙКИ SSH НА МАРШУТИЗАТОРАХ. Я ВООБЩЕ ХЗ ЗАЧЕМ НО ПО ЗАДАНИЮ ТАК НАДО

HQ-RTR и BR-RTR
```
useradd -m -U -s /bin/bash net_admin 
passwd net_admin 
P@ssw0rd 
P@ssw0rd 
echo “net_admin ALL=(ALL) NOPASSWD: ALL” >> /etc/sudoers  
```




## Туннель м-у сетями HQ и BR на маршах HQ-RTR и BR-RTR

HQ-RTR
```
nmtui> изменить подключение> добавить >ip-туннель
Имя профиля: tun0
Устройство: tun0
Режим: gre 
Родительский6 ens33
Локальный: 172.16.1.2
Удаленный: 172.16.2.2
Конфигурация: ipv4 вручную 
Адреса: 10.10.10.1/30
```
И перезапускаем tunnel через nmtui (выключаем и включаем 
интерфейс) 

без этой херни тоже работать не будет
```
nmcli connection modify tun0 ip-tunnel.ttl 64
```


BR-RTR
```
nmtui> изменить подключение> добавить >ip-туннель
Имя профиля: tun0
Устройство: tun0
Режим: gre 
Родительский6 ens33
Локальный: 172.16.2.2
Удаленный: 172.16.1.2
Конфигурация: ipv4 вручную 
Адреса: 10.10.10.2/30
```
И перезапускаем tunnel через nmtui (выключаем и включаем 
интерфейс) 

без этой херни тоже работать не будет
```
nmcli connection modify tun0 ip-tunnel.ttl 64
```
Проверяем пинги с двух роутеров на 10.10.10.1 и 10.10.10.2



## Обеспечьте динамическую маршрутизацию на маршрутизаторах HQ-RTR и BR-RTR:

HQ-RTR И BR-RTR

```
dnf install frr
systemctl enable --now frr
nano /etc/frr/daemons // заменить no на yes в ospfd=yes
```


Перезапустить frr
```
systemctl restart frr
vtysh
```

ДАЛЕЕ РАБОТА КАК В CISCO
```
conf t
router ospf
```

Команды для HQ-RTR 

```
conf t
router ospf
network 192.168.100.0/27 area 0 
network 192.168.200.0/27 area 0 
network 10.10.10.0/30 area 0
ospf router-id 172.16.1.2
 passive-interface ens33
 passive-interface ens34
 passive-interface ens35
 area 0 authentication
 exit
interface tun0
 ip ospf authentication
 ip ospf authentication-key P@ssw0rd
 do wr
 exit
exit
exit
```

Команды для BR-RTRy

```
router ospf
 network 172.30.100.0/27 area 0
 network 10.10.10.0/30 area 0
 ospf router-id 172.16.2.2
 passive-interface ens33
 passive-interface ens34
 area 0 authentication
 exit
interface tun0
 ip ospf authentication
 ip ospf authentication-key P@ssw0rd
 do wr
 exit
exit
exit
```

## Настройка динамической трансляции адресов на маршрутизаторах HQ-RTR и BR-RTR

HQ-RTR И BR-RTR:

1. Включить и запустить firewalld
```
systemctl --now enable firewalld 
```
 
 2. Установить зону trusted по умолчанию
```
firewall-cmd --set-default-zone=trusted 
```

3. Включить маскарадинг (NAT) в зоне trusted (постоянно)
```
firewall-cmd --zone=trusted --add-masquerade --permanent 
```

4. Перезапустить firewalld для применения permanent-правил
```
systemctl restart firewalld
```

Проверяем на двух устройствах sh ip ospf neighbor

после этого видимо между hq-srv и br-srv начнет пинговать
172.16.30.10 <-> 192.168.100.10


## Настройте инфраструктуру разрешения доменных имён для офисов HQ и BR:

HQ-SRV
```
dnf install bind
nano /etc/named.conf

в строчке  listen-on port 53{any; };
allow-query  {any; }; после этой строчки пишем 
forwarders {10.39.0.1; };
```
 

в конце файла добавить 
```
zone "au-team.irpo" IN {
	type master;
	file "/opt/dns/au-team.irpo";
};

zone "100.168.192.in-addr.arpa" IN { 
	type master;
	file "/opt/dns/100.168.192.in-addr.arpa";
};
zone "200.168.192.in-addr.arpa" IN {
	type master;
	file "/opt/dns/200.168.192.in-addr.arpa";
};
```

Далее копируем файл шаблона и заполняем по скринам.

```
mkdir /opt/dns
cp /var/named/named.empty /opt/dns/au-team.irpo
nano /opt/dns/au-team.irpo
```

![[cache/kYfC212XQqJZfH9JMbr5Mj-ukI1sonblFnkQTvZnlcsC65Hz82heGbKtEcsr6iJuag01TKF4qD6sMvL8HDPIgbFZ 1.jpg]]

```
cp /var/named/named.empty /opt/dns/100.168.192.in-addr.arpa 
nano 100.168.192.in-addr.arpa 
```

![[cache/6B_nma4v-zG5mwpPg3YPvE_N6-1XIvY95OwY07jMUk7N2TH568TDKF1Rry3rX-aj5zhH8fUI25mU6Di1l9B4Rt06.jpg]]

```
cp /var/named/named.empty /otp/dns/200.168.192.in-addr.arpa 
nano 200.168.192.in-addr.arpa 
```


![[cache/DjdkrNHYbS4WfmhaN-A8Aibo2j_ymMZ-H2uIpnnerOShXE1sggoXceGNiMLMz42hbSGkFcyADouCgBV9D8AwpDz4 1.jpg]]


```
chmod -R 777 /opt/dns 
```

ПРОВЕРЯЕМ КОНФИГУРАЦИЮ И ИСПРАВЛЯЕМ ОШИБКИ ЕСЛИ ЕСТЬ 
```
named-checkconf -z 
systemctl restart named 
```

Далее заходим в nmtui и меняем ДНС сервер с (10.39.0.1) на 
192.168.100.10. Так же указываем домен поиска au-team.irpo. НА HQ-SRV ТОЖЕ. ТАКЖЕ ДОБАВЛЯЕМ В nmtui в домен поиска au-team.irpo, или пингуем типа ping hq-srv.au-team.irpo



После этого в nmtui переходим на вкладку «Активировать 
подключение». Выключаем и включаем интерфейс, на который ставили ДНС. 

НА HQ-CLI И ПРОВЕРЯЕМ РАБОТОСПОБНОСТЬ 
```
ping br-rtr 
ping br-srv 
ping hq-rtr 
ping hq-srv 
ping ya.ru
```

```
timedatectl set-timezone Europe/Moscow 
timedatectl (ПРОВЕРИТЬ ЗОНУ, ПО ЗАДАНИЮ ВРЕМЯ МЕНЯТЬ НЕ 
ПРОСЯТ)
```