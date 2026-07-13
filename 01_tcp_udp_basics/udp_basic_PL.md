# udp_basic

## Cel
Zrozumienie jak działa protokół UDP, czym się różni od TCP oraz jakie zagrożenia wynikają z jego używania. 

## Czym jest UDP
UDP to młodszy, szybszy lecz mniej troskliwy o dane brat TCP. Nie nawiązuje połączenia, nie ma tu handshake, nie numeruje, nie potwierdza odbioru, nie ponawia gdy coś się zagubiło. Ten protokół po prostu wysyła dane, jest bezpołączeniowy i bezstanowy - jego "motto" to "wyślij i zapomnij". Stosowany często w grach, transmisji audio i video, rozmowy głosowe VoIP, szybkich zapytaniach DNS oraz w DHCP. 

## Różnice między UDP a TCP 

| TCP | UDP|
|-----|----|
| nawiązuje połączenie (handshake) | nie nawiązuje połączenia, od razu wysyła |
| numeruje pakiety | nie numeruje pakietów |
| sprawdza, czy wszystko dotarło | nic nie sprawdza |
| jak coś zginie -> wysyła ponownie | jeśli zaginęło - trudno, nie wysyła |
| wolniejszy, ale pewny | szybki ale bez gwarancji dostarczenia |

## Jak działa UDP 
- aplikacja chce wysłać dane, 
- UDP pakuje dane w datagram, 
- dodaje port źródłowy i docelowy, 
- wysyła, 
- koniec. Żadnego potwierdzenia, żadnego sprawdzenia, żadnego układania w kolejności.   
Jeśli datagram zaginie - aplikacja albo sobie poradzi (np. poprosi o powtórzenie na poziomie aplikacji), albo nie (VoIP - chwilowa cisza, gramy dalej).

## Kto używa UDP i dlaczego 
| Protokół | Port | Dlaczego UDP |
| DNS | 53 | krótkie zapytania, nie marnuje czasu na handshake |
| DHCP | 67, 68 | klient nie ma jeszcze swojego IP więc nie może użyć TCP |
| NTP | 123 | szybka synchronizacja czasu | 
| VoIP (SIP/RTP) | różne | rozmowa na żywo, tu liczy się szybkość niż każdy pakiet |
| Streaming video | różne | jeden zagubiony piksel, nie wpływa na przekaz |
| Gry online | różne | szybkość to podstawa, jest ważniejsza niż 100% danych| 

## Zagrożenia związane z UDP 

### UDP flood
- co to jest: atakujący wysyła dużą ilość fałszywych pakietów UDP na losowe porty ofiary. Ofiara sprawdza każdy port czy działa na nim jakaś usługa, która oczekuje danych, a ponieważ porty są losowe, w większości przypadków na danym porcie nie działa żadna usługa. System więc odpowiada atakującemu komunikatem ICMP "Destination Unreachable" (cel nieosiągalny). Ofiara musi sprawdzić każdy port, wygenerować miliony odpowiedzi ICMP, to prowadzi do zapchania łączą ogromną ilością ruchu.
- jak wykryć: można poznać po nagłym, dużym wzroście ruchu UDP, szczególnie jeśli odbywa się on na wielu różnych portach docelowych, 
- jak się chronić: 
	- blokada podejrzanego IP,
	- Rate limiting na firewallu, Ustalając rate limit na firewallu należy precyzyjnie określić port, dla którego są wprowadzane limity. Przykładowa reguła:
	```
	iptables -A INPUT -p udp -m hashlimit\
		--hashlimit-name UDP_LIMIT \
		--hashlimit-above 50/second \
		--hashlimit-burst 100 \
		--hashlimit-mode srcip \
		-j DROP
	```
	Ta reguła wprowadza ograniczenia dla portu 53, na którym działa DNS. Wyjaśnienie reguły:
	- `-A INPUT` - dodaj regułę do ruchu przychodzącego, 
	- `-p udp` - dotyczy tylko pakietów UDP,
	- `-m hashlimit` - użyj modułu hashlimit (bardziej precyzyjny niż limit),
	- `--hashlimit-name UDP_LIMIT` - nazwa każdej reguły musi być unikalna, 
	- `--hashlimit-above 50` - jeśli pakietów z danego IP jest więcej niż 50 na sekundę. 
	- `--hashlimit-burst 100` - dozwolony jest skok do 100 pakietów (np. na starcie zapytania),
	- `--hashlimit-mode srcip` - limit liczony dla każdego adresu źródłowego z osobna, 
	- `-j DROP` - odrzuć nadmiarowe pakiety. 
- jak reagować: 
	- niezwłocznie zablokować na firewall IP atakującego, 
	- wprowadzić rate limiting dla UDP ogółem. 

### DNS amplification (wzmocnienie DNS)
- co to jest: to atak, w którym atakujący wykorzystuje IP spoofing (podszywanie się pod IP ofiary). Atakujący wysyła małe zapytanie do DNS, wpisując w polu adres źródłowy adres IP ofiary. Choć zapytanie jest małe, odpowiedz może być ogromna, np gdy atakujący wysyła zapytanie o listę wszystkich rekordów dla danej domeny. Odpowiedź może mieć nawet kilka tysięcy bajtów. W wyniku takiego ataku ofiara jest zalewana ogromną ilością danych, co zapycha jej łącze internetowe i powoduje problemy z połączeniem. 
- dlaczego UDP: ponieważ UDP nie ma handshake, więc można sfałszować adres źródłowy (IP spoofing). TCP wymaga handshake więc taki atak nie jest możliwy, 
- jak wykryć: duży ruch DNS z jednego źródła, odpowiedzi DNS bez odpowiadających im zapytań z sieci wewnętrznej. 
- jak się chronić: 
	- ograniczenia rekurencji DNS (dotyczy firmowych DNS). Sednem ograniczenia rekurencji DNS jest by serwer odpowiadał wyłącznie na zapytania od zaufanych źródeł (np. z wewnętrznej sieci). Czyli: 
		- serwer DNS może odpowiadać tylko na zapytania od każdego (np. o domenę), 
		- jednak jeśli pytanie wymaga rekurencji (w sytuacji gdy serwer musi poszukać odpowiedzi na innych serwerach), to odpowiada wyłącznie zaufanym źródłom. Jeśli zapytanie do serwera wysyła ktoś spoza zaufanego źródła serwer odpowiada tylko na proste zapytania (np. o domeny, które ma już w swojej podręcznej pamięci), inne zapytania ignoruje.  
	Takie ograniczenie rekurencji jest częścią konfiguracji samego serwera DNS, który ma wbudowane mechanizmy pozwalające mu decydować, skąd akceptuje zapytania. Sposób konfiguracji zależy od oprogramowania serwera. 
	- rate limiting - ograniczenie liczby zapytań DNS z jednego adresu IP w jednostce czasu celem uniemożliwienia ataku DNS amplification. Polega na ustawieniu odpowiedniej reguły na firewallu dotyczącej portu 53 na którym działa DNS. Przykładowe polecenie:  
	```
	iptables -A INPUT -p udp --dport 53 -m hashlimit \
		--hashlimit-name DNS_LIMIT \
		--hashlimit-above 10/second \
		--hashlimit-burst 20 \
		--hashlimit-mode srcip \
		-j DROP
		```
		To ograniczenie uniemożliwia wysyłania wielu zapytań w krótkim czasie z jednego IP, uniemożliwiając atak DNS amplification. 
- jak reagować: 
	- zablokować IP atakującego na firewall (`iptables -A INPUT -s IP -j DROP`), lub odizolować zaatakowane urządzenie, 
	- następnie należy ustawić rate limiting oraz ograniczenie rekurencji DNS.

### NTP amplification
- co to jest: jest to atak typu DDoS, wykorzystujący serwery NTP (Network Time Protocol) do wzmacniania ruchu. Atakujący wysyła małe zapytanie do serwera NTP, podszywając się pod adres IP ofiary (IP spoofing), a serwer odpowiada dużym pakietem (np. zwraca listę ostatnich 600 klientów). Jest chętnie wykorzystywany przez atakujących ponieważ współczynnik amplifikacji (wzmocnienia) może być nawet kilkaset razy większy  niż w przypadku DNS. Efekt działania ataku jest taki sam jak DNS amplification -> zapchane łącze.
- jak się chronić:
	- wyłączenie polecenia `monlist` - to stare polecenie zwracające listę ostatnich kilkuset adresów IP, które się z nim łączyły. Obecnie zespoły Blue Team korzystają z nowszego i bezpieczniejszego polecenia `mru`, którego nie da się wykorzystać w ataku,
	- ograniczenie dostępu do serwera - czyli ustawienie by serwer odpowiadał tylko na zapytania od zaufanych sieci, a nie z całego internetu. Ograniczenie to można wprowadzić za pomoca list kontroli dostępu (ACL) w pliku konfiguracyjnym ntp.conf, 
	- rate limiting. Przykład:
	```
	iptables -A INPUT -p udp --dport 123 -m hashlimit \
		--hashlimit-name NTP_LIMIT \
		--hashlimit-above 5/second \
		--hashlimit-burst 10 \
		--hashlimit-mode srcip \
		-j DROP
		```
- jak reagować: 
	- zablokować IP atakującego na firewall (`iptables -A INPUT -s IP -j DROP`), lub odizolować zaatakowane urządzenie, 
	- ustawić rate limiting na firewall, 
	- sprawdzić, czy tylko zaufane sieci mają dostęp do serwera, jeśli nie to ograniczyć dostęp, 
	- wyłączyć (jeśli nie zostało wcześniej wyłączone) polecenie `monlist` na serwerze. 

## Dlaczego UDP jest trudniejszy do obrony niż TCP
- brak handshake - atakujący ma możliwość sfałszowania adresu IP,
- brak potwierdzeń - nie wiadomo, czy ruch jest legalny, czy to atak, 
- firewall nie widzi sesji - więc nie ma czego śledzić. 

## Przesłanki do podejrzenia ataku przez UDP
- nagły wzrost ruchu UDP z jednego IP, 
- duże odpowiedzi DNS/NTP bez odpowiadających zapytań,
- ruch UDP na nietypowych portach (np. 4444/UDP),
- serwer DNS odpowiada do wewnętrznego hosta, który o nic nie pytał. 

## Case study
Kontekst: Firma ma publiczny serwer DNS `ns1.firma.com` (IP 192.0.2.10). Pojawia się alert z monitoringu sieci :"nagły wzrost ruchu UDP na porcie 53". Fragment z tcpdump uruchomiony na serwerze: 
```
14:32:01.100 IP 203.0.113.50.54321 > 192.0.2.10.53: 12345+ [1au] ANY? google.com. (58)
14:32:01.101 IP 203.0.113.50.54322 > 192.0.2.10.53: 12346+ [1au] ANY? google.com. (58)
14:32:01.101 IP 203.0.113.50.54323 > 192.0.2.10.53: 12347+ [1au] ANY? google.com. (58)
... (tysiące podobnych zapytań z 203.0.113.50) ....
14:32:01.200 IP 192.0.2.10.53 > 198.51.100.10.12345: 12345 15/4/8 A 142.250.x.x (4096)
14:32:01.201 IP 192.0.2.10.53 > 198.51.100.10.12346: 12346 15/4/8 A 142.250.x.x (4096)
... (tysiące dużych odpowiedzi do 198.51.100.10) ...
```
Jednocześnie użytkownicy zgłaszają, że połączenie internetowe jest bardzo powolne.   

Analiza: 
- w przypadku tym nastąpił atak DNS amplification:
	- atakujący o adresie IP 203.0.113.50 wysłał wiele zapytań do DNS o wszystkie rekordy google.com (ANY?), 
	- 12345+ to ID transakcji, flaga plus na końcu oznacza, że rekurencja jest pożądana, 
	- każde zapytanie miało wielkość 58 bajtów, wiec zapewne jedna linijka tekstu, 
	- atakujący użył IP spoofing by podszyć się pod adres IP ofiary w tym przypadku 198.51.100.10,
	- na każde zapytanie na adres ofiary przyszły duże odpowiedzi 4096 bajtów, wszystkie rekordy google.com,
	- 15/4/8 oznacza 15 rekordów odpowiedzi / 4 serwery autorytatywnych / 8 dodatkowych rekordów. 
		- 15 rekordów odpowiedzi - serwer zna odpowiedź i wysyła 15 rekordów,
		- 4 serwery autorytatywne - serwer mówi" "te 4 serwery DNS są właścicielami domeny google.com, im możesz zaufać",
		- 8 rekordów dodatkowych - serwer dodaje adresy IP tych 4 serwerów autorytatywnych (po dwa na każdy), żeby nie trzeba było pytać drugi raz. 
	- A 142.250.x.x to rekord A (adres IPv4) dla google.com.
- dlaczego tak się stało:
	- firewall - rate limiting jest nie ustawiony, nie ma żadnych ograniczeń zapytań, 
	- na serwerze nie skonfigurowano ograniczeń zapytań rekurencyjnych do zakresu zaufanych źródeł, ani nie określono zaufanych sieci. 
- co należy zrobić:
	 - zablokować IP atakującego na firewall za pomocą polecenia `iptables -A INPUT -s 203.0.113.50 -j DROP`,
	 - ustawić rate limiting dla UDP ogółem lub dla UDP na porcie 53, 
	 - skonfigurować ograniczenie rekurencji DNS na serwerze, tak by tylko na zapytania z zaufanych adresów DNS odpowiadał. 