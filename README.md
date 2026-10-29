<!--
Бланк звіту до лабораторної роботи «DNS».
Скопіювати цей файл у власний публічний репозиторій звіту як README.md і заповнити.
Коментарі такого вигляду в GitHub не відображаються; видаляти їх не обов'язково.
Вивід команд вставляти текстом у блоки коду, а не знімками екрана.
Порожні поля не допускаються: якщо щось не вдалося, записати, що саме і чому.
-->

# Лабораторна робота. DNS

| Поле | Значення |
|---|---|
| ПІБ | Зізевський Михайло Юрійович |
| Група | Кбзі-2.02 |
| Ідентифікатор (`sNN`) | cs-202-08 |
| Дата виконання | 25-26.10 |
| Середовище | GitHub Codespaces |
| Версія `dig` | 9.18.39 |

## Параметри варіанта

<!-- Переписати з картки stand/cards/sNN.md. -->

| Параметр | Значення |
|---|---|
| `$TTL`, с | 45 |
| TTL запису SOA, с | 20 |
| Поле `minimum` SOA, с | 40 |
| Псевдонім (CNAME на `www`) | shop |
| Другий хост і його TTL, с | next, 35 |
| Пріоритет MX | 15 |
| Токен TXT | 5887307d |
| Ім'я для завдання 5 | new-kxzrr.cs-202-08.lab |

## Завдання 1. Складові URI

### 1.1. Розбір

<!-- Відсутню складову позначати «—». Порт за замовчуванням записувати з позначкою «(за замовч.)». -->

| № | scheme | userinfo | host | port | path | query | fragment |
|---|---|---|---|---|---|---|---|
| 1 | http   | -  | www.cs-202-08.lab | 80 (за замовч.) | /catalog/reports | lang=3&q=uk | comments |
| 2 | http   | -  | shop.cs-202-08.lab | 80 | /reports/	| lang=%D0%BA%D0%B8%D1%97%D0%B2	| -|
| 3 | http   | user:pass | next.cs-202-08.lab | 80 (за замовч.)	| /catalog | - | summary |
| 4 | https  | - | www.cs-202-08.lab | 9443 | /catalog/../reports | - | - |
| 5 | mailto | - | - | - | admin@cs-202-08.lab | subject=catalog | - |

| № | Ім'я, що запитується в DNS | Заголовок `Host` | Ціль запиту (request target) |
|---|---|---|---|
| 1 | www.cs-202-08.lab	 | www.cs-202-08.lab | /catalog/reports?lang=3&q=uk |
| 2 | shop.cs-202-08.lab | shop.cs-202-08.lab | /reports/?lang=%D0%BA%D0%B8%D1%97%D0%B2 |
| 3 | next.cs-202-08.lab | next.cs-202-08.lab | /catalog |
| 4 | www.cs-202-08.lab  | www.cs-202-08.lab:9443 | /reports (після нормалізації); /catalog/../reports при --path-as-is |
| 5 | — (HTTP-запиту нема) | - | - |

**Декодоване значення параметра URI 2:** %D0%BA%D0%B8%D1%97%D0%B2 декодується як UTF-8 у «київ» (байти D0BA = к, D0B8 = и, D197 = ї, D0B2 = в).

**Чому URI 5 не має складової host, хоча містить символ `@`:** У схемі mailto: немає подвійного слешу //, отже, немає й компонента authority, у якому буває userinfo і host. Усе до ? є path, тобто admin@cs-202-08.lab це адрес поштової скриньки повністю, а @ у ній звичайний символ шляху. Хост cs-202-08.lab з'явиться лише тоді, коли поштовий клієнт шукатиме MX-запис, але до разбору URI це не належить.

### 1.2. Перевірка (після завдання 3)

<!-- Для URI 1–3: рядки запиту (починаються з ">") і тіло відповіді сервера. -->

```text
> GET /catalog/reports?lang=3&q=uk HTTP/1.1
> Host: www.cs-202-08.lab
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.24.0 (Ubuntu)
< Date: Mon, 26 Oct 2026 17:57:00 GMT
< Content-Type: text/plain
< Content-Length: 140
< Connection: keep-alive
< 
server: web-a
Host: www.cs-202-08.lab
request URI: /catalog/reports?lang=3&q=uk
path: /catalog/reports
query: lang=3&q=uk
client: 10.53.1.1
* Connection #0 to host www.cs-202-08.lab left intact

> GET /reports/?lang=%D0%BA%D0%B8%D1%97%D0%B2 HTTP/1.1
> Host: shop.cs-202-08.lab
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.24.0 (Ubuntu)
< Date: Mon, 26 Oct 2026 17:58:17 GMT
< Content-Type: text/plain
< Content-Length: 163
< Connection: keep-alive
< 
server: web-a
Host: shop.cs-202-08.lab
request URI: /reports/?lang=%D0%BA%D0%B8%D1%97%D0%B2
path: /reports/
query: lang=%D0%BA%D0%B8%D1%97%D0%B2
client: 10.53.1.1
* Connection #0 to host shop.cs-202-08.lab left intact

> GET /catalog HTTP/1.1
> Host: next.cs-202-08.lab
> Authorization: Basic dXNlcjpwYXNz
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.24.0 (Ubuntu)
< Date: Mon, 26 Oct 2026 17:58:35 GMT
< Content-Type: text/plain
< Content-Length: 102
< Connection: keep-alive
< 
server: web-b
Host: next.cs-202-08.lab
request URI: /catalog
path: /catalog
query: 
client: 10.53.1.1
* Connection #0 to host next.cs-202-08.lab left intact
```

Значення заголовка `Authorization` і результат декодування:

```text
Authorization: Basic dXNlcjpwYXNz

$ echo 'dXNlcjpwYXNz' | base64 -d
user:pass
```

URI 4: етап, на якому запит не вдається (пошук імені / TCP / TLS), і рядки виводу, що це підтверджують:

```text
root@ns-cs-202-08:/# curl -v 'https://www.cs-202-08.lab:9443/catalog/../reports'
* Host www.cs-202-08.lab:9443 was resolved.
* IPv6: (none)
* IPv4: 10.53.0.80
*   Trying 10.53.0.80:9443...
* connect to 10.53.0.80 port 9443 from 10.53.1.1 port 41786 failed: Connection refused
* Failed to connect to www.cs-202-08.lab port 9443 after 1 ms: Couldn't connect to server
* Closing connection
curl: (7) Failed to connect to www.cs-202-08.lab port 9443 after 1 ms: Couldn't connect to server
```

Запит не вдається на етапі TCP-з'єднання. Пошук імені успішний (Host www.cs-202-08.lab:9443 was resolved, IPv4: 10.53.0.80).

```text
Далі Trying 10.53.0.80:9443... і Connection refused: на порту 9443 нічого не слухає. Рядків про TLS-рукостискання немає, бо до TLS справа не дійшла.

root@ns-cs-202-08:/# curl -v 'http://www.cs-202-08.lab/catalog/../reports'
* Host www.cs-202-08.lab:80 was resolved.
* IPv6: (none)
* IPv4: 10.53.0.80
*   Trying 10.53.0.80:80...
* Connected to www.cs-202-08.lab (10.53.0.80) port 80
> GET /reports HTTP/1.1
> Host: www.cs-202-08.lab
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.24.0 (Ubuntu)
< Date: Mon, 26 Oct 2026 18:11:46 GMT
< Content-Type: text/plain
< Content-Length: 101
< Connection: keep-alive
< 
server: web-a
Host: www.cs-202-08.lab
request URI: /reports
path: /reports
query: 
client: 10.53.1.1
* Connection #0 to host www.cs-202-08.lab left intact

root@ns-cs-202-08:/# curl -v --path-as-is 'http://www.cs-202-08.lab/catalog/../reports'
* Host www.cs-202-08.lab:80 was resolved.
* IPv6: (none)
* IPv4: 10.53.0.80
*   Trying 10.53.0.80:80...
* Connected to www.cs-202-08.lab (10.53.0.80) port 80
> GET /catalog/../reports HTTP/1.1
> Host: www.cs-202-08.lab
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.24.0 (Ubuntu)
< Date: Mon, 26 Oct 2026 18:11:53 GMT
< Content-Type: text/plain
< Content-Length: 112
< Connection: keep-alive
< 
server: web-a
Host: www.cs-202-08.lab
request URI: /catalog/../reports
path: /reports
query: 
client: 10.53.1.1
* Connection #0 to host www.cs-202-08.lab left intact
```

| Запит | `request URI`, отриманий сервером | `path`, отриманий сервером |
|---|---|---|
| звичайний | 	/reports | /reports |
| `--path-as-is` | /catalog/../reports | /reports |

**Розбіжності між URI та тим, що отримав сервер, і їх пояснення:**
Фрагмент (#comments, #summary) серверу не надходить у рядку GET і в request URI його немає. Фрагмент обробляє лише клієнт.
Порт 80 у URI 2 вказано явно, але в заголовку Host його немає (Host: shop.cs-202-08.lab): curl не додає стандартний порт.
userinfo у URI 3 (user:pass@) не потрапило ні в Host, ні в ціль запиту (GET /catalog). curl перетворив його на заголовок Authorization: Basic dXNlcjpwYXNz, що декодується в user:pass.
Percent-кодування у параметрі URI 2 сервер отримав без змін (lang=%D0%BA%D0%B8%D1%97%D0%B2)
Сегмент .. у URI 4: за замовчуванням curl нормалізує шлях ще до відправки (GET /reports). З --path-as-is curl надсилає GET /catalog/../reports, але сервер у полі path усе одно повідомив /reports: nginx нормалізує шлях сам, тоді як request URI залишається сирим.
Різні вебсервери. www і shop відповіли від web-a (10.53.0.80), бо shop — псевдонім www. next відповів від web-b (10.53.0.81), бо його адреса .81.
URI 4 з портом 9443 не вдається на етапі TCP, тому сервер його взагалі не отримав.

## Завдання 2. Проходження ієрархії вручну

**Команди**
```text
dig @10.53.0.2 ns.cs-202-08.lab A +norec
dig @10.53.0.3 ns.cs-202-08.lab A +norec
dig @10.53.1.1 ns.cs-202-08.lab A +norec
```

**Вивід**
```text
root@ns-cs-202-08:/# dig @10.53.0.2 ns.cs-202-08.lab A +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.0.2 ns.cs-202-08.lab A +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 55969
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 6d2f94db3e9f6f00010000006ade13b865aa40f78847b09f (good)
;; QUESTION SECTION:
;ns.cs-202-08.lab.              IN      A

;; AUTHORITY SECTION:
lab.                    172800  IN      NS      ns1.nic.lab.

;; ADDITIONAL SECTION:
ns1.nic.lab.            172800  IN      A       10.53.0.3

;; Query time: 80 msec
;; SERVER: 10.53.0.2#53(10.53.0.2) (UDP)
;; WHEN: Sun Oct 25 14:35:36 UTC 2026
;; MSG SIZE  rcvd: 117

root@ns-cs-202-08:/# dig @10.53.0.3 ns.cs-202-08.lab A +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.0.3 ns.cs-202-08.lab A +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 38907
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: e8c9cd8091e78787010000006ade156ad99437aad53e1d9c (good)
;; QUESTION SECTION:
;ns.cs-202-08.lab.              IN      A

;; AUTHORITY SECTION:
cs-202-08.lab.          3600    IN      NS      ns.cs-202-08.lab.

;; ADDITIONAL SECTION:
ns.cs-202-08.lab.       3600    IN      A       10.53.1.1

;; Query time: 81 msec
;; SERVER: 10.53.0.3#53(10.53.0.3) (UDP)
;; WHEN: Sun Oct 25 14:42:50 UTC 2026
;; MSG SIZE  rcvd: 103

root@ns-cs-202-08:/# dig @10.53.1.1 ns.cs-202-08.lab A +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.1.1 ns.cs-202-08.lab A +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5329
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: ac3341176d328d44010000006ade15a19d3e0c12fe6fb3c5 (good)
;; QUESTION SECTION:
;ns.cs-202-08.lab.              IN      A

;; ANSWER SECTION:
ns.cs-202-08.lab.       45      IN      A       10.53.1.1

;; AUTHORITY SECTION:
cs-202-08.lab.          45      IN      NS      ns.cs-202-08.lab.

;; Query time: 0 msec
;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)
;; WHEN: Sun Oct 25 14:43:45 UTC 2026
;; MSG SIZE  rcvd: 103
```

| Сервер | status | flags | ANSWER | AUTHORITY | ADDITIONAL |
|---|---|---|---|---|---|
| 10.53.0.2 | NOERROR | qr | 0 | 1(NS lab. -> ns1.nic.lab.) | 1(A ns1.nic.lab. -> 10.53.0.3), без OPT |
| 10.53.0.3 | NOERROR | qr | 0 |1 (NS cs-202-08.lab. -> ns.cs-202-08.lab.) | 1 (A ns.cs-202-08.lab. -> 10.53.1.1), без OPT|
| 10.53.1.1 | NOERROR | qr aa | 1 | 1 (NS cs-202-08.lab.)| 0, без OPT |

**Вивід `dig ns.cs-202-08.lab A +trace`:**

```text
; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> ns.cs-202-08.lab A +trace
;; global options: +cmd
.                       85625   IN      NS      a.root-servers.net.
;; Received 59 bytes from 10.53.0.53#53(10.53.0.53) in 1 ms

lab.                    172800  IN      NS      ns1.nic.lab.
;; Received 117 bytes from 10.53.0.2#53(a.root-servers.net) in 81 ms

cs-202-08.lab.          3600    IN      NS      ns.cs-202-08.lab.
;; Received 103 bytes from 10.53.0.3#53(ns1.nic.lab) in 40 ms

ns.cs-202-08.lab.       45      IN      A       10.53.1.1
cs-202-08.lab.          45      IN      NS      ns.cs-202-08.lab.
;; Received 103 bytes from 10.53.1.1#53(ns.cs-202-08.lab) in 0 ms
```

| Запит | TTL запису NS | Прапорець `aa` | Секція, у якій міститься NS |
|---|---|---|---|
| `dig @10.53.0.3 cs-202-08.lab NS +norec` | 3600 | ні | AUTHORITY |
| `dig @10.53.1.1 cs-202-08.lab NS +norec` | 45 | так| ANSWER |

**1. Які відповіді є відсиланнями і за якими ознаками це визначено?**
Відсиланнями є відповіді 10.53.0.2 і 10.53.0.3. Ознаки: секція ANSWER порожня, в AUTHORITY є записи NS з вказівкою, до кого звертатися далі, немає прапорця aa, а в ADDITIONAL є адреса наступного сервера. Відповідь 10.53.1.1 є остаточною: в ANSWER є шуканий запис A, прапорець aa встановлений.
**2. Для чого у відповіді TLD присутній запис у секції ADDITIONAL і що сталося б, якби його не було?**
Це glue-запис. Сервер імен ns.cs-202-08.lab розташований усередині зони cs-202-08.lab, для якої він сам і є сервером. Щоб дізнатися його адресу, треба звернутися до сервера цієї зони, а щоб до нього звернутися, треба знати його адресу. Виходить замкнене коло. Без glue-запису резолвер не знав би, куди надсилати наступний запит, і розв'язання імені завершилося б помилкою (SERVFAIL).
**3. Яка з двох копій запису NS є авторитативною і чому?**
Авторитативною є копія з 10.53.1.1. Вона має прапорець aa, міститься в секції ANSWER і береться з файлу самої зони (TTL 45 с, це $TTL). Копія на TLD (10.53.0.3) є лише делегуванням: сервер не авторитативний для cs-202-08.lab, тому відповідь без aa, запис лежить в AUTHORITY, а TTL 3600 с задано в зоні lab.

## Завдання 3. Заповнення власної зони

**Вивід `zone-reload`:**

```text
zone cs-202-08.lab/IN: loaded serial 2026102502
OK
zone reload queued
```

<!-- Шість запитів до 10.53.1.1 з +norec, по одному на кожен рядок таблиці картки. -->

| Запис | status | Прапорець `aa` | Значення в ANSWER | TTL | Відповідає картці (так/ні) |
|---|---|---|---|---|---|
| `www` A | NOERROR | так | 10.53.0.80 |45 |так |
| псевдонім CNAME (shop) | NOERROR | так | www.cs-202-08.lab. |45 |так |
| `mail` A | NOERROR |так | 10.53.0.80 |45 |так |
| `@` MX | NOERROR | так| 15 mail.cs-202-08.lab. |45 | так| 
| `@` TXT | NOERROR |так | “lab-token=5887307d”	 |45 |так |
| другий хост (next) A | NOERROR |так | 10.53.0.81 | 35	|так |

Порівняння відповідей резолвера та власного сервера:

```text
root@ns-cs-202-08:/# dig shop.cs-202-08.lab 

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> shop.cs-202-08.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 62424
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;shop.cs-202-08.lab.            IN      A

;; ANSWER SECTION:
shop.cs-202-08.lab.     45      IN      CNAME   www.cs-202-08.lab.
www.cs-202-08.lab.      45      IN      A       10.53.0.80

;; Query time: 41 msec
;; SERVER: 10.53.0.53#53(10.53.0.53) (UDP)
;; WHEN: Sun Oct 25 15:27:31 UTC 2026
;; MSG SIZE  rcvd: 81

root@ns-cs-202-08:/# dig @10.53.1.1 shop.cs-202-08.lab +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.1.1 shop.cs-202-08.lab +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 60465
;; flags: qr aa; QUERY: 1, ANSWER: 2, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 74b3cb07220d40b6010000006ade2005295d69c15fec5858 (good)
;; QUESTION SECTION:
;shop.cs-202-08.lab.            IN      A

;; ANSWER SECTION:
shop.cs-202-08.lab.     45      IN      CNAME   www.cs-202-08.lab.
www.cs-202-08.lab.      45      IN      A       10.53.0.80

;; AUTHORITY SECTION:
cs-202-08.lab.          45      IN      NS      ns.cs-202-08.lab.

;; ADDITIONAL SECTION:
ns.cs-202-08.lab.       45      IN      A       10.53.1.1

;; Query time: 1 msec
;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)
;; WHEN: Sun Oct 25 15:28:05 UTC 2026
;; MSG SIZE  rcvd: 142

root@ns-cs-202-08:/# dig cs-202-08.lab MX

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> cs-202-08.lab MX
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 8435
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;cs-202-08.lab.                 IN      MX

;; ANSWER SECTION:
cs-202-08.lab.          45      IN      MX      15 mail.cs-202-08.lab.

;; Query time: 0 msec
;; SERVER: 10.53.0.53#53(10.53.0.53) (UDP)
;; WHEN: Sun Oct 25 15:28:24 UTC 2026
;; MSG SIZE  rcvd: 63

root@ns-cs-202-08:/# dig @10.53.1.1 cs-202-08.lab MX +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.1.1 cs-202-08.lab MX +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 44664
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 7d4857b1eb66c2d4010000006ade2034b99557525a154fe5 (good)
;; QUESTION SECTION:
;cs-202-08.lab.                 IN      MX

;; ANSWER SECTION:
cs-202-08.lab.          45      IN      MX      15 mail.cs-202-08.lab.

;; AUTHORITY SECTION:
cs-202-08.lab.          45      IN      NS      ns.cs-202-08.lab.

;; ADDITIONAL SECTION:
mail.cs-202-08.lab.     45      IN      A       10.53.0.80
ns.cs-202-08.lab.       45      IN      A       10.53.1.1

;; Query time: 0 msec
;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)
;; WHEN: Sun Oct 25 15:28:52 UTC 2026
;; MSG SIZE  rcvd: 140
```

| Запит | flags | ANSWER (кількість, типи) | AUTHORITY (кількість, типи) | ADDITIONAL (кількість, типи) | TTL |
|---|---|---|---|---|---|
| псевдонім через resolver-a |qr rd ra |2 (CNAME, A) | 0| 0 (лише OPT)	|45 |
| псевдонім через 10.53.1.1 |qr aa	 | 2 (CNAME, A)|1 (NS) |1 (A ns), без OPT	 | 45|
| MX через resolver-a | qr rd ra|1 (MX) |0 |0 (лише OPT)	 |45 |
| MX через 10.53.1.1 |qr aa	 | 1 (MX)|1 (NS) |2 (A mail, A ns), без OPT | 45 | 

**Пояснення відмінностей:**
Власний сервер є авторитативним, тому відповідь має прапорець aa, а разом з ANSWER він додає секцію AUTHORITY (NS зони) і в ADDITIONAL допоміжні адреси (ns, а для MX ще й mail). Резолвер не авторитативний: у його відповіді немає aa, але є ra (рекурсія доступна) і rd (клієнт її запитав). Він повертає лише відповідь на питання і не передає NS та допоміжні записи, тому AUTHORITY і ADDITIONAL порожні. Самі записи в ANSWER і TTL збігаються, бо резолвер отримав їх щойно з власного сервера. Якби запит повторили пізніше, TTL у резолвера зменшувався б.

Повідомлення `zone-check` після тимчасового додавання `@ IN CNAME www`:

```text
dns_master_load: /lab/zone/cs-202-08.lab.zone:21: cs-202-08.lab: CNAME and other data
zone cs-202-08.lab/IN: loading from master file /lab/zone/cs-202-08.lab.zone failed: CNAME and other data
zone cs-202-08.lab/IN: not loaded due to errors.
```

**1. Який TTL має кожен запис зони і звідки це значення взято?**
Записи www, shop, mail, @ MX, @ TXT, а також NS і ns A мають TTL 45 с. Вони не мають власного TTL у рядку, тому беруть значення директиви $TTL 45 на початку файлу зони. Запис next має TTL 35 с, бо він заданий явно в самому рядку запису. Запис SOA має TTL 20 с, він також заданий явно (@ 20 IN SOA).
**2. Скільки записів містить секція ANSWER у відповіді на запит псевдоніма і чому?**
Два: CNAME shop -> www і A www -> 10.53.0.80. Запитували тип A, а shop є псевдонімом. Сервер повертає сам CNAME і, оскільки цільове ім'я www лежить у тій самій зоні, одразу додає його адресу, щоб клієнтові не потрібен був другий запит. Якщо запитувати саме тип CNAME (перша серія запитів), у ANSWER буде один запис.
**3. Чому запис CNAME неможливо розмістити для імені `@`?**
CNAME означає, що ім'я є псевдонімом іншого імені, і для нього не може бути інших записів. Але @ (вершина зони) завжди містить SOA і NS, а в нашому випадку ще й MX та TXT. Отже, CNAME суперечив би їм. Це правило з RFC 1034 та RFC 2181.

## Завдання 4. TTL і видимість змін

Два запити з інтервалом 5 с:

```text
root@ns-cs-202-08:/# dig www.cs-202-08.lab +noall +answer
sleep 5
dig www.cs-202-08.lab +noall +answer
www.cs-202-08.lab.      45      IN      A       10.53.0.80
www.cs-202-08.lab.      40      IN      A       10.53.0.80
```

**Прогноз** (записати до вимірювання). Формула часу появи нової адреси через залишок R, найбільша можлива затримка:

Резолвер зберігає запис у кеші й відлічує TTL від 45 с (значення $TTL) до нуля, не звертаючись до сервера. Тому після zone-reload він далі віддаватиме стару адресу 10.53.0.80, доки не спливе залишок TTL R, який був у кеші в момент зміни. Нова адреса 10.53.0.81 з'явиться після спливання запису та першого наступного запиту до сервера.
Формула: затримка =+- R, час появи нової адреси = час zone-reload + R.
Найбільша можлива затримка: R не може перевищувати TTL запису, тобто затримка ≤ 45 с. Вона досягається, коли запис потрапив у кеш одразу перед зміною (R =+- 45).

Перевірка, що сервер до `zone-reload` повертає стару адресу:

```text
root@ns-cs-202-08:/# dig @10.53.1.1 www.cs-202-08.lab +norec +short
10.53.0.80
```

Залишок R і час (крок 4), час `zone-reload` (крок 5), перевірка на сервері (крок 6):

```text
$ date +%T; dig www.cs-202-08.lab +noall +answer; zone-reload; date +%T
19:23:47
www.cs-202-08.lab.      45      IN      A       10.53.0.80
zone cs-202-08.lab/IN: loaded serial 2026102507
OK
zone reload queued
19:23:47

$ dig @10.53.1.1 www.cs-202-08.lab +norec +short
10.53.0.81
```

Очікуваний час появи нової адреси (підставити R у формулу):
R = 45 с, час зміни 19:23:47. Очікуваний час появи = 19:23:47 + 45 с = 19:24:32.

Вивід циклу спостереження (перші два рядки, два останні рядки зі старою адресою, рядок з новою адресою):

```text
19:24:11 10.53.0.80
19:24:13 10.53.0.80
...
19:24:29 10.53.0.80
19:24:31 10.53.0.80
19:24:33 10.53.0.81
```

Рядок `server:` з відповіді `curl`:
server: web-b
Раніше, до зміни, www відповідав сервер web-a, зараз web-b (адреса 10.53.0.81).

| Час зміни | Очікуваний час | Спостережений час | Виміряна затримка, с | Прогноз, с |
|---|---|---|---|---|
| 19:23:47|19:24:32| 19:24:33| 46|	45 |

**1. Чим пояснюється розбіжність між прогнозом і виміряним значенням?**
Розбіжність у 1 с невелика і пояснюється дискретністю вимірювання. Цикл опитує резолвер раз на 2 с, тому момент спливання запису (19:24:32) потрапляє між двома опитуваннями, 19:24:31 і 19:24:33. Фіксується перше опитування після спливання. Додатково TTL у кеші відлічується цілими секундами, а команда dig і date виконуються з невеликою затримкою. Прогноз «затримка дорівнює R» підтвердився.
**2. Якою була б найбільша можлива затримка для цього запису і за яких умов?**
Найбільша затримка дорівнює TTL запису, тобто 45 с ($TTL). Вона досягається, коли запис потрапив у кеш резолвера безпосередньо перед зміною зони (R = 45). У нашому експерименті умови були майже саме такими, тому виміряне значення (46 с) лежить на верхній межі. Якщо запис у кеші вже наполовину витрачений, затримка менша.
**3. Як слід діяти, щоб перенести сервіс на нову адресу, якщо TTL запису становить 86 400 с?**
Заздалегідь, не менш ніж за добу до перенесення (за повний старий TTL), зменшити TTL запису до невеликого значення, наприклад 60 с, і збільшити serial. Дочекатися, поки всі кеші, що зберігали запис зі старим TTL, спливуть. Потім змінити адресу, після чого клієнти побачать зміну протягом хвилини. Коли сервіс стабільно працює на новій адресі, повернути великий TTL. Старий сервер залишити працюючим на час міграції.

## Завдання 5. Негативне кешування

**Прогноз** (записати до вимірювання). Формула за RFC 2308 і розраховане значення:
За RFC 2308 час негативного кешування дорівнює min(TTL запису SOA, поле minimum) = min(20; 40) = 20 с. Відлік іде від моменту, коли резолвер отримав NXDOMAIN, тому ім'я має зарезолвитися приблизно через 20 с після першого запиту.

Перевірка, що сервер до `zone-reload` повертає `NXDOMAIN`:

```text
root@ns-cs-202-08:/# dig @10.53.1.1 new-kxzrr.cs-202-08.lab +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.1.1 new-kxzrr.cs-202-08.lab +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 49860
;; flags: qr aa; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 858afdad19f725f0010000006adfabc6f6b3eb5ae231dbe5 (good)
;; QUESTION SECTION:
;new-kxzrr.cs-202-08.lab.       IN      A

;; AUTHORITY SECTION:
cs-202-08.lab.          20      IN      SOA     ns.cs-202-08.lab. admin.cs-202-08.lab. 2026102507 3600 600 86400 40

;; Query time: 0 msec
;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)
;; WHEN: Mon Oct 26 19:36:38 UTC 2026
;; MSG SIZE  rcvd: 125
```

Запит через resolver-a (крок 3), час `zone-reload` (крок 4), перевірка на сервері (крок 5):

```text
19:37:15

$ dig new-kxzrr.cs-202-08.lab
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 32998
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1
;; AUTHORITY SECTION:
cs-202-08.lab.          20      IN      SOA     ns.cs-202-08.lab. admin.cs-202-08.lab. 2026102507 3600 600 86400 40
;; SERVER: 10.53.0.53#53(10.53.0.53) (UDP)

$ zone-reload; date +%T
zone cs-202-08.lab/IN: loaded serial 2026102508
OK
zone reload queued
19:37:15

$ dig @10.53.1.1 new-kxzrr.cs-202-08.lab +norec +short
10.53.0.80
```

Вивід циклу спостереження (перші два рядки, останній рядок з `NXDOMAIN`, перший рядок з `NOERROR`):

```text
19:37:15 status: NXDOMAIN
19:37:17 status: NXDOMAIN
...
19:37:34 status: NXDOMAIN
19:37:36 status: NOERROR
```

| Негативну відповідь закешовано | Запис створено | Ім'я резолвиться через resolver-a | Виміряно, с | Прогноз, с |
|---|---|---|---|---|
| 19:37:15 | 19:37:15 | 19:37:36 | 21|  20 |

<!-- Якщо завдання повторювалося з іншим ім'ям (new2-…), зазначити це й причину. -->

**1. Чому тривалість негативного кешування визначається записом SOA, а не записом, який запитували?**
Запитуваного запису не існує, тому в нього немає власного TTL. Щоб резолвер знав, скільки часу вважати ім'я неіснуючим, авторитативний сервер у відповіді NXDOMAIN додає в секцію AUTHORITY запис SOA зони. За RFC 2308 час негативного кешування дорівнює меншому з TTL цього SOA і поля minimum. У нашому випадку це min(20; 40) = 20 с, і виміряна затримка (21 с) це підтверджує.

**2. Чи збігається TTL запису SOA в секції AUTHORITY з TTL запису SOA у файлі зони? Поясніть.**
У нашому експерименті збігається: у файлі зони TTL SOA дорівнює 20 с, і відповідь резолвера показала 20. Але збіг не принциповий. Резолвер віддає не значення з файлу, а значення, обмежене за RFC 2308 до min(TTL SOA, minimum), і далі зменшує його при кожному повторному запиті з кешу. Оскільки в нашому варіанті 20 менше за 40, ці величини збігаються, і за цими даними їх не відрізнити. Якби minimum був меншим, ніж TTL SOA, резолвер показав би саме minimum, а не значення з файлу.

**3. Скрипт розгортання перевіряє ім'я сервісу до створення запису, а потім одразу після. Що він побачить і як це виправити?**
Перша перевірка отримає NXDOMAIN, і резолвер запам'ятає його на 20 с. Друга перевірка одразу після створення запису піде в кеш і знову поверне NXDOMAIN, хоча запис уже існує. Скрипт помилково вирішить, що розгортання не вдалося. Як виправити: перевіряти після створення напряму на авторитативному сервері (dig @10.53.1.1 ... +norec), який кешу не має; або повторювати перевірку через резолвер з паузою, більшою за час негативного кешування; або не робити попередню перевірку існування імені через загальний резолвер; або зменшити minimum у SOA, якщо такі розгортання часті.

## Завдання 6. Затримка пошуку імені

**Прогноз** (записати до вимірювання). До яких серверів звернеться resolver-a:

| Ім'я | Перший запит | Повторний запит |
|---|---|---|
| A: нове ім'я у власній зоні | лише власний сервер 10.53.1.1 (делегування cs-202-08.lab уже в кеші) | кеш, жодного сервера |
| B: нове ім'я в зоні `lab.` |лише TLD 10.53.0.3 (делегування lab. у кеші, ім'я не існує) |кеш |
| C: ім'я в неіснуючому TLD | лише root 10.53.0.2 (відповідь NXDOMAIN від кореня)| кеш|
| D: `www.cs-202-08.lab` після перезапуску резолвера | root -> TLD -> власний сервер (кеш порожній)|кеш |

Час першого запиту =+- сума затримок усіх серверів, до яких звернувся резолвер; повторного =+- 0 мс (кеш)

Вивід вимірювань:

```text
$ date +%T; A=y$RANDOM.cs-202-08.lab; dig $A | grep 'Query time'; dig $A | grep 'Query time'
19:48:23
;; Query time: 2 msec
;; Query time: 0 msec

$ date +%T; B=x$RANDOM.lab; dig $B | grep 'Query time'; dig $B | grep 'Query time'
19:48:31
;; Query time: 40 msec
;; Query time: 0 msec

$ date +%T; C=x$RANDOM.zz$RANDOM; dig $C | grep 'Query time'; dig $C | grep 'Query time'
19:48:39
;; Query time: 81 msec
;; Query time: 1 msec

$ docker compose -f stand/compose.yaml restart resolver-a      # термінал codespace
$ date +%T; dig www.cs-202-08.lab | grep 'Query time'; dig www.cs-202-08.lab | grep 'Query time'
19:51:01
;; Query time: 402 msec
;; Query time: 1 msec
```

| Ім'я | Перший запит, мс | Повторний, мс | Сервери, до яких звернувся резолвер (за журналами) |
|---|---|---|---|
| A (`y15678.cs-202-08.lab`) | 2 | 0 | власний сервер 10.53.1.1 (19:48:23.680); у журналах root і TLD запитів немає |
| B (`x20317.lab`) | 40 | 0 | TLD 10.53.0.3 (19:48:31.789); у журналі root запиту немає |
| C (`x23143.zz905`) | 81 | 1 | root 10.53.0.2 (19:48:39.319); у журналі TLD запиту немає |
| D (`www.cs-202-08.lab`) | 402 | 1 | root (3 запити) -> TLD -> власний сервер |


Значення `time_namelookup` з `curl` після перезапуску резолвера:

```text
$ docker compose -f stand/compose.yaml restart resolver-a      # термінал codespace
$ curl -so /dev/null -w '%{time_namelookup}\n' http://www.cs-202-08.lab/
0.402710
```
Перезапуски `resolver-a` (очищення кешу для вимірювання за методичкою): ≈19:50 перед першим вимірюванням D; ≈19:51:50 перед `curl`; ≈19:56 для повторного вимірювання D (402 мс).

Фрагменти журналів запитів root, TLD і власного сервера для рядка D:

```text
root: 26-Oct-2026 19:51:01.347 ... query: . IN NS
root: 26-Oct-2026 19:51:01.428 ... query: a.root-servers.net IN A
root: 26-Oct-2026 19:51:01.508 ... query: www.cs-202-08.lab IN A
tld:  26-Oct-2026 19:51:01.629 ... query: www.cs-202-08.lab IN A
own:  26-Oct-2026 19:51:01.669 ... query: www.cs-202-08.lab IN A
```

Журнали для рядків A, B, C (підтвердження стовпця «Сервери»):

```text
own:  26-Oct-2026 19:48:23.680 client @0x7761c5296598 10.53.0.53#59277 (y15678.cs-202-08.lab): query: y15678.cs-202-08.lab IN A -E(0)DC (10.53.1.1)
tld:  26-Oct-2026 19:48:31.789 client @0x7cb77c004488 10.53.0.53#43517 (x20317.lab): query: x20317.lab IN A -E(0)DC (10.53.0.3)
root: 26-Oct-2026 19:48:39.319 client @0x7cdf8006ac88 10.53.0.53#33751 (x23143.zz905): query: x23143.zz905 IN A -E(0)DC (10.53.0.2)
```

1. **Яку затримку вносить кожен із серверів root, TLD і власний сервер?**
Root додає ≈ 80 мс (рядок C: 81 мс, опитувався лише root). TLD додає ≈ 40 мс (рядок B: 40 мс, опитувався лише TLD). Власний сервер додає 0–2 мс (рядок A: 2 мс). Повторні запити коштують 0–1 мс, бо беруться з кешу. Рядок D (402 мс) коштує дорожче за просту суму 3 × 80 + 40 + 0 = 280 мс: різниця ≈ 120 мс залишається неповністю поясненою.

2. **Чому в рядку A перший запит не проходить через root і TLD, хоча ім'я нове?**
Резолвер уже знав, де шукати авторитативний сервер зони: відсилання для `lab.` і `cs-202-08.lab.` закешовані за попередніми запитами (TTL відсилань 172800 с і 3600 с). Нове лише саме ім'я, тому резолвер одразу питає власний сервер. Журнали це підтверджують: запиту `y15678.cs-202-08.lab` у root і TLD немає.

3. **Вебсторінка завантажує ресурси з п'яти різних доменів. Як це впливає на час першого відкриття сторінки?**
Кожне нове ім'я потребує окремого пошуку. Без кешу це від ≈ 40–80 мс (якщо відсилання відомі) до ≈ 400 мс (повний ланцюжок, як у рядку D), і цей час додається до TCP/TLS-з'єднання з кожним доменом. У гіршому послідовному випадку це ≈ 5 × 0,4 с ≈ 2 с. Браузер частково розпаралелює пошук, але домени ресурсів стають відомі лише після розбору HTML, тож перше відкриття повільніше за наступні, де все береться з кешу.

## Завдання 7. Інцидент: зупинка власного сервера

**Прогноз** (записати до вимірювання). Що поверне кожен резолвер після зупинки сервера та спливання TTL і скільки часу займе відповідь:
Запис next має TTL 35 с, а запит робиться через 45 с після зупинки сервера, тому в обох резолверах запис уже спливе. Для нової відповіді потрібен власний сервер 10.53.1.1, який зупинено.
resolver-a поверне SERVFAIL без записів в ANSWER, бо спливший запис він не віддає. Відповідь буде не миттєвою: резолвер деякий час чекатиме на недоступний сервер (очікую кілька секунд).
resolver-b (serve-stale, RFC 8767) віддасть застарілий запис із кешу: NOERROR, 10.53.0.81, з коротким TTL і майже миттєво.


Кешування запису в обох резолверах (крок 2) і час зупинки сервера (крок 3):

```text
$ dig @10.53.0.53 next.cs-202-08.lab +noall +answer
next.cs-202-08.lab.     35      IN      A       10.53.0.81
$ dig @10.53.0.54 next.cs-202-08.lab +noall +answer
next.cs-202-08.lab.     35      IN      A       10.53.0.81
$ named-stop; date +%T; sleep 45
20:21:50
```

Запити із заміром часу (крок 5):

```text
$ time dig @10.53.0.53 next.cs-202-08.lab +tries=1 +timeout=30
;; communications error to 10.53.0.53#53: timed out
;; no servers could be reached
real    0m30.058s

$ time dig @10.53.0.53 next.cs-202-08.lab +tries=1 +timeout=30
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 25876
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1
;; Query time: 0 msec
;; WHEN: Mon Oct 26 20:23:05 UTC 2026
real    0m0.020s

$ time dig @10.53.0.54 next.cs-202-08.lab +tries=1 +timeout=30
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 35860
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; ANSWER SECTION:
next.cs-202-08.lab.     30      IN      A       10.53.0.81
;; Query time: 1800 msec
;; WHEN: Mon Oct 26 20:23:07 UTC 2026
real    0m1.820s
```

| Запит | status або тайм-аут | ANSWER, TTL | `real` |
|---|---|---|---|
| resolver-a, перший | тайм-аут (`no servers could be reached`) | — | 30,058 с |
| resolver-a, другий | SERVFAIL | порожня | 0,020 с |
| resolver-b | NOERROR | 10.53.0.81, TTL 30 | 1,820 с |

Час `named-start` і вивід циклу спостереження до першої успішної відповіді (крок 6):

```text
$ named-start; date +%T; while true; do echo "$(date +%T) $(dig @10.53.0.53 next.cs-202-08.lab +short +tries=1 +timeout=3)"; sleep 2; done
20:23:09
20:23:09 
20:23:11 
...
20:23:39 
20:23:41 10.53.0.81
```

Перевірка після завдання (`dig @10.53.1.1 cs-202-08.lab SOA +norec +short`):

```text
ns.cs-202-08.lab. admin.cs-202-08.lab. 2026102508 3600 600 86400 40
```

### Постмортем

1. **Що сталося:** о 20:21:50 власний авторитативний сервер 10.53.1.1 зупинено командою `named-stop`; о 20:23:09 його запущено знову (`named-start`). Недоступність тривала ≈ 79 с.

2. **Вплив на клієнтів resolver-a і resolver-b:** клієнти resolver-a лишилися без відповіді: перший запит завис на 30 с до тайм-ауту, наступні отримували SERVFAIL. Клієнти resolver-b отримали застарілу, але робочу адресу 10.53.0.81 із затримкою 1,8 с.

3. **Хронологія з виміряним часом:** 20:21:50 зупинка сервера; ≈ 20:22:35 перший запит до resolver-a (після `sleep 45`), тайм-аут через 30 с; 20:23:05 другий запит: SERVFAIL за 20 мс; 20:23:07 resolver-b: NOERROR, TTL 30, 1,8 с; 20:23:09 запуск сервера; 20:23:41 resolver-a знову віддає 10.53.0.81 (через 32 с після запуску).

4. **Чому кеш допоміг або не допоміг:** звичайний кеш не допоміг: запис із TTL 35 с спливла раніше, ніж надійшов запит, а оновити її було ніде. Допоміг лише режим serve-stale в resolver-b, який віддав прострочений запис.

5. **Що слід змінити:** додати другий авторитативний сервер у делегування (другий запис NS на іншому вузлі), налаштувати моніторинг доступності порту 53 і ввімкнути serve-stale на всіх резолверах, а для критичних записів використовувати більший TTL.


1. **Який TTL мав запис у відповіді resolver-b і навіщо такий TTL?**
30 с, а не 35 с із зони. Це TTL, який резолвер сам призначає застарілим відповідям. Він короткий, щоб клієнти скоро повторили запит і отримали свіжі дані, щойно авторитативний сервер знову стане доступним, а застаріла адреса не кешувалася довго. Значення 30 с збігається з типовим для serve-stale (RFC 8767 рекомендує невелике значення порядку 30 с).

2. **Порівняйте результат і тривалість двох запитів до resolver-a.**
Вони різні: перший завершився тайм-аутом клієнта через 30 с без відповіді, другий одразу (0,020 с) повернув SERVFAIL. Гіпотеза: резолвер продовжував опитувати недоступний сервер довше, ніж чекав `dig` (30 с), а коли визнав спробу невдалою, закешував помилку. Другий запит одразу отримав цю закешовану помилку, не звертаючись до сервера.

3. **Чому після `named-start` ім'я через resolver-a зарезолвилося не одразу?**
Сервер запрацював о 20:23:09, а перша адреса з'явилася о 20:23:41 (через 32 с). Увесь цей час цикл отримував порожні відповіді (SERVFAIL). Гіпотеза: резолвер зберігав закешовану невдачу (або вважав сервер недоступним) і не звертався до нього, доки не сплив строк цього запису, приблизно 30–36 с. Резолвер не знає, що сервер знову працює.

4. **У якій ситуації serve-stale шкодить, а не допомагає?**
Коли запис на сервері вже змінено, а резолвер через недоступність сервера далі віддає застарілу адресу. У завданні 4 ми бачили, що резолвер тримає стару адресу до спливання TTL. Serve-stale подовжує цей період і за межі TTL: якщо сервіс перенесено на нову адресу (або старий вузол виведено з експлуатації), клієнти resolver-b продовжуватимуть ходити на стару адресу, доки сервер недоступний.

## Фінальний файл зони

<!-- Повний вміст stand/students/sNN/zone/sNN.lab.zone після завершення роботи. -->

```text
; Zone cs-202-08.lab — starting version.
; After every change: increase the serial, then run  zone-reload
$TTL 45
@     20 IN SOA ns.cs-202-08.lab. admin.cs-202-08.lab. (
            2026102508 ; serial
            3600       ; refresh
            600        ; retry
            86400      ; expire
            40         ; minimum
        )
@     IN NS ns.cs-202-08.lab.
ns    IN A  10.53.1.1

; ---- add records from your variant card below this line ----
www   IN A      10.53.0.81
shop  IN CNAME  www
mail  IN A      10.53.0.80
@     IN MX     15 mail
@     IN TXT    "lab-token=5887307d"
next  35 IN A   10.53.0.81
new-kxzrr IN A 10.53.0.80
```

## Висновки

**Що виявилося несподіваним.** Перший запит до resolver-a після зупинки сервера (завдання 7) завершився не швидкою помилкою, а тайм-аутом на 30 с, а другий запит одразу повернув SERVFAIL:

```text
;; communications error to 10.53.0.53#53: timed out
real    0m30.058s
...
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL
real    0m0.020s
```

Я очікував, що resolver-a відповість помилкою за кілька секунд. Насправді резолвер довго опитував недоступний сервер і лише потім закешував невдачу, тому другий запит отримав її з кешу за 20 мс. Також несподіваною була помилка в зоні: рядки, додані з пробілами на початку, `zone-reload` прочитав як тип запису (`unknown RR type 'www'`), і зона не завантажилась.

**Яке питання залишилося без відповіді.** У завданні 6 (рядок D) повний прохід ланцюжка після перезапуску резолвера зайняв 402 мс (`Query time: 402 msec`, `time_namelookup` 0,4027 с). За моделлю «кількість запитів × затримка сервера» (3 запити до root по ≈ 80 мс, один до TLD ≈ 40 мс, власний сервер ≈ 0 мс) виходить ≈ 280 мс, тобто ≈ 120 мс не пояснено. У журналах root, TLD і власного сервера видно запити 19:51:01.347, .428, .508 (root), .629 (TLD), .669 (власний сервер), але інтервал між третім запитом до root і запитом до TLD дорівнює 121 мс, а ≈ 80 мс перед першим запитом до root у журналах не відображено. Перевірено: журнали трьох серверів і три незалежні вимірювання (402 мс, 402 мс, 0,4027 с). З'ясувати не вдалося, як стенд застосовує штучну затримку (до запиту, до відповіді чи до обох).

## Декларація використання ШІ

Використовувалися інструменти ШІ: так

<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 | Claude Sonnet 5.5 | допоможи виправити формулювання у README. додай, якщо потрібно точності в поясненнях | Пропозиції перевірено і використано для редагування звіту. |
| 2 | Claude Sonnet 5.5 | перевір README. напиши конкретні помилки й потрібні виправлення, не змінюючи результати виконаних команд. | Зауваження перевірено і підтверджені помилки виправлено у звіті. |
