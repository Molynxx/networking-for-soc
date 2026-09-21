# http_basics

## Cel
Zrozumienie czym jest protokół http, jakich używa metod oraz na jakie zagrożenia jest narażony.

## Czym jest http
HTTP (Hypertext Transfer Protocol) jest protokołem warstwy aplikacji, służący do komunikacji pomiędzy klientem (aplikacja, przeglądarka) a serwerem www. To jeden z najważniejszych protokołów dla SOC, ponieważ:
- niemal cały ruch webowy przechodzi przez protokół HTTP, 
- większość aplikacyjnych ataków takich jak SQLi, XSS, phishing czy brute force, można wykryć w logach HTTP, 
- zrozumienie kodów, metod oraz nagłówków to podstawa triage'u.   

## Jak działa HTTP
Działanie tego protokołu opiera się na żądaniach i odpowiedziach -> klient wysyła żądanie, serwer odpowiada na nie. HTTP to nie tunel jak SSH, to seria niezależnych pytań i odpowiedzi. Po tej wymianie zależnie od wersji HTTP następują działania:
- HTTP 1.0 - jedno żądanie = jedno połączenie TCP - po każdym pytaniu i odpowiedzi połączenie zostaje zamykane, więc przy kolejnym musi nastąpić ponowne three way handshake, to mało wydajne dlatego ta wersja HTTP jest w zasadzie martwa, nieużywana, 
- HTTP 1.1 - problem braku wydajności został rozwiązany poprzez podtrzymywanie połączenia. Po każdym pytaniu i po każdej odpowiedzi serwer czeka jeszcze chwilę, czy nie nadejdzie kolejne. Dopiero gdy przez jakiś czas jest cisza, połączenie jest zamykane. Można wysyłać kilka żądań, bez czekania na odpowiedzi jednak jeśli jedno żądanie się zablokuje, reszta czeka. W praktyce przeglądarki otwierają kilka połączeń równolegle żeby obejść ten problem. HTTP/1.1 jest łatwy w analizie SOC ponieważ plik jest tekstowy, widać wszystko w logach, 
- HTTP 2.0 - tutaj weszło więcej zmian:
	- multipleksacja - czyli obsługa wielu żądań w jednym połączeniu TCP, 
	- binarny format zamiast tekstowego, 
	- większa wydajność dzięki kompresji nagłówków,  
	Protokół nadal działa na TCP, a jeśli TCP zgubi pakiet, wszystko się blokuje (Head-of-line blocking). Trudniejszy do analizy niż poprzednik, ponieważ jest binarny. Jednak dzięki temu, że używa TCP jest widoczny na firewallu, 
- HTTP/3 - najnowsza wersja (z 2022 roku). Względem poprzednika zaszło tu naprawdę wiele zmian:
	- nie używa już TCP, korzysta z QUIC, 
	- QUIC to protokół korzystający z UDP, 
	- problem Head-of-line blocking nie występuje, 
	- jest szyfrowany od samego początku, 
	- szybki handshake (0-1 RTT),    
	Jednak jest znacznie trudniejszy do monitorowania, ponieważ SNI widoczne jest tylko w Initial Packet, a żeby móc go monitorować potrzebne jest odpowiednie oprogramowanie, np. Zeek, Suricata lub proxy.

## Metody HTTP 
Metoda to informacja dla serwera, co klient chce zrobić. 

### GET
To najczęstszy ruch. 
- co przekazuje: to informacja, że klient chce pobrać dane z serwera, 
- przykład: GET /index.html HTTP/1.1,
- SOC: 
	- GET z długimi, zakodowanymi parametrami powinny budzić podejrzenia, ponieważ mogą wskazywać na ataki SQLi lub XSS, 
	- czujność powinna budzić także duża ilość GET - normalny użytkownik nie wysyła tysięcy zapytań GET do serwera, więc to może wskazywać na rekonesans przy użyciu skanera przed przeprowadzeniem ataku.

### POST
- co przekazuje: klient chce przesłać dane do serwera, 
- przykład: POST /login HTTP/1.1, 
- SOC: 
	- POST /login co kilka sekund może oznaczać atak brute force, 
	- duże POST może wskazywać na exfiltrację, 
	- nietypowe POST do API może oznaczać próbę nadużycia (np. /api/debug, wiele POST z jednego IP w krótkim czasie, POST z polami, których API nie oczekuje "role=admin", poza godzinami pracy z nietypowego IP),

### HEAD 
Podobnie jak GET, jednak serwer zwraca tylko nagłówki, nie zwraca treści. 
- SOC: bywa używany przez skanery do sprawdzania, co istnieje. Więc jeśli w logach widać wiele żądań HEAD w krótkim czasie, to może oznaczać rekonesans przed atakiem, 

### PUT 
- co przekazuje: klient chce wgrać plik na serwer, 
- SOC: może posłużyć do wgrania webshella, jeśli serwer nie powinien przyjmować plików - czerwona flaga. 

### DELETE 
- co przekazuje: klient chce usunąć zasób, 
- SOC: rzadko występuje jako zagrożenie, jednak jest podejrzane jeśli nie pasuje do aplikacji. 

### OPTIONS
- co przekazuje: pyta serwer o to, jakie metody są dozwolone, 
- SOC: 
	- jeśli OPTIONS z jednego IP jest wysyłane setki razy na różne ścieżki może to oznaczać skanowanie, czyli rekonesans przed atakiem, 
	- jeśli w odpowiedzi na OPTIONS serwer zdradza zbyt wiele, czyli zwraca listę metod które nie powinny być publiczne (np. PUT, DELETE) może oznaczać lukę, podatność, która może zostać wykorzystana przez atakującego. 

### PATCH
- co przekazuje: klient chce częściowo modyfikować zasób, 
- SOC: to rzadkie zagrożenie, choć może być wykorzystywane w atakach na API, dlatego należy sprawdzać czy PATCH nie zawiera pól spoza schematu (np. "role", "is_admin").

### TRACE 
- co przekazuje: ze serwer powinien zwrócić to co wysłał klient (echo),
- SOC: samo pojawienie się tej metody powinno budzić podejrzenia, ponieważ normalnie nikt tego nie używa. Jednak jeśli TRACE występuje w logach, może to oznaczać kradzież sesji ofiary. To rodzaj ataku XST (Cross-Site Tracing), który działa następująco: 
	- atakujący wstrzykuje skrypt do przeglądarki ofiary, 
	- skrypt ma za zadanie wysłać TRACE z ciasteczkiem ofiary, 
	- serwer odsyła mu to ciasteczko z powrotem, 
	- skrypt je odczytuje i wysyła do atakującego. 

### CONNECT
- co przekazuje: klient prosi serwer o utworzenie tunelu do innego miejsca, używane głównie przez proxy. 
- SOC: może służyć do nadużywania tunelowania. Taki ruch jest normalny gdy firma ma proxy lub przeglądarka chce połączyć się przez proxy. Jednak CONNECT dla nietypowych portów (nie 443), dla podejrzanych domen, dużo CONNECT z jednego IP lub CONNECT poza godzinami pracy powinno budzić podejrzenia. To może bowiem oznaczać, że atakujący może przez CONNECT ukryć ruch C2.

## Kody odpowiedzi HTTP
To język, którym posługuje się serwer z klientem. Gdy klient prosi o stronę, serwer odsyła nie tylko treść, ale też trzycyfrowy kod, który mówi co się stało z żądaniem. Przykład: Jeśli wpisujesz w przeglądarkę google.com - serwer odsyła 200 -> "mam tę stronę, oto ona". Pierwsza cyfra określa, do jakiej kategorii należy odpowiedź. 
- Kategorie
	- 2xx - sukces - serwer otrzymał, zrozumiał i wykonał żądanie. 
		- 200 OK - to standardowy kod sukcesu, serwer zwraca to o co poprosił klient, np. może przeglądać google.com, 
		- 201 Created - oznacza, że serwer coś utworzył. Np. gdy zakładasz konto, serwer potwierdza, że zostało utworzone, 
		- 204 No Content - oznacza sukces, lecz serwer nie zwraca niczego. Np. serwer odpowiada "OK" po przyjęciu formularza, nic nie odsyła nowej strony. 
	- 3xx - przekierowanie - serwer mówi, że należy przejść gdzie indziej.
		- 301 Moved Permanently - strona na stałe zmieniła swój adres, np. `example.com` przekierowuje na `www.example.com`, 
		- 302 Found - tymczasowe przekierowanie, np. gdy po logowaniu serwer przekierowuje na dashboard.  
		Serwer wysyła także nagłówek `Location:` z nowym adresem.
	- 4xx - błąd klienta - serwer informuje, że występuje problem po stronie klienta.
		- 400 - Bad Request - serwer nie zrozumiał żądania, (z powodu złej składni). 
		- 401 Unauthorized - serwer informuje, że wymagane jest logowanie. Np. próba wejścia na `/admin` bez logowania,
		- 403 Forbidden - serwer informuje ze klient nie ma dostępu, pomimo tego, że jest zalogowany. Np. gdy klient próbuje wejść na cudzy plik,
		- 404 Not Found - to informacja od serwera, że zasób nie istnieje. Np. po wpisaniu przez klienta nieprawidłowego adresu, 
		- 405 Method Not Allowed - zabroniona metoda, np. w przypadku gdy klient próbuje wysłać POST tam gdzie jest dozwolony tylko GET.  
	- 5xx - błąd serwera - serwer informuje, że problem leży po jego stronie.
		- 500 Internal Server Error - to informacja o ogólnym błędzie serwera, np. gdy skrypt na serwerze się wysypał, 
		- 502 Bad Gateway - informacja, że pośrednik otrzymał złą odpowiedz, np. gdy reverse proxy nie dogadał się z backendem, 
		- 503 Service Unavailable - to informacja, że serwer chwilowo nie działa, np. podczas konserwacji lub przeciążenia, 
		- 504 Gateway Timeout - oznacza pośrednik nie doczekał się odpowiedzi, np. gdy backend odpowiada zbyt wolno. 

## Nagłówki HTTP
Nagłówki to rodzaj metadanych, zawierają dodatkowe informacje, które lecą razem z żądaniem/odpowiedzią, nie z treścią strony.	
- Nagłówki żądania (klient -> serwer):
	- Host - informuje do jakiej domeny chce połączyć się klient, np. `Host: example.com`,
	- User-Agent - to informacja, jaka przeglądarka/narzędzie wysyła żądanie, np. `User-Agent: Mozilla/5.0`, 
	- Referer - informuje z jakiej strony przyszło żądanie, np. `Referer: google.com`, 
	- Cookie - przechowuje sesję użytkownika, np. `Cookie: session=asd234`,
	- Authorization - przechowuje dane logowania, np. `Authorization: Basic dJWSAuASDyr`, 
	- X-Forwarded-For - informuje, jakie było oryginalne IP klienta (jeśli jest proxy), np. `X-Forwarded-For: 203.0.111.12`.
- Nagłówki odpowiedzi (serwer -> klient):
	- Server - mówi, jakie oprogramowanie działa na serwerze, np. `Server: nginx`, 
	- Content-Type - to informacja, jaki typ treści serwer zawiera, np. `Content-type: text/html`, 
	- Content-length - informuje, jak długa jest treść, np. `Content-Length: 1234`, 
	- Set-Cookie - serwer ustawia ciasteczko sesji, np. `Set-Cookie: session=abc345`, 
	- Location - przy przekierowaniach informuje, dokąd iść, np. `https://exp.com/login`.

## Przykład pełnego żądania i odpowiedzi
- żądanie: 
```
GET /login.php HTTP/1.1
Host: exp.com
User-Agent: Mozilla/5.0
Accept: text/html
Cookie: session=sdf456
```
- odpowiedź:
```
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html
Content-Length: 1123
Set-Cookie: session=zxc456; Path=/; HttpOnly
``` 
