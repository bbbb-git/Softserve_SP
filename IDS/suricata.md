#### 1. Встановлення та запуск Suricata

Suricata встановлена з PPA-репозиторію OISF (`ppa:oisf/suricata-stable`) на Ubuntu 24.04. Перевірка версії та зібраних фіч:

![вивід sudo suricata --build-info: версія 8.0.6, підтримка AF_PACKET, Rust, suricata-update тощо](IDS_photo3.png)
![вивід sudo suricata --build-info: версія 8.0.6, підтримка AF_PACKET, Rust, suricata-update тощо](IDS_photo4.png)

При першому запуску сервіс падав з помилкою — дефолтний конфіг вказував на неіснуючий інтерфейс `eth0`, тоді як на сервері реальний інтерфейс — `enp1s0`.

![вивід journalctl -u suricata: помилка eth0: failed to find interface: No such device, а також подальші події — зупинку і успішний перезапуск після виправлення конфігу](IDS_photo5.png)

Виправлено `/etc/suricata/suricata.yaml`: `HOME_NET` встановлено на `192.168.122.0/24`, інтерфейс у секції `af-packet` змінено на `enp1s0`. Також встановлені правила через `suricata-update`.

![вивід systemctl status suricata: сервіс у стані active (running), потоки запущено](IDS_photo6.png)

#### 2. Зупинка Suricata та додавання власних правил

Сервіс зупинявся командою `systemctl stop suricata` (підтверджено записом `Deactivated successfully` у логах вище, Скрін 5). Створено файл `/var/lib/suricata/rules/local.rules` з двома власними правилами, додано до `rule-files` у конфігурації Suricata:
alert icmp any any -> any any (msg:"ICMP ping detected on host"; sid:1000001; rev:1;)
alert http any any -> any any (msg:"HTTP GET request observed"; flow:to_server,established; http.method; content:"GET"; sid:1000002; rev:1;)
**Правило 1 (sid 1000001)** — тип `alert`, протокол `icmp`, джерело/призначення — будь-які IP і порти (`any any -> any any`). Правило спрацьовує на будь-який ICMP-пакет (наприклад, ping) і генерує повідомлення "ICMP ping detected on host".

**Правило 2 (sid 1000002)** — тип `alert`, протокол `http`. Умова `flow:to_server,established` означає, що пакет має йти від клієнта до сервера у вже встановленому TCP-з'єднанні; `http.method` + `content:"GET"` перевіряють, що це саме GET-запит. Правило генерує повідомлення "HTTP GET request observed".

![вміст local.rules: обидва правила, як показано вище](IDS_photo7.png)

#### 3. Запуск Suricata та тестування правил

Перед запуском конфігурація перевірена в тестовому режимі:

![вивід suricata -T -c /etc/suricata/suricata.yaml -v: "2 rule files processed, 52151 rules successfully loaded, 0 rules failed"](IDS_photo8.png)

Після успішного запуску (Скрін 6 вище) правила протестовані генерацією реального трафіку:

![фрагмент fast.log з двома спрацюваннями правила ICMP на запит і на відповідь ping](IDS_photo1.png)

![два термінали одночасно: зліва виконуються ping -c 3 8.8.8.8 та curl http://testmynids.org/uid/index.html, справа — tail -f fast.log зі спрацюваннями обох правил](IDS_photo2.png)

**Висновок:** обидва кастомних правила успішно завантажені та коректно спрацьовують на відповідний трафік, що підтверджує працездатність налаштованої Suricata IDS.
