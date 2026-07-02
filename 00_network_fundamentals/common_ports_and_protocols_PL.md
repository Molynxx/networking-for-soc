# common_ports_and_protocols

## Cel
Zapoznanie się z najważniejszymi protokołami sieciowymi, zrozumienie ich działania, przypisanych portów, a także zagrożeń z perspektywy SOC. 

## Czym są protokoły sieciowe 
Protokół to zestaw reguł, które określają, w jaki sposób urządzenia komunikują się przez sieć. Definiują format wiadomości, kolejność ich wysłania oraz sposób reagowania na błędy. Bez protokołów urządzenia różnych producentów nie mogłyby się ze sobą porozumieć. 

## Czym są porty
Port to 16-bitowy numer (od 0 do 65535), który identyfikuje konkretną usługę lub aplikację działającą na urządzeniu. Podczas gdy IP wskazuje na konkretne urządzenie w sieci, port wskazuje na konkretny proces na tym urządzeniu. Można to porównać np. do adresu zamieszkania: IP to adres budynku (np. ul. Sportowa 10). port w tym przypadku to numer mieszkania w tym budynku (np. mieszkanie 22 = SSH).  
Dzięki portom urządzenie może obsługiwać jednocześnie wiele usług, np. serwer www (port 443), SSH (port 22), FTP (port 21) i pocztę (port 25).   
Podział portów:
- `0-1023` - to porty uprzywilejowane (well-known), które są zarezerwowane dla standardowych usług takich jak HTTPS, SSH, FTP, DNS, SMPT, itp, 
- `1024-49151` - to porty zarejestrowane, używane przez aplikacje, np MySQL - port 3306, 
- `49152-65535` - to porty dynamiczne/prywatne, które są używane tymczasowo przez klientów.  
Podejrzane usługi często działają na nietypowych porach (np. reverse shell na porcie 4444). 

## Protokoły warstwy aplikacji (7)

### HTTP (Hypertext Transfer Protocol)
- port: 80, 
- transport: TCP,
- co robi: przesyła strony internetowe pomiędzy serwerem a klientem (przeglądarką). PO wpisaniu adresu strony, przeglądarka wysyła żądanie HTTP do serwera, a serwer w odpowiedzi wysyła stronę. 
- jak działa: klient wysyła request  (np. GET /index.html HTTP/1.1). Serwer odpowiada response (np. HTTP/1.1 200 OK + treść strony). Protokół HTTP jest bezstanowy - to oznacza, że każde jedno żądanie jest niezależnie od poprzednich, 
- zagrożenia: SQL injection ( wstrzyknięcie kodu SQL przez parametr URL), XSS (wstrzyknięcie JavaScript  w stronę), path traversal (../../etc/passwd). Należy także pamiętać, że dane w tym protokole nie są szyfrowane, więc są wrażliwe na podsłuch. W logach widać każde żądanie - metodę, URL oraz kod odpowiedzi. 

### HTTPS (HTTP Secure)
- port: 443, 
- transport: TCP, 
- co robi: port ten robi to samo, z tą jedną różnicą, że dane są szyfrowane przez TLS, co sprawia, że nie jest tak podatny na podsłuch jak HTTP, 
- jak działa: w pierwszej kolejności jest uzgadniany klucz szyfrowania (TLS handshake - warstwa 6), a dopiero potem normalny ruch HTTP idzie szyfrowanym tunelem, 
- zagrożenia: atakujący może używać HTTPS do ukrycia złośliwego ruchu (C2 - zainfekowane urządzenie komunikuje się z zewnętrznym serwerem, by ustalić co ma robić, to ruch dwukierunkowy, bo serwer odpowiada,  exfiltracja - wykradanie danych, przesyłanie ich na serwer przy wykorzystaniu C2). W SOC nie widać treści, ale widać metryki: cel, źródło, wielkość pakietów. 

### SSH (Secure Shell)
- port: 22,
- transport: TCP, 
- co robi: pozwala na zdalne, szyfrowane logowanie do systemu (konsola). To następca protokołu Telnet, który nie był szyfrowany, 
- jak działa: klient łączy się z serwerem, następnie jest uzgadniane szyfrowanie (wymiana kluczy), a potem następuje uwierzytelnianie, autoryzacja, otwarcie sesji, zamknięcie sesji, 
- zagrożenia: podstawowym zagrożeniem dla tego protokołu jest atak brute force, np atakujący próbuje haseł by dostać się na konto root. W logach `auth.log` widać `Failed password for root from IP`. Niebezpieczne jest gdy po wielu takich nieudanych logowaniach z danego IP następuje `Accepted password for root` - w takim przypadku jest prawdopodobne, że konto zostało skompromitowane. 

### Telnet
- port: 23, 
- transport: TCP, 
- co robi: umożliwia zdalne logowanie, lecz jest ono nieszyfrowane, obecnie zastąpiony SSH, 
- jak działa: tak samo jak SSH, lecz ponieważ nie ma szyfrowania, wszystko jest w jawnym tekście (plaintext), więc każdy na trasie może podejrzeć hasło, 
- zagrożenia: już samo używanie tego protokołu oznacza czerwoną flagę. Otwarty port 23 to prawdopodobnie stary system albo złośliwa usługa. Nie należy tego protokołu używać w systemach produkcyjnych. 

### FTP (File Transfer Protocol)
- port: 21 (sterowanie) i 20 (dane), 
- transport: TCP,
- co robi: umożliwia transfer plików pomiędzy klientem a serwerem, 
- jak działa: klient łączy się na porcie 21 - tutaj są polecenia: 
	- `USER, PASS`- logowanie, 
	- `LIST` - przeglądanie folderów, 
	- `RETR` - żądanie pliku,
	- `STOR` - wysyłanie pliku, 
	Przy każdym poleceniu wymagającym przesłania danych (`RETR`, `STOR`), serwer otwiera drugie połączenie na porcie 20 i tym kanałem przesyła dane. 
- zagrożenia: hasła i pliki są w plaintext, Anonymous FTP (logowanie bez hasła) to częsta luka. To co powinno zaniepokoić to podejrzany transfer plików z serwera, co może oznaczać exfiltrację danych.

### FTPS (FTP Secure)
- port: 
	- 21 (z TLS - klient łączy się na port 21 i żąda zabezpieczenia, dopiero po tym przełącza się na szyfrowane połączenie),
	- 990 (implicit TLS - połączenie jest od razu szyfrowane od samego początku bez potrzeby pytania klienta o zabezpieczenie),
- transport: TCP, 
- co robi: dokładnie to samo co z FTP, ale już z szyfrowaniem TLS, 
- jak działa: tak samo jak FTP, ale najpierw uzgadnia TLS, a polecenia i dane są szyfrowane, 
- zagrożenia: jest znacznie bezpieczniejszy niż FTP, jednak nadal nieautoryzowany transfer plików to możliwa kradzież danych, 

### SFTP (SSH File Transfer Protocol)
- port: 22, 
- transport: TCP, 
- co robi: odpowiada za transfer plików przez SSH. Nie należy mylić z FTP, to zupełnie inny protokół, 
- jak działa: jest częścią SSH, po nawiązaniu sesji SSH, klient może wysyłać pliki przez ten sam tunel, 
- zagrożenia: podobnie jak SSH - największym zagrożeniem jest brute force, jednak w logach widać tylko połączenie SSH, a nie same pliki. 

### SMTP (Simple Mail Transfer Protocol)
- port: 
	- 25 - domyślny,
	- 587 - submission z TLS - jawnie prosi o szyfrowanie, 
	- 465 - SMTPS - od razu szyfrowany, 
- transport: TCP, 
- co robi: przesyła maile między serwerami i od klienta do serwera (tylko proces wysłania maila, za odbieranie odpowiada inny protokół), 
- jak działa: klient lub serwer nawiązuje połączenie, identyfikuje się, podaje nadawcę, odbiorcę i treść maila, a serwer przekazuje ją dalej lub dostarcza do skrzynki odbiorcy. Odbywa się to za pomocą poleceń tekstowych np.: `HELO`, `MAIL FROM`, `RCPT TO`, `DATA`. 
- Zagrożenia: open relay - serwer pozwala wysyłać maile bez autoryzacji -> spam. Phishing - atakujący wysyła maile podszywające się pod firmę. Należy analizować nagłówki maili. 

### POP3 (Post Office Protocol v3)
- port:
	- 110 (nieszyfrowany),
	- 995 (szyfrowany - TLS),
- transport: TCP, 
- co robi: odpowiada za odbieranie maili z serwera na komputer klienta. Protokół ten ściąga je lokalnie, usuwając je z serwera, 
- jak działa: klient łączy się, podaje login i hasło, pobiera wiadomości i kasuje je od razu z serwera, 
- zagrożenia: hasła w plaintext na porcie 110 - bezpieczniej używać portu 995 lub protokołu IMAP, który zostawia kopię na serwerze - to ułatwia audyt, brute force.

### IMAP (Internet Message Access Protocol)
- port: 
	- 143 (plain), 
	- 993 (IMAPS - z TLS), 
- transport: TCP, 
- co robi: Odbiera maile z serwera jednocześnie pozostawiając na nim kopię. Klient tylko je wyświetla, 
- jak działa: klient synchronizuje  foldery (INBOX, SENT) z serwerem, Wiadomości pozostają na serwerze, 
- zagrożenia: brute force kont mailowych. 

### DNS (Domain Name System)
- port: 53, 
- transport: UDP / TCP, 
- co robi: tłumaczy adresy domenowe np. google.com na adresy IP (142.250.x.x), 
- jak działa: jak książka telefoniczna internetu - urządzenie pyta DNS: "jaki IP ma google.com?", a serwer sprawdza i odpowiada. Jeśli jednak nie wie - pyta wyżej, rekurencyjnie. 
- zagrożenia: DNS tunnelling - ukrywanie ruchu C2 w zapytaniach DNS. DGA (Domain Generation Algorithm) - malware generuje losowe nazwy domen, żeby utrudnić blokowanie. DNS spoofing/poisoning - fałszywe odpowiedzi DNS. 

### DHCP (Dynamic Host Configuration Protocol)
- port: 
	- 67 (serwer),
	- 68 (klient),
- transport: UDP, 
- co robi: automatycznie przydziela adresy IP, maskę, bramę, DNS,
- jak działa: działa na zasadzie 4 kroków:
	- Discover - klient wysyła wiadomość: "Jestem nowy, czy ktoś tu rozdaje adresy IP?",
	- Offer - serwer odpowiada: "Cześć jestem mam dla Ciebie adres 192.168.1.25",
	- Request - klient akceptuje: "ok, wezmę ten adres",
	- ACK - serwer potwierdza: "Potwierdza, masz ten adres na czas na 24h".   
	Po 12h, klient czyli w połowie czasu, klient próbuje odnowić "dzierżawę" tego adresu, wysyłając do serwera zapytanie: "Hej, czy mogę przedłużyć ten adres?". Jeśli serwer odpowie, adres się przedłuża na kolejne 24h, a proces się powtarza. Nie zachodzi przerwa w połączeniu ani zmiana adresu. Jeśli jednak serwer nie odpowiada to po 24h klient wysyła zapytanie ponownie, jeśli serwer nadal nie odpowiada -> adres wygasa. Po wygaśnięciu klient zaczyna cały proces od nowa Discover -> Offer -> Request -> Ack, może wtedy dostać ten sam adres jeśli jest dostępny. 
- zagrożenia: Rogue DHCP - atakujący stawia fałszywy serwer DHCP i rozdaje IP ze swoją bramą -> MITM. DHCP starvation - wyczerpanie puli adresów przez fałszywe żądania. 

### NTP (Network Time Protocol)
- port: 123,
- transport: UDP,
- co robi: synchronizuje zegary między urządzeniami. To jest krytyczne dla logów, bo bez poprawnego czasu skorelowanie zdarzeń jest utrudnione, 
- jak działa: Klient pyta serwer NTP o dokładny czas -> serwer odpowiada -> klient koryguje swój zegar, 
- zagrożenia: NTP amplification - atak DDoS: atakujący wysyła małe zapytanie NTP, serwer odpowiada dużym pakietem, a to powoduje zalewanie ofiary. Fałszowanie czasu - atakujący zmienia czas, żeby zatrzeć ślady w logach. 

### SNMP (Simple Network Management Protocol)
- port:
	- 161 - zapytania, 
	- 162 - traps -pułapki/alerty, 
- transport: UDP, 
- co robi: monitoruje i zarządza urządzeniami takimi jak routery, switche, drukarki, 
- jak działa: menadżer SNMP pyta urządzenie: "Jaki jest Twój CPU? Ile błędów na interfejsie?", a urządzenie odpowiada. Może także samo wysyłać alerty (SNMP traps),
- zagrożenia: domyślny community string ("public" dla odczytu, "private" dla zapisu) jest często niezmieniony. Atakujący może odczytać konfigurację sieci lub zmienić ustawienia. 

### RDP (Remote Desktop Protocol)
- port: 3389, 
- transport: TCP, 
- co robi: to zdalny pulpit Windows - można widzieć i kontrolować ekran zdalnego komputera, 
- jak działa: klient łączy się z serwerem RDP i po uwierzytelnieniu i autoryzacji jest przesyłany obraz ekranu oraz polecenia klawiatury/myszy, 
- zagrożenia: brute force na RDP jest jedną z najczęstszych metod włamań, szczególnie ransomware. Otwarty port 3389 do Internetu to czerwona flaga dla SOC. 

## Protokoły warstwy prezentacji (6) 

### TLS/SSL (Transport Layer Security / Secure Sockets Layer)
- port: nie ma własnego, działa jako warstwa pod HTTP, SMTP, FTP, itd, 
- co robi: jego zadaniem jest szyfrowanie danych między klientem a serwerem, dzięki czemu hasła, numery kart i treść stron nie są przesyłane otwartym tekstem, 
- jak działa (TLS handshake w uproszczeniu):
	- klient wysyła żądanie "Hej, chcę szyfrowane połączenie. Obsługuję takie algorytmy szyfrowania: ...",
	- serwer odpowiada: "ok, użyjemy tego algorytmu, oto Twój certyfikat (dowód tożsamości)",
	- klient sprawdza certyfikat pod kątem ważności oraz zaufania, 
	- zostaje uzgodniony klucz sesyjny (symetryczny - szybszy),  
	- od tego momentu cały ruch jest szyfrowany za pomocą tego klucza, 
- zagrożenia: nieważny lub fałszywy certyfikat to MITM -> atakujący podstawia swój certyfikat, dzięki czemu może czytać wszystko. W logach: błędy certyfikatów, podejrzani wystawcy (np. darmowe certyfikaty Let's Encrypt dla domen phishingowych).

## Protokoły warstwy sesji (5)
Tu nie występują protokoły, są tu wyłącznie mechanizmy sesji:
- sesja: logowanie -> sesja się otwiera, teraz można przeglądać konto, dokonywać zmian, a wszystko dzieje się w ramach jednej sesji. Po wylogowaniu sesja zostaje zamknięta, 
- ciasteczka (cookies): to mały plik tekstowy, który serwer daje przeglądarce. Przeglądarka odsyła go przy każdym żądaniu. W pliku jest identyfikator sesji, stąd serwer wie, że to wciąż ten sam użytkownik, 
- tokeny: podobnie jak ciasteczka, lecz używanego głownie przez API i aplikacje mobilne (np. JWT, JSON Web Token).   
Zagrożenia: session hijacking - atakujący kradnie ciasteczko sesyjne np. przez XSS, MITM lub malware i podszywa się pod zalogowanego użytkownika. Nie potrzebuje hasła, po porostu używa wykradzionej sesji. 

## Protokoły warstwy transportowej (4)

### TCP (Transmission Control Protocol)
- porty: 0-65535, 
- co robi: to niezawodna metoda transmisji danych, dzięki nawiązaniu połączenia z odbiorcą, utrzymaniu go i sprawdzaniu sum kontrolnych gwarantuje, że dane dotrą bezbłędnie w całości oraz w odpowiedniej kolejności, 
- jak działa "TCP handshake - three-way handshake"):
	- SYN: - klient wysyła żądanie do serwera: "Chcę się połączyć, mój numer startowy to X", 
	- SYN-ACK - serwer odpowiada: "OK, przyjąłem, mój numer to Y. POtwierdzam Twój X+1",
	- ACK - klient odpowiada serwerowi: "Potwierdzam Y+1, połączenie nawiązane".   
	Następnie zostają przesyłane dane, TCP numeruje każdy segment, a jeśli coś zaginie prosi o powtórzenie. Na końcu następuje FIN (zakończenie). 
- zagrożenia: SYN flood - atakujący wysyła masę SYN bez dokończenia handshake, co powoduje, że serwer czeka na ACK, to zapycha pamięć i prowadzi do DoS. Podejrzane porty: 4444, 1337, 31337. 

### UDP (User Data Protocol)
- porty: 0-65535, 
- co robi: zapewnia szybką transmisję bez gwarancji dostarczenia. Nie sprawdza sum kontrolnych, nie nawiązuje połączenia, nie numeruje pakietów, 
- jak działa: jego zadaniem jest wysłać i zapomnieć. Nawet jeśli jakiś pakiet zaginie, odbiorca sam musi sobie z tym poradzić. Dobrym przykładem transferu UDP jest np streaming video, 
- zagrożenia: UDP flood - atakujący zalewa ofiarę pakietami UDP, DNS amplification - małe zapytanie DNS, duża odpowiedź. 

## Protokoły warstwy sieciowej (3)

### IP (Internet Protocol)
- port: nie posiada, IP nie używa portów, 
- co robi: adresuje i trasuje pakiety. Każdy pakiet ma adres nadawcy i odbiorcy, 
- jak działa: routery czytają adres docelowy i decydują, którędy wysłać pakiet dalej. IP nie gwarantuje dostarczenia wiadomości, to zadanie dla protokołu TCP.
- zagrożenia: IP spoofing - fałszowanie adresu źródłowego. DDoS - zalewanie ofiary pakietami z wielu IP. W logach widać adresy źródłowe ataków. 

### ICMP (Internet Control Message Protocol)
- port: brak, 
- co robi: wspiera diagnostykę sieci -> ping (echo request -> echo reply). Sprawdza czy host jest osiągalny, 
- jak działa: ping, czyli wysłanie małych pakietów, by sprawdzić czy serwer odpowie, jeśli nie odpowiada, może to oznaczać brak połączenia, blokadę ICMP przez firewall lub wyłączony host. 
- zagrożenia: ICMP tunneling - ukrywanie ruchu C2 w pakietach ICMP (pingi, które niosą dane), Ping flood - zalewanie ICMP echo request.

## Protokoły warstwy łącza danych (2)

### ARP (Address Resolution Protocol)
- port: brak, 
- co robi: tłumaczy adresy IP na MAC w sieci lokalnej, 
- jak działa: 
	- urządzenie x (192.168.1.5) chce wysłać dane do urządzenia y (192.168.1.10). Urządzenie x zna adres IP urządzenia y, lecz nie zna MAC, 
	- urządzenie x wysyła ARP Request (broadcast - do wszystkich) z zapytaniem: "Kto ma IP 192.168.1.10? Podaj swój MAC", 
	- urządzenie y odpowiada ARP Reply: "Ja mam ten adres, mój MAC to aa:cc:bb:dd:ee:ff", 
	- urządzenie x zapisuje parę IP+MAC w tablicy ARP i wysyła dane bezpośrednio, 
- zagrożenia: ARP spoofing/poisoning - atakujący wysyła fałszywy ARP Reply ("ja jestem bramą 192.168.1.1") i przechwytuje cały ruch sieciowy ofiary. Tablicę ARP można podejrzeć za pomocą polecenia `ip neigh` lub `arp -a`. 

## Warstwa fizyczna (1)
Nie posiada żadnych protokołów, jej zadaniem jest tylko przesłanie wszystkiego co przygotowały wyższe warstwy przez medium transmisyjne. 
