# dns_log_analysis

## Cel
Nauka czytania logów DNS celem zrozumienia jak wykryć podejrzany ruch. 

## Case study 1

- fragment logów:
```
08:00:01 192.168.1.105 -> resolver: A? www.google.com  
08:00:02 resolver -> 192.168.1.105: www.google.com A 142.250.203.206  

08:04:10 192.168.1.105 -> resolver: A? aGFzwolJv.evil.com  
08:04:11 resolver -> 192.168.1.105: aGFzwolJv.evil.com A 45.67.89.100  

08:09:15 192.168.1.105 -> resolver: A? cGFzc3dvcmQ.evil.com  
08:09:16 resolver -> 192.168.1.105: cGFzc3dvcmQ.evil.com A 45.67.89.100 

08:14:20 192.168.1.105 -> resolver: A? dXNlcjphZG1pbg.evil.com  
08:14:21 resolver -> 192.168.1.105: dXNlcjphZG1pbg.evil.com A 45.67.89.100 

08:19:25 192.168.1.105 -> resolver: A? eW91cGFzc3dvcmQ.evil.com  
08:19:26 resolver -> 192.168.1.105: eW91cGFzc3dvcmQ A 45.67.89.100 
```
Analiza:  
- co jest podejrzane: 
	- zapytania z jednego hosta o jedną domenę w regularnych odstępach czasu, 
	- nazwa subdomeny bardzo przypomina kodowanie base64, 
- na co to wskazuje: atak DNS tunnelling, którego celem jest eksfiltracja. Urządzenie 192.168.1.105 zostało zainfekowane, malware wysyła do resolvera atakującego fragmenty zaszyfrowanych za pomocą base64 danych. 
- co należy zrobić:
	- odciąć zainfekowane urządzenie od sieci, 
	- zablokować domenę na resolverze oraz IP na firewallu, 
	- sprawdzić IP/domenę w threat intelligence, 
	- usunąć malware z zainfekowanego komputera, 
	- sprawdzić co dokładnie wyciekło - analiza base64, zdekodować subdomeny, 
	- ustalić od kiedy trwa atak, 
	- sprawdzić czy inne komputery w sieci nie łączą się z domeną evil.com

## Case study 2

- fragment logów
```
09:02:11 192.168.1.35 -> resolver: A? a8f3b.example.com  
09:02:12 resolver -> 192.168.1.35: NXDOMAIN

09:03:05 192.168.1.35 -> resolver: A? b9g4c.example.com  
09:03:06 resolver -> 192.168.1.35: NXDOMAIN

09:04:11 192.168.1.35 -> resolver: A? c2d4e.example.com  
09:04:12 resolver -> 192.168.1.35: NXDOMAIN

09:05:07 192.168.1.35 -> resolver: A? d3e5f.example.com  
09:05:08 resolver -> 192.168.1.35: NXDOMAIN

09:06:11 192.168.1.35 -> resolver: A? e4a9c.example.com  
09:06:12 resolver -> 192.168.1.35: e4a9c.example.com A 185.30.120.10
```
Analiza:  
- co jest podejrzane: wiele zapytań o nieistniejące domeny z jednego hosta w krótkich odstępach czasu oraz końcowe powodzenie, 
- na co to wskazuje: atak DGA - malware generuje wiele domen, atakujący zna ich listę i każdego dnia rejestruje tylko jedną z nich. Malware odpytuje kolejno domeny aż trafi na tę zarejestrowaną. Atak ma na celu utrzymanie komunikacji C2, bo nawet jeśli jednego dnia domena zostanie zablokowana - to drugiego dnia C2 jest już na innej domenie. 
- co należy zrobić: 
	- odciąć zainfekowany host od sieci,
	- zablokować IP i domenę, 
	- sprawdzić IP i domenę w threat intelligence, 
	- usunąć malware, 
	- sprawdzić od kiedy trwa atak
	- sprawdzić logi pod kątem eksfiltracji, jeśli nastąpiła to jakie dane wyciekły, 
	- sprawdzić inne hosty w sieci, czy któryś z nich nie generuje takich samych zapytań, 
	- ustalić, do czego malware zdążył uzyskać dostęp, jeśli połączył się z C2 mógł pobrać dodatkowe moduły. 

## Case study 3
- fragment logu:
```
11:00:00 10.0.0.7 -> resolver: A? evil.com
11:00:01 resolver -> 10.0.0.7: evil.com A 203.0.113.10 (TTL 60)

11:01:10 10.0.0.7 -> resolver: A? evil.com
11:01:11 resolver -> 10.0.0.7: evil.com A 203.0.113.55 (TTL 60)

11:02:20 10.0.0.7 -> resolver: A? evil.com
11:02:21 resolver -> 10.0.0.7: evil.com A 203.0.113.88 (TTL 60)

11:03:30 10.0.0.7 -> resolver: A? evil.com
11:03:31 resolver -> 10.0.0.7: evil.com A 203.0.113.21 (TTL 60)

11:04:40 10.0.0.7 -> resolver: A? evil.com
11:04:41 resolver -> 10.0.0.7: evil.com A 203.0.113.74 (TTL 60)
```
Analiza:  
- co jest podejrzane: jeden host pyta o tę samą domenę w odstępie 70 sekund, domena, o którą pyta ma inny adres IP przy każdej kolejnej odpowiedzi, TTL 60 to czerwona flaga - DNS przechowuje pamięć o domenie w cache tylko przez minutę, 
- na co to wskazuje: atak Fast Flux celem utrzymania komunikacji C2 - atakujący posiada sieć zainfekowanych komputerów oraz jedną złośliwą domenę, na której znajduje się serwer C2. Co minutę domena jest podpinana na kolejne IP z botnetu atakującego. Dzięki temu nawet jeśli jeden adres IP zostanie zablokowany to ruch w kolejnej minucie idzie już przez inne IP. 
- co należy zrobić: 
	- odciąć zainfekowane urządzenie od sieci, 
	- zablokować domenę i IP, 
	- sprawdzić domenę i IP w threat intelligence, 
	- usunąć malware, 
	- sprawdzić logi pod kątem eksfiltracji i jeśli istnieje, ustalić jakie dane wyciekły i do czego malware miało dostęp,
	- sprawdzić czy inne hosty nie generują takiego samego rodzaju ruchu. 

## Case study 4

- fragment logów:
```
13:00:00 172.16.0.20 -> resolver: ANY? firma.pl
13:00:01 resolver -> 172.16.0.20: odpowiedź ANY (duża >4000 bajtów)

13:00:05 172.16.0.20 -> resolver: ANY? firma.pl
13:00:06 resolver -> 172.16.0.20: odpowiedź ANY (duża >4000 bajtów)

13:00:10 172.16.0.20 -> resolver: ANY? firma.pl
13:00:11 resolver -> 172.16.0.20: odpowiedź ANY (duża >4000 bajtów)

... powtarza się wielokrotnie z różną częstotliwością ....
```
Analiza:  
- co jest podejrzane: jeden host wysyła wiele zapytań ANY (zapytanie o wszystkie rekordy - duża odpowiedź) z dużą częstotliwością. Rekord ANY nie jest używany w normalnych ruchu właśnie ze względu na wielkość odpowiedzi, zwykły użytkownik nie potrzebuje takich informacji. Samo pojawienie się tego rekordu w logach to czerwona flaga dla SOC, 
- na co to wskazuje: DNS amplification - atakujący wysyła zapytanie ANY fałszując adres źródłowy, więc duża odpowiedź idzie do ofiary a nie do atakującego. Celem tego ataku jest zablokowanie ruchu na komputerze ofiary lub usługi online - DDoS. 
- co należy zrobić:
	- jeśli to komputer w sieci wewnętrznej - odciąć go od sieci. Należy jednak pamiętać, że resolver może zostać użyty do ataku na zewnętrznych adresach IP, 
	- sprawdzić czy na resolverze jest ustawiony ACL, 
	- sprawdzić czy resolver ma wyłączoną rekurencję dla zewnętrznych hostów, 
	- ustawić rate limiting (ograniczyć ilość zapytań z jednego hosta w określonym oknie czasowym), 
	- zablokować możliwość wysyłania zapytań ANY w konfiguracji DNS.

## Case study 5
- fragment logów:
```
07:30:00 192.168.1.50 -> resolver: TXT? evil.com
07:30:01 resolver -> 192.168.1.50: TXT odpowiedź: "b3BlbmFpZA=="

07:35:00 192.168.1.50 -> resolver: TXT? evil.com
07:35:01 resolver -> 192.168.1.50: TXT odpowiedź: "c3RvcA=="

07:40:00 192.168.1.50 -> resolver: TXT? evil.com
07:40:01 resolver -> 192.168.1.50: TXT odpowiedź: "d3l5bGlq"
```
Analiza:  
- co jest podejrzane: jeden host co 5 minut odpytuje o rekord TXT tą samą domenę, a odpowiedź przychodzi w postaci danych zaszyfrowanych za pomocą base64. 
- na co to wskazuje: DNS tunnelling, którego celem jest komunikacja C2. Host pyta domenę o rekord TXT, który może przechowywać dowolne dane. W tym przypadku TXT zawiera krótkie, zaszyfrowane komunikaty przeznaczone dla malware od atakującego. W taki sposób atakujący może sterować malware na urządzeniu (wydawać mu polecenia np. stop, wyślij, itp). 
- co należy zrobić: 
	- odciąć zainfekowane urządzenie od sieci, 
	- zablokować domenę, 
	- sprawdzić domenę w threat intelligence, 
	- odkodować odpowiedzi TXT, by dowiedzieć się jakie polecenia wykonało malware, 
	- sprawdzić logi pod kątem eksfiltracji, a jeśli wystąpiła ustalić co wyciekło, 
	- sprawdzić od kiedy trwa atak, 
	- sprawdzić czy inne hosty z sieci nie mają podobnych zdarzeń w logach. 

## Case study 6
- fragment logów:
```
09:10:00 resolver -> 192.168.1.77: odpowiedź A dla bank.pl = 203.0.113.200
09:10:05 resolver -> 192.168.1.77: odpowiedź A dla bank.pl = 203.0.113.200

09:15:00 192.168.1.77 -> resolver: A? bank.pl
09:10:05 resolver -> 192.168.1.77: A bank.pl = 198.51.100.150

09:20:00 192.168.1.77 -> resolver: A? bank.pl
09:20:05 resolver -> 192.168.1.77: A bank.pl = 198.51.100.150  
```
Analiza:  
- co jest podejrzane: adres domeny bank.pl zmienił się nagle. 
- na co to wskazuje: Cache poisoning - atakujący (MitM) zobaczył zapytanie od IP bank.pl, wysłał odpowiedź ze sfałszowanym adresem IP zanim nadeszła prawdziwa odpowiedź. Resolver ją przyjął i zapisał w cache na czas TTL określony w fałszywej odpowiedzi. Zwykle w takich przypadkach TTL jest długi, a to powinno natychmiast budzić podejrzenia. Przez czas TTL resolver będzie odpowiadał każdemu hostowi, który zapyta o bank.pl z cache podając fałszywy adres (złośliwego serwera). 
- co należy zrobić: 
	- niezwłocznie zablokować IP,
	- wyczyścić cache DNS lub zrestartować usługę, 
	- sprawdzić czy jest włączone 
		- DNSSEC, 
		- losowanie query ID i portów (jeśli reslover jest stary, nowoczesne mają to ustawione domyślnie), 
		- ACL - czy jest ustawione, 
		- jaki czas TTL jest ustawiony dla tej domeny,
	- sprawdzić czy inne hosty dostały fałszywy adres domeny, czy któryś łączył się z tym IP, 
	- pamiętać o aktualizacji resolvera. 

## Case study 7
- fragment logów:
```
12:00:00 10.0.0.15 -> resolver: MX? firma.pl
12:00:01 resolver -> 10.0.0.15: MX firma.pl = mail.firma.pl

12:00:30 10.0.0.15 -> resolver: SOA? firma.pl
12:00:31 resolver -> 10.0.0.15: SOA firma.pl = ns1.firma.pl admin@firma.pl

12:01:00 10.0.0.15 -> resolver: SRV? _ldap._tcp.firma.pl
12:01:01 resolver -> 10.0.0.15: SRV _ldap._tcp.firma.pl = dc1.firma.pl:389

12:00:00 10.0.0.15 -> resolver: AXFR? firma.pl
12:00:01 resolver -> 10.0.0.15: REFUSED
```
Analiza:  
- co jest podejrzane: wszystkie rekordy ujęte w logu są podejrzane:
	- MX - zapytanie o serwer poczty, zwykłe hosty nigdy o to nie pytają, pyta o to serwer pocztowy, 
	- SOA - zwykle nikomu taka informacja nie jest potrzebna, jeśli więc widnieje w logach jest to podejrzane, 
	- SRV - zapytanie o identyfikator domeny, to również nie jest zapytanie zwykłego użytkownika. 
	- AXFR - ktoś próbował pobrać całą strefę - próba zakończyła się niepowodzeniem, ponieważ ACL blokuje opcję pobierania całej strefy dla serwerów innych niż zaufane, jednak ten rekord w logach zawsze jest sygnałem, że ktoś grzebie, 
- na co to wskazuje: DNS reconnaissance - to atak polegający na rozpoznaniu, atakujący w ten sposób może przygotować się przed dalszym atakiem (np. phishing), 
- co należy zrobić:
	- zablokować IP, 
	- sprawdzić IP w threat intelligence,
	- sprawdzić ACL pod kątem zaufanych hostów i serwerów, 
	- usunąć adres admina z rekordu SOA (jeśli został tam wpisany), 
	- sprawdzić czy IP nie występowało już wcześniej w logach. 
