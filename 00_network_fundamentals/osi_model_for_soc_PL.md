# osi_model_for_soc

## Cel
Zrozumienie jak działa model OSI, za co są odpowiedzialne poszczególne warstwy modelu oraz jaki to ma wpływ na diagnozy SOC.

## OSI 
Jest to model teoretyczny, który dzieli komunikację sieciową na 7 warstw - od kabla po aplikację, którą widać na ekranie. Został stworzony by różne firmy mogły tworzyć sprzęt i oprogramowanie, które będą ze sobą sprawnie współpracować. Dzięki temu:
- karta sieciowa z Intela, działa z routerem Cisco, 
- przeglądarka Chrome działa z serwerem Apache, 
- Windows łączy się z Linuxem przez SSH.   
UWAGA: To jest wyłącznie model teoretyczny (szablon), który nie jest zgodny z rzeczywistością - prawdziwy internet używa modelu TCP?IP który posiada 4 warstwy. Jednak model OSI warto znać, bo daje wspólny język np. stwierdzenie, że problem występuje w warstwie 3 to wszyscy wiedzą, że chodzi o IP/routing. 

## Warstwy modeli OSI

### 7 Warstwa aplikacji
Jest to warstwa, w której działają protokoły, które obsługują aplikacje wchodzące w bezpośrednią interakcję z użytkownikiem. To najbliższa użytkownikowi warstwa, czyli to co widzi na ekranie. W tej warstwie działają protokoły: 
- `HTTP` - przeglądanie stron WWW (nieszyfrowane),
- `HTTPS` - przeglądanie stron WWW (szyfrowane - TLS),
- `SSH` - zdalne logowanie do systemu (szyfrowane),
- `Telnet` - zdalne logowanie do systemu (starsze, nieszyfrowane, w dzisiejszych czasach nieużywane),
- `FTP` - transfer plików (nieszyfrowany)
- `FTPS` - transfer plików (szyfrowane - TLS),
- `SFTP` - transfer plików przez SSH, 
- `SMTP` - wysyłanie maili, 
- `POP3` - odbieranie maili (ściąga na komputer),
- `IMAP` - odbieranie maili (zostają na serwerze),
- `DNS` - tłumaczy nazwy domen na adresy IP,
- `DHCP` - automatycznie przydziela adresy IP,
- `NTP` - synchronizuje czas między urządzeniami, 
- `SNMP` - monitoring i zarządzanie urządzeniami sieciowymi,
- `RDP` - zdalny pulpit.   
Zagrożenia: Ataki SQLi, XSS, brute force, phishing - to wszystko dzieje się w tej warstwie. W logach widać konkretne żądania HTTP, próby logowania SSH, zapytania DNS.

### 6 Warstwa prezentacji 
Ta warstwa odpowiada za przekształcenie danych tak, aby aplikacja mogła je zrozumieć. Jednymi ale bardzo ważnym protokołem tej warstwy jest protokół TLS/SSL, który szyfruje połączenia. Warstwa ma za zadanie:
- Zaszyfrować połączenia (TLS/SSL),
- przekonwertować formaty - by treść była zrozumiałą dla aplikacji, 
- skompresować - czyli zmniejszyć rozmiar danych przed wysłaniem,
- zakodować znaki - ASCII, UTF-8 - by tekst wyglądał tak samo u nadawcy i odbiorcy.  
Zagrożenia: Nieważne certyfikaty TLS. Atak MITM z fałszywym certyfikatem -> to atak właśnie w tej warstwie. 

### 5 Warstwa sesji
Tak, jak należy rozpocząć rozmowę telefoniczną wybierając numer tak warstwa ta nawiązuje połączenie "rozmowę" między dwoma urządzeniami - początek (wybranie numeru), środek (wymiana zdań), zakończenie (zakończenie połączenia). Na tą warstwę składają się trzy elementy:
- sesja - logowanie, przeglądanie, wylogowanie = to jest jedna sesja,
- ciasteczka (cookies) - przechowują identyfikator sesji, 
- tokeny - identyfikują użytkownika.   
Zagrożenia: Session hijacking - atakujący może wykraść ciasteczko sesyjne i podszywać się pod zalogowanego użytkownika, nie potrzebuje hasła. 

### 4 Warstwa transportowa
To warstwa, której cellem jest niezawodne dostarczenie danych miedzy hostami. Jej zadania to:
- wybór protokołu - decyduje, czy użyć TCP czy UDP, 
- segmentacja - dzieli dane na mniejsze części - segmenty, 
- numeracja - nadaje numer każdej części, żeby można je było poskładać w całość w odpowiedniej kolejności, 
- kontrola przepływu - reguluje szybkość wysyłania, by odbiorca nadążał, 
- wykrywanie błędów - sprawdza, czy dane dotarły w całości i bez błędów (chceksum).
Zagrożenia: SYN flood (atak na TCP), podejrzane protokoły (np. 4444, 1337, 31337) - reverse shell. 

### 3 Warstwa sieciowa
To warstwa nawigacji, ustala kierunek wysłania danych. Ta warstwa ma zadania:
- Adresowanie IP - nadaje każdemu urządzeniu w sieci unikalny adres IP, dzięki czemu wiadomo, do kogo wysłać dane,
- ICMP - diagnostyka sieci (ping),
- routing (trasowanie) - określa najlepszą drogę, którą dane mają podążać, aby dotrzeć do docelowego adresu IP, 
- fragmentacja - dzieli segmenty z warstwy transportowej na mniejsze fragmenty jeśli są zbyt duże, aby zmieścić się w jednej ramce. Fragmenty składane są ponownie u odbiorcy.   
Zagrożenia: IP spoofing, (fałszowanie adresu), DDoS, ICMP tunneling. Za pomocą logów można sprawdzić z jakiego IP przyszedł atak. 

### 2 Warstwa łącza danych
Warstwa ta odpowiada za j=komunikację w tej samej sieci lokalnej, Jeśli warstwa sieciowa decyduje gdzie paczka ma dotrzeć, to warstwa łącza danych fizycznie przekazuje ją do najbliższego urządzenia (routera, switcha). Warstwa ma zadania:
- ARP - protokół, który tłumaczy adres IP na adres MAC (kto ma IP X? Podaj swój MAC). MAC to fizyczny adres karty sieciowej, unikalny, 48 bitowy, zapisywany szesnastkowo, np. aa:bb:cc:dd:ee:ff,
- tworzenie ramek - pakuje dane z warstwy sieciowej w ramki, dodając adresy MAC źródła i celu oraz informacje kontrolne, 
- wykrywanie błędów - dodaje sumę kontrolną, aby sprawdzić, czy ramka nie została uszkodzona podczas transmisji. Jeśli ramka została uszkodzona odrzuca ją. 
- kontrola dostępu do medium - zarządza tym, które urządzenie w danej chwili może nadawać w sieci. 
Zagrożenia: ARP spoofing - atakujący podszywa się pod cudzy adres MAC w sieci i przechwytuje ruch. 

### 1 Warstwa Fizyczna
Warstwa ta fizycznie przenosi sygnał, zależnie od medium:
- kabel miedziany Ethernet - skrętka, wtyczka RJ-45,
- światłowód - szkło, światło - najszybsze,
- fale radiowe - Wi-Fi, Bluetooth, 
- impulsy elektryczne - prąd w kablu. 
Zagrożenia: SOC rzadko analizuje tą warstwę, warto jednak widzieć, że fizyczny dostęp do switcha jest bardzo niebezpieczny, bo atakujący może podpiąć własne urządzenie. 

## Warstwy modelu TCP/IP (stosowany w praktyce)

4 - Warstwa aplikacji - odpowiada za te same czynności, które w modelu OSI wykonują warstwy: aplikacji, prezentacji i sesji, 
3 - Warstwa transportowa - odpowiada za te same czynności co warstwa transportowa w modelu OSI,
2 - warstwa internetowa - odpowiada za te same zadania co warstwa sieci modelu OSI, 
1 - warstwa dostępu do sieci - łączy w sobie zadania z warstw: łącza danych i fizyczne z modelu OSI. 

## Zestawienie modeli OSI i TCP/IP

| Model OSI | Model TPC/IP |
|-----------|--------------|
| Aplikacji |       |
| Prezentacji | Aplikacji |
| Sesji |     |
| Transportowa | Transportowa |
| Sieciowa | Internetowa |
| Łącza danych | Dostępu do sieci |
| Fizyczna |    |
