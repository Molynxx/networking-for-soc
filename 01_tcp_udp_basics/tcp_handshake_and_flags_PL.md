# tcp_handshake_and_flags

## Cel
Zrozumienie jak działa protokół sieciowy TCP oraz w jaki sposób nawiązuje sesję i dlaczego.

## Czym jest TCP
Jest to protokół warstwy transportowej, który realizuje transmisję danych. Jest bardzo skuteczny bo zapewnia, że dane dotrą do odbiorcy bezbłędnie, w całości i w odpowiedniej kolejności. Dzięki temu, że TCP ustanawia sesję może także kontrolować prędkość transmisji, tak aby była dostosowana do możliwości odbioru odbiorcy. Protokół ten upewnia się, za pomocą sum kontrolnych, czy wszystkie dane dotarły do celu, a jeśli nie -> wysyła do skutku, aż dostanie potwierdzenie, że wszystko zostało poprawnie dostarczone. Protokół ten nie ma własnego portu bo jest mechanizmem używanym przez inne protokoły np.:
- HTTP - port 80,
- HTTPS - port 443, 
- SMTP - port 25 /587 / 465,
- FTP - port 21,
- SSH - port 22. 

## Zadania protokołu TCP
Protokół wykonuje następujące zadania:
- nawiązuje połączenie ustanawiając sesję (three-way handshake),
- dzieli dane na segmenty,
- numeruje każdy segment, żeby dane po stronie odbiorcy mogły zostać poskładane w odpowiedniej kolejności, 
- sprawdza, czy wszystkie dane dotarły, a jeśli coś zgonie, żąda ponownego wysłania,
- gwarantuje dostarczenie danych do odbiorcy. 

## Three-Way Handshake
Protokół TCP zanim zacznie wysyłać dane nawiązuje połączenie, trzy wiadomości na początek -> Three-Way Handshake:
- krok 1: `SYN`
	- klient mówi: "Cześć chcę nawiązać połączenie. Będę numerował swoje pakiety od numeru X.",
	- technicznie: klient wysyła segment z flagą `SYN` i swoim numerem startowym, 
- krok 2: `SYN-ACK`
	- serwer odpowiada: "OK, przyjąłem. Ja zaczynam numerację od Y. Potwierdzam, że dostałem twój X - czekam na X+1."
	- technicznie: serwer wysyła segment z flagami `SYN` i `ACK`, swoim numerem startowym Y i potwierdzeniem X+1,
- krok 3: `ACK`  
	- klient potwierdza: "potwierdzam Y - czekam na Y+1. Połączenie otwarte."
	- technicznie: klient wysyła segment z flagą `ACK` i potwierdzeniem Y+1.   
Od tego momentu dopiero następuje przepływ danych. 

## Po co te numery X i Y
Te numery służą do:
- sprawdzenia, czy wszystko doszło, 
- ułożenia pakietów w odpowiedniej kolejności, 
- wykrycia duplikatów.   
Jeśli coś zginie - odbiorca mówi: "Hej brakuje 105, wyślij jeszcze raz".

## Flagi TCP
Każdy segment TCP ma flagi - jednobitowe znaczniki, jest ich 6:
- `SYN` - [S] ta flaga występuje tylko raz, na samym początku. Oznacza "chcę nawiązać połączenie", 
- `ACK` - [.] występuje regularnie po każdy segmencie i oznacza "potwierdzam odbiór",
- `FIN` - [F] ta flaga również występuje w trakcie połączenia tylko raz i oznacza "kończę połączenie", 
- `RST` - [R] występuje tylko w przypadku kiedy zachodzi potrzeba nagłego zakończenia połączenia, (firewall blokuje połączenie, port jest zamknięty, błąd aplikacji) - "zrywam połączenie",
- `PSH` - [P] informuje, że odbiorca powinien przekazać dane od razu do warstwy wyższej, bez buforowania - "wyślij natychmiast nie czekaj",
- `URG` - [U] rzadko używane, oznacza, że dane mają większy priorytet "pilne." 

## Jak wygląda Handshake w tcpdump
```
IP klient.12345 > serwer.22: Flags [S], seq 100
IP serwer.22 > klient.12345: Flags [S.], seq 500, ack 101
IP klient.12345 > serwer.22: Flags [.], ack 501
```
Widać tutaj wyraźnie, "rozmowę" klienta z serwerem przez port 22, a więc połączenie zostało uzgodnione przez SSH. Klient wysłał `SYN` i numer segmentu, serwer odpowiedział `SYN-ACK` [S.] i podał swój nr pierwszego segmentu oraz nr kolejnego segmentu dla klienta, następnie klient potwierdził `ACK` podając kolejny numer segmentu serwera. 

## Zagrożenia i jak się przed nimi chronić 

### SYN Flood
- co to jest: To atak, w którym atakujący wysyła masę żądań SYN (często z fałszywych IP). Serwer odpowiada `SYN-ACK` na każde połączenie i rezerwuje dla niego zasoby, oczekując ACK. Ponieważ jednak `ACK` nigdy nie nadchodzi, czeka w zawieszenie. Przy dużej ilości takich żądań, pula połączeń się wyczerpuje a inni użytkownicy nie mają możliwości połączenia (DoS).
- jak wykryć: 
	- w `tcpdump` / Wireshark: tysiące [S] bez odpowiadających [S.], 
	- `ss -s` - pokazuje statystyki TCP (dużo `SYN-RECV`),
	- monitoring ruchu: jeden IP wysyła setki SYN na sekundę, 
- jak się chronić:
	- `SYN cookies` - serwer nie rezerwuje zasób przy SYN, tylko koduje informację w numerze sekwencyjnym (adres IP klienta, port, znacznik czasu), który odsyła w `SYN-ACK`. Gdy dostaje `ACK` od klienta, sprawdza ten numer i jeśli jest poprawny, klient jest prawdziwy i dopiero wtedy rezerwuje zasoby. Można to włączyć za pomocą polecenia (Linux): 
		- `sysctl -w net.ipv4.tcp_syncookies=1`
	- `rate limiting` w firewall - czyli ograniczenie liczby `SYN` z jednego IP do np. 25 na sekundę. Reszta jest odrzucana. Ograniczenie to można wprowadzić za pomocą poleceń:
		- `iptables -A INPUT -p tcp --syn -m limit --limit 25/s --limit-burst 50 -j ACCEPT`,
		- `iptables -A INPUT -p tcp --syn -j DROP`,   
		Co to oznacza: 
			- `iptables` - narzędzie do konfiguracji firewalla w Linux,
			- `-A INPUT` - dodaj regułę do łańcucha INPUT (ruch przychodzący do tego komputera),
			- `-p tcp` - dotyczy tylko protokołu TCP,
			- `--syn` - dotyczy tylko pakietów z flagą `SYN` (nowe połączenia),
			- `-m limit`- użyj modułu 'limit' - to on ogranicza częstotliwość,
			- `--limit 25/s` - przepuszczaj max 25 `SYN` na sekundę,
			- `--limit-burst 50` - pozwól na chwilowy wybuch 50 SYN (np. przy starcie aplikacji),
			- `-j ACCEPT` - przepuść tylko to co mieści się w limicie, 
			- `-j DROP` - wszystkie, które dotarły do tej linijki po prostu odrzuć bez odpowiedzi. 	
	- `SYN proxy` - firewall przejmuje handshake i dopiero po udanym `ACK` przekazuje do serwera. Jak to działa:
		- klient -> `SYN` -> Firewall (zamiast serwera),
		- firewall -> `SYN/ACK` -> klient,
		- klient -> `ACK` -> firewall
		- dopiero po tych czynnościach firewall otwiera połączenie z serwerem i przekazuje ruch. To pozwala serwerowi nie widzieć fałszywych `SYN` bo firewall przekazuje tylko prawdziwe (potwierdzone). 
- jak zareagować: 
	- zablokować IP atakującego na firewall (`iptables -A INPUT -s IP -j DROP`),
	- wprowadzić zabezpieczenia SYN Cookies, Rate Limit, SYN proxy.

### TCP reset attack
- co to jest: atakujący podsłuchuje (MitM) istniejące połączenie i wysyła fałszywy segment z flagą `RST`. Serwer zrywa połączenie. 
- jak wykryć:
	- w `tcpdump` pojawia się nagłe [R] w trakcie trwającej sesji,
	- `RST` wysłane od IP, które nie brało udziału w połączeniu, 
-jak się chronić:
	- szyfrowanie (TLS, SSH) - atakujący nie zna numerów sekwencyjnych nie może więc skutecznie wysłać RST. Należy pamiętać ze TCP nie szyfruje danych, odpowiada za to protokół, który korzysta z TCP. Przykładowo: HTTP jest protokołem, w którym dane przesyłane są jawnie, natomiast HTTPS szyfruje dane, 
	- uwierzytelnianie segmentów TCP (TCP-AO) - rzadko stosowane. To mechanizm, który podpisuje kryptograficznie każdy segment TCP. Odbiorca sprawdza podpis i jeśli się on nie zgadza, segment jest odrzucany. To uniemożliwia atakującemu wysłanie fałszywego `RST` i wstrzyknięcie danych do sesji (hijacking). Jest rzadko stosowane, ponieważ to rozwiązanie wymaga, żeby obie strony (klient i serwer) miały ten sam klucz. To trudne w dużych sieciach, dlatego powszechnie używa się po prostu TLS/SSH - to szyfruje dane i rozwiązuje ten sam problem w prostszy sposób. 
- jak reagować:
	- niezwłocznie zablokować podejrzane IP atakującego na firewall (`iptables -A INPUT -s IP -j DROP`),
	- odizolować źródło MitM w sieci - jeśli to skompromitowane w sieci odłączyć go od sieci i sprawdzić do kogo należy (MAC, IP, ewidencja IT), jeśli to obce urządzenie połączone przez Wi-Fi zmienić hasło do Wi-Fi, 
	- w przypadki skompromitowanego hosta wewnętrznego, sprawdzić czy przechwycone połączenie było szyfrowane, jeśli nie, należy uznać tokeny sesyjne i dane logowania za skompromitowane, 
	- unieważnić sesję, zmienić hasła użytkowników, których sesje mogły zostać naruszone, 
	- wdrożyć szyfrowanie (HTTPS, TLS dla SMTP, itd).

### TCP session hijacking
- co to jest: atakujący przejmuje istniejącą sesję TCP (MitM), zgadując numery sekwencyjne. Może w ten sposób wstrzyknąć własne dane w sesję ofiary. 
- jak wykryć:
	- duplikaty `ACK`, niespodziewane zmiany w strumieniu danych. W `tcpdump` /Wireshark normalny ruch wygląda tak:
	```
		- ack 501
		- ack 601
		- ack 701
	```
	Jeśli jednak nagle pojawiają się takie wpisy:
	```
		- ack 501
		- ack 501
		- ack 501   -> duplikaty ACK, oznaczają, że ktoś próbuje coś wstrzyknąć
		- ack 999   -> przeskoczyło o 300, oznacza pytanie skąd taki numer
		```
	- pakiety z niepasującymi numerami sekwencyjnymi, 
- jak się chronić: 
	- szyfrowanie (TLS, SSH) uniemożliwia wstrzyknięcie sensownych danych, 
	- losowe numery sekwencyjne (współczesne systemy to robią). Kiedyś numery sekwencyjne były przewidywalne (np. rosły o 1). Atakujący mógł zgadnąć numer i wstrzyknąć fałszywy pakiet. Dlatego współczesne systemy operacyjne, Linux i Windows, generują całkowicie losowe początkowe numery sekwencyjne. Nie da się ich zgadnąć. To uniemożliwia atakującemu wstrzyknięcie sensownego pakietu, nawet jeśli podsłuchuje ruch (MitM).  
- jak reagować:
	- niezwłocznie zablokować IP atakującego na firewall (`iptables -A INPUT -s IP -j DROP`), 
	- zakończyć przejętą sesję (restart usługi, unieważnienie tokenów),
	- sprawdzić, jakie dane mogły zostać wstrzyknięte lub wykradzione podczas przejęcia sesji, 
	- wdrożyć szyfrowanie (TLS/SSH) dla chronionej usługi, 
	- przeanalizować, w jaki sposób atakujący uzyskał pozycję MitM (ARP spoofing, rouge ARP, skompromitowany host) i usunąć przyczynę, 
	- jeśli źródłem ataku jest skompromitowany host wewnętrzny - odizolować go i przeprowadzić analizę powłamaniową 

## Case study

Sytuacja: Serwer WWW firmy przestał odpowiadać. Użytkownicy zgłaszają, że strona się nie ładuje. Administrator uruchomił tcpdump na 10 sekund i dostał następujący wynik:
```
12:00:01.100 IP 45.33.22.11.54321 > serwer.80: Flags [S], seq 1000
12:00:01.101 IP 45.33.22.11.54322 > serwer.80: Flags [S], seq 2000
12:00:01.101 IP 45.33.22.11.54323 > serwer.80: Flags [S], seq 3000
12:00:01.102 IP 45.33.22.11.54324 > serwer.80: Flags [S], seq 4000
12:00:01.102 IP 45.33.22.11.54325 > serwer.80: Flags [S], seq 5000
.... setki podobnych linijek z tego samego IP ....
12:00:05.200 IP 192.168.1.50.12345 > serwer.80: Flags [S], seq 6000
12:00:06.200 IP 192.168.1.50.12345 > serwer.80: Flags [S], seq 7000
12:00:07.200 IP 192.168.1.50.12345 > serwer.80: Flags [S], seq 8000
```
Analiza:
- w pliku widać wiele zapytań `SYN`, ale nie widać aby przyszła akceptacja `ACK`, to wskazuje na atak SYN Flood, w którym atakujący wysyła tysiące próśb `SYN` do serwera lecz nie odpowiada `ACK`. Ten atak powoduje wyczerpanie zasobów serwera, poprzez zawieszone prośby, na które nie ma odpowiedzi od hosta. Atak nastąpił z zewnętrznego IP 45.33.22.11. 
- prośby `SYN` przychodzące z lokalnego IP nie powiodły się z powodu wyczerpania zasobów serwera z powodu powyższego ataku, 
- aby zatrzymać atak, należy odciąć źródło ataku, czyli zablokować adres IP atakującego na firewall za pomocą polecenia: `iptables -A INPUT -s 45.33.22.11 -j DROP`,
- aby zabezpieczyć się przed przyszłymi atakami tego typu należy użyć:
	- SYN Cookies - serwer nie będzie rezerwować zasobów, tylko zapisze informacje o prośbie w numerze sekwencyjnym, który wysyła w `SYN-ACK`, a dopiero po nadejściu `ACK` weryfikuje czy numer się zgadza i jeśli tak serwer rezerwuje zasoby, 
	- Rate limiting na firewall, polega na ograniczeniu próśb `SYN` z jednego IP na sekundę, co utrudnia przeprowadzenie ataku z jednego IP, 
	- SYN proxy - firewall przejmuje proces handshake i dopiero po otrzymaniu od klienta `ACK` przekazuje ruch do serwera. Dzięki temu serwer nie widzi fałszywych `SYN`.