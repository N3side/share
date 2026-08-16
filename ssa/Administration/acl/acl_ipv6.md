в ipv6 нет шаблонной маски

в ipv6 кроме запрета на все по умолчанию включены еще два разрешающих правила


permit icmp any any na
permit icmp any any ns

в ipv6 сетях в отличие от ipv4 нет arp протокола
(arp. по ip адресу узнает mac адрес. пакеты пропадают именно из-за этого)

Эти правила это аналог arp запроса когда устройству надо узнать mac адрес соседа по ipv6 адресу. Он отправляет сообщение запрос соседа (ns). Ответ соседа (na) это ответ на запрос ns, т.е получение mac адреса. Без этих разрешающих правил все правила ipv6 теряют смысл

Команды для настройки:

ipv6 access-list [название]
 deny ipv6 [mac]::/64 any
 permit ipv6 any any


int s 0/0/0
 pv6 traffic-filter [имя правила] in/out


permit icmp any any
permit udp any any eq domain
permit tcp any any eq domain
permit tcp host 2010::2 host 80:80:80::5 eq ftp
deny tcp host 2020::2 host 80:80:80::2 eq www
deny tcp host 2020::2 host 80:80:80::2 eq 443
permit tcp host 2020::2 host 30:30:30::2 eq 22
permit tcp host 2010::2 host 30:30:30::2 eq telnet
permit tcp 2010::/64 host 80:80:80::11 eq smtp
permit tcp 2010::/64 host 80:80:80::11 eq pop3
permit tcp 2020::/64 host 80:80:80::11 eq pop3
permit tcp 2020::/64 host 80:80:80::11 eq smtp
permit tcp 2010::/64 host 80:80:80::2 eq www
permit tcp 2010::/64 host 80:80:80::2 eq 443