## Технология PPPoE (PPP over Ethernet)

**PPPoE** — это протокол, позволяющий инкапсулировать кадры PPP внутри кадров Ethernet. Он объединяет преимущества обоих миров:

- **От Ethernet:** доступность и дешевизна 

- **От PPP:** поддержка авторизации (**CHAP**), учет трафика и автоматическое назначение IP-адресов.

### Почему это важно для провайдеров?

Главная причина использования PPPoE — **аутентификация CHAP**. Провайдер проверяет оплатил ли клиент и после этого выдает пароль

---

## Архитектура подключения

Маршрутизатор пользователя соединяется с DSL-модемом по Ethernet. Внутри этого соединения создается **PPP-туннель**.

- **MTU:** Поскольку заголовок PPPoE занимает **8 байт**, стандартный размер кадра Ethernet (1500 байт) уменьшается до **1492 байт**.
    

---

## Настройка PPPoE на стороне клиента (Cisco IOS)

Настройка разделена на создание виртуального интерфейса (**Dialer**) и привязку его к физическому порту.

### 1. Настройка виртуального интерфейса (Dialer)

Это логический интерфейс, где происходит вся "магия" PPP.

Фрагмент кода

```
interface dialer 2
 encapsulation ppp              ! Включаем инкапсуляцию PPP
 ip address negotiated         ! Получаем IP от провайдера автоматически
 ppp chap hostname Fred        ! Логин CHAP
 ppp chap password Barney      ! Пароль CHAP
 ip mtu 1492                   ! Уменьшаем MTU (стандарт для PPPoE)
 dialer pool 1                 ! Привязываем логический интерфейс к пулу №1
```

### 2. Настройка физического интерфейса

Служит только "транспортом" для туннеля.

Фрагмент кода

```
interface GigabitEthernet 0/1
 no ip address                 ! На физическом интерфейсе IP не нужен
 pppoe-client dial-pool-number 1 ! Связываем с пулом Dialer-интерфейса
 pppoe enable                  ! Включаем PPPoE
 no shutdown
```

---

## Решение проблемы с загрузкой сайтов (TCP MSS)

Иногда сайты не открываются из-за того, что пакеты размером 1500 байт отбрасываются. Для решения на **внутреннем (LAN)** интерфейсе маршрутизатора настраивается корректировка размера сегмента TCP:

Фрагмент кода

```
interface GigabitEthernet 0/0 (LAN)
 ip tcp adjust-mss 1452        ! Оптимальное значение (1492 MTU - 40 байт заголовков)
```

---

## Команды для проверки и диагностики

| **Команда**               | **Описание**                                               |
| ------------------------- | ---------------------------------------------------------- |
| `show ip interface brief` | Проверка статуса (Dialer должен быть UP/UP и с IP-адресом) |
| `show pppoe session`      | Просмотр активных сессий PPPoE и их идентификаторов        |
| `show interface dialer 2` | Детальная статистика туннеля                               |
| `debug ppp negotiation`   | Анализ процесса согласования (LCP, CHAP, IPCP)             |

### 4 критические точки отказа (Checklist):

1. **LCP:** Провайдер не отвечает (физика или настройки PPPoE).
2. **Authentication:** Неверный логин или пароль CHAP.
3. **IPCP:** Провайдер не может выделить IP-адрес.
4. **MTU/MSS:** Сессия установлена, но данные (сайты) не передаются.





Команды для настройки:

```

// маршрутизатор заказчика
int dialer 2

 // ppp и ip номеронабиратели:
 encapsulation ppp
 ip add negotiated
 
 // проверка подлинности только входящих:
 ppp chap hostname Fred
 ppp chap password Barney
 
 // пул номеронабирателей должен соответствовать
 ip mtu 1492
 dialer pool 1 // он
 no sh
 
int gi 0/1
 no ip add
 pppoe enable
 pppoe-client dial-pool-number 1 // и он
 no sh
 

```


Команды для проверки:

```
sh ip int brief
sh int dialer 2
sh ip route
sh pppoe session
debug ppp negotiation
```



3.2.2.7 команды:

### ISP

```
hostname ISP

username Cust1 password 0 ciscopppoe

int virtual-Template 1
 peer default ip add pool PPPoEPOOL
 ip unnumbered gi 0/1
 ppp authentication chap
 
int gi 0/1
 no sh
 ip add 189.132.32.254 255.255.240.0
 pppoe enable group global

bba-group pppoe global
 virtual-template 1

ip local pool PPPoEPOOL 189.132.32.1 189.132.32.10

```

### Cust 1

```

hostname Cust1

int gi 0/1
 no sh
 pppoe enable
 pppoe-client dial-pool-number 1
 
int dialer 1
 dialer pool 1
 ip add negotiated
 encapsulation ppp
 ppp authentication chap
 ppp chap hostname Cust1
 ppp chap password ciscopppoe
 
ip route 0.0.0.0 0.0.0.0 Dialer1

```