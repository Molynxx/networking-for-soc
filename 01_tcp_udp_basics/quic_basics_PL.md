# quic_basic

## Cel
Zrozumienie działania protokołu QUIC oraz zagrożeń związanych z tym rodzaje połączenia. 

## Czym jest QUIC (Quick UDP Internet Connections)
QUIC to nowy protokół transportowy na UDP, który łączy szybkość UDP, bezpieczeństwo TLS 1.3, niezawodność TCP, ale bez jego wad. Został stworzony przez Google, aby przyspieszyć internet oraz rozwiązać problemy, które istnieją w połączeniu TCP, takie jak:
- wolny handshake (2-3 RTT z TLS) - RTT (Round Trip Time) to czas od wysłania pakietu do otrzymania odpowiedzi, np.: SYN -> SYN-ACK wynosi 1 RTT, jeśli dołożyć do tego TLS czas zanim dane w ogóle ruszą wydłuża się do 2-3 RTT, podczas gdy RTT dla QUIC wynosi 0-1. 
- head-of-line blocking - w TCP jeden zgubiony pakiet blokuje cały strumień, 
- zrywa połączenie po zmianie sieci np: Wi-Fi -> LTE - QUIC ma Connection ID, czyli identyfikator połączenia, który nie zależy od IP i portu. Dzięki temu zmiana sieci nie wpływa na połączenie. Ma to jednak jedną wadę dla SOC - jest trudniej namierzyć atakującego. 
QUIC powstał właśnie po to żeby te problemy rozwiązać. W jednym połączeniu QUIC może płynąć wiele strumieni na raz, np: 
- strumień 1: obraz, 
- strumień 2: arkusz CSS, 
- strumień 3: dane API.   
Jeśli jeden pakiet w strumieniu 1 się zgubi, pozostałe działają dalej. QUIC sam dba o potwierdzenia (ACK), retransmisje, kolejność w strumieniach. 

## Handshake w QUIC
QUIC łączy transport i szyfrowanie w jednym procesie. 
- Pierwsze połączenie (1 RTT):
	- klient wysyła `ClientHello` z propozycją kluczy, 
	- Serwer odpowiada `ServerHello` i ustala klucze, 
	- od tego momentu połączenie jest ustanowione i dane lecą zaszyfrowane. 
- Kolejne połączenie (0 RTT):
	- jeśli klient raz łączył się z serwerem, może wysyłać dane od razu w pierwszym pakiecie. To dlatego mówi się, że QUIC ma 0-1 RTT. 

## Ataki na QUIC

### QUIC Flood
- co to jest: to zalewanie serwera ogromną ilością pakietów QUIC, 
- cel: DDoS, 
- jak działa: atakujący wysyła duże ilości pakietów na serwer ofiary, ponieważ QUIC działa na UDP a UDP jest bezstanowy, serwer nie wie czy te pakiety przychodzą od zwykłego użytkownika czy od atakującego. Każdy pakiet QUIC wymaga od serwera: parsowania, sprawdzania, odpowiadania, więc przy dużej ilości pakietów serwer zużywa CPU, pamięć oraz łącze, 
- jak wykryć: 
	- widoczny wzrost ruchu UDP na porcie 443, 
	- duża liczba pakietów z jednego IP, 
	- wzrost zużycia CPU na serwerze, co oznacza, że serwer bardzo intensywnie pracuje, 
	- spadek responsywności usługi.
- jak się bronić: 
	- rate limiting na UDP 443 - ograniczy ilość połączeń z jednego IP, 
	- filtrowanie pakietów QUIC na firewall - nowoczesne firewalle umieją śledzić połączenia QUIC, dzięki czemu mogą sprawdzać czy ruch jest normalny (np. czy pochodzi od prawdziwego klienta). Dodatkowo mogą blokować podejrzane wzorce. 
	- usługi anty-DDoS (np. Cloudflare, Akamai) - to usługi lub urządzenia stojące przed serwerem, które mogą:
		- absorbować ruch - wytrzymują ogromne ataki, które zabiłyby przeciętny serwer, 
		- używać testów "retry" - gdy wykryją podejrzane zapytanie, mogą wysłać do klienta specjalne wyzwanie "retry". Prawdziwy klient (np. przeglądarka) odpowie na wyzwanie, a atakujący nie odpowie. Dzięki temu usługa może wykryć fałszywy ruch i go odrzucić,
		- analizować zachowanie - używają uczenia maszynowego do odróżniania prawdziwych użytkowników od atakujących na podstawie ich zachowania, 
		- stosować limitowanie (rate limiting). 

### 0-RTT Replay
- co to jest: to atak polegający na wysłaniu przechwyconego pierwszego pakietu QUIC (0-RTT), 
- cel: powielenie akcji ofiary, np. podwójny przelew, 
- jak działa: QUIC pozwala klientowi wysłać dane w pierwszym pakiecie (0-RTT), jeśli wcześniej łączył się z serwerem. Atakujący, przechwytuje taki pakiet, kopiuje go i wysyła ponownie. Serwer może uznać to za nowe, prawidłowe żądanie, 
- jak wykryć: 
	- powtarzające się identyczne pakiety 0-RTT, 
	- nietypowe duplikaty żądań, 
	- alerty z WAF na ponowne wykonanie akcji, 
- jak się bronić:
	- ustawienie konfiguracji serwera tak żeby akceptował 0-RTT tylko dla żądań idempotentnych (np. GET) - czyli dla operacji, które można bezpiecznie powtarzać. Nie dla operacji zmieniających stan jak np. POST, PUT, DELETE, 
	- ustawienie w konfiguracji serwera jednorazowych tokenów - serwer wymaga unikalnego tokenu, który może być użyty tylko raz, 
	- ustawienie na serwerze krótkiego czasu ważności 0-RTT - czyli 0 RTT jest akceptowane tylko przez kilka sekund od pierwszego żądania, 
	- ustawienie na serwerze szyfrowania metadanych - dodatkowe dane są szyfrowane i weryfikowane, a jakakolwiek zmiana ich przez atakującego powoduje odrzucenie zapytania. 

### Connection ID Spoofing 
- co to jest: to podszywanie się pod cudze połączenie QUIC, 
- cel: przechwycenie sesji, wstrzyknięcie danych, zakłócenie komunikacji, 
- jak działa: 
	- klient i serwer nawiązują połączenie QUIC, 
	- połączenie ma swoje Connection ID, czyli unikalny identyfikator sesji, 
	- atakujący chce wejść do tej sesji, lecz musi poznać Connection ID, a może to zrobić poprzez:
		- podsłuch pasywny (jeśli ID nie jest szyfrowane), 
		- zgadywanie (jeśli ID jest słabe/przewidywalne),
	- atakujący wysyła pakiety z tym samym Connection ID, ale ze swojego IP, 
	- serwer, widząc znany ID, może pomyśleć, że to dalsza część sesji klienta, 
	- jeśli atakującemu się uda, może wstrzykiwać dane lub przechwycić ruch, 
- jak wykryć: 
	- Jeśli w logach widać pakiety z niepasującym Connection ID (różne ID w tej samej sesji), oznacza, że ktoś próbuje odgadnąć Connection ID by przejąć sesje, 
	- dwa różne IP używają tego samego Connection ID jednocześnie:
		- normalnie w sesji występuje IP klienta, oraz identyfikator sesji (np. 192.168.1.10 i abc123), 
		- jeśli w logach widać np. 108.51.100.20 i abc123 to może oznaczać, że atakującemu udało się dostać do sesji, 
		- UWAGA: zmiana IP nie zawsze oznacza, że atakujący jest w sesji, QUIC wspiera utrzymanie połączenia w przypadku zmiany sieci np. Wi-Fi -> LTE, 
	- jedno Connection ID pojawia się z wielu IP w krótkim czasie powinno wzbudzać podejrzenia, 
- jak się bronić:
	- konfiguracja serwera:
		- serwer ma generować losowe, długie Connection ID, 
		- serwer ma rotować Connection ID w trakcie trwania sesji, co utrudnia odgadnięcie, 
		- serwer ma szyfrować Connection ID, żeby podsłuchujący nie mógł go odczytać, 
		- serwer ma weryfikować czy pakiety z nowego IP pasują do sesji (kryptograficznie), 
		- serwer ma odrzucać pakiety, które nie przechodzą weryfikacji.

### Version Downgrade
- co to jest: to jest atak, który zmusza klienta i serwer do użycia słabszej wersji QUIC,
- cel: 
	- osłabienie szyfrowania, 
	- ułatwienie dalszego ataku, 
	- wykorzystanie znanych podatności w starszych wersjach,
- jak to działa:
	- klient łączy się z serwerem i chce użyć nowej wersji QUIC, 
	- atakujący dostaje się pomiędzy klienta i serwer (MitM) i udaje, że serwer nie obsługuje nowej wersji, 
	- klient dostaje informacje "używaj starszej", więc przechodzi na starszą wersję, 
	- stara wersja może mieć słabsze zabezpieczenia, które atakujący może wykorzystać,
- jak wykryć:
	- klient rozpoczyna negocjację nowej wersji, lecz łączy się po starszej wersji, a zmiana ta nie wynika z faktycznej przerwy ani błędu. Na co należy zwrócić uwagę to:
		- nagła zmiana wersji na starszą, 
		- powtarzające się negocjacje rozpoczynające się od zera. 
		- klient w jednym połączeniu używa starszej wersji, choć wcześniej używał nowej, 
- jak się bronić: 
	- QUIC ma wbudowany mechanizm zabezpieczenia, głównie po stronie klienta, który musi sprawdzać informacje od serwera, dlatego należy:
		- aktualizować oprogramowanie - nowe wersje mają zaimplementowane mechanizmy ochrony przed Downgrade, 
		- zarządzanie wersjami na serwerze: jeśli serwer wspiera wiele wersji QUIC, należy wprowadzać nowe wersje stopniowo i jednocześnie wycofywać stare, 
		- wymuszanie najnowszych wersji - w konfiguracji serwera można priorytetyzować najnowsze wersje, aby minimalizować okno na ataki downgrade.

### QUIC Amplification
- co to jest: To atak DDoS, w którym serwer QUIC zostaje wykorzystany jako wzmacniacz ruchu, 
- cel: DDoS - czyli zablokowanie usługi, zalanie ofiary ruchem,
- jak to działa: 
	- atakujący znajduje serwer QUIC, który odpowiada na UDP i wysyła na niego mały pakiet, 
	- w nagłówku pakietu wpisuje fałszywe IP (czyli IP ofiary zamiast swojego), 
	- serwer QUIC myśli, że ofiara prosi o dane i odpowiada dużą odpowiedzią na IP ofiary, 
	- atakujący wysyła tysiące takich pakietów, przez co ofiara dostaje potężną falę ruchu, 
- jak wykryć:
	- dużo odpowiedzi QUIC z jednego serwera do jednego IP, 
	- odpowiedź jest większa niż zapytanie, 
	- IP, które nie wysyłało zapytań, 
- jak się bronić: 
	- serwery QUIC mają wbudowany mechanizm obrony `anti-amplification limit (3x)` - oznacza to, że serwer nie może wysłać odpowiedzi większej niż 3x zapytanie, które dostał. Ta opcja domyślnie jest włączona, nie należy jej wyłączać, 
	- `retry / Address Validation` - w konfiguracji serwera powinno być ustawione `retry=true` oraz `RequireAddressValidation` (quic-go). Przy takiej konfiguracji serwer będzie wysyłać pakiet Retry z tokenem, który klient musi odesłać w kolejnym pakiecie initial. Jeśli tego nie zrobi - handshake się nie powiedzie, 
	- `Flow control` (max data) - konfiguracja QUIC na serwerze, którą ustawia się podczas jego konfiguracji. Określają ile danych może przesłać klient/serwer na starcie, 
	- `rate limiting` - konfiguracja na firewallu, pozwala ustalić limity połączeń serwera z jednym IP w określonym oknie czasowym, 
	- `Anti-Spoofing` - konfigurowany na firewall - wykrywa i blokuje pakiety ze sfałszowanym adresem źródłowym IP.

## QUIC w praktyce SOC

### Co jest widoczne w ruchu QUIC
QUIC to szyfrowany protokół, jednak są dane, które nie są zaszyfrowane: 
- w pierwszym pakiecie (initial packet), który jest najbardziej czytelny, żeby serwer mógł rozpocząć handshake widać:
	- SNI - nazwa domeny (np. google.com),
	- ALPN - protokół aplikacyjny (np h3 dla HTTP/3), 
	- wersję QUIC,
	- Connection ID - identyfikator sesji, 
	- Token - informacja czy klient wraca do serwera, 
- w kolejnych pakietach: 
	- wszystko jest szyfrowane, nie widać treści, numerów pakietów, ani tego czy to jest retransmisja, 
	- jednak zawsze widać:
		- rozmiar pakietu - bo to znajduje się w nagłówku UDP, 
		- czas wysyłania - UDP ma timestamp, 
		- IP źródłowe oraz docelowe,
		- port - zazwyczaj 443 UDP.

### Sesja w bezstanowym UDP?
Tu znajduje się kluczowa informacja, bo chociaż samo UDP jest bezstanowe to QUIC tworzy sesję po swojej stronie, ma:
- Connection ID, 
- handshake,
- strumienie, 
- zarządzanie połączeniem.   
Dlatego też w logach QUIC można zobaczyć połączenie, nawet jeśli UDP tego nie robi. Z perspektywy SOC - nie należy patrzeć na UDP jako bezstanowe - należy patrzeć na QUIC jako warstwę UDP, która ma stan. 

### Jak monitorować QUIC w praktyce
- logi z firewalla, można w nich zobaczyć:
	- IP źródłowe i docelowe, 
	- port (443 UDP),
	- liczbę pakietów,
	- czas trwania połączenia,
- logi z Zeek lub Suricata pokazują:
	- SNI (nazwę domeny),
	- wersję QUIC,
	- Connection ID,
	- historię handshake, 
- logi z proxy - jeśli jest firmowe proxy QUIC może terminować szyfrowanie, dzięki czemu można zobaczyć HTTP/3 jak zwykły HTTP,
- analiza wolumenu i częstotliwości, gdzie zobaczyć można:
	- ile danych jest przesyłanych, 
	- jak często, 
	- do kogo,   
	To wystarczy żeby wykryć beaconing, eksfiltrację, DDoS. 

### Kiedy warto blokować QUIC
- QUIC warto blokować gdy:
	- nie ma narzędzi do analizy QUIC, 
	- WAF/proxy nie obsługuje HTTP/3, 
	- zachodzi konieczność, by cały ruch był widoczny (DLP, inspekcja),
	- pożądane jest wymuszenie ruchu przez TCP, który łatwiej kontrolować.
	Blokada QUIC powoduje, że przeglądarki wracają do HTTP/2 po TCP.
- Kiedy nie blokować QUIC:
	- WAF/proxy obsługuje QUIC, 
	- potrzebna jest wydajność (mobile, słabe sieci),
	- nie ma powodów do głębokiej inspekcji.

### Kiedy QUIC jest podatny na ataki:
Ataki na QUIC (amplifikacja, flood, 0-RTT) są możliwe ponieważ:
- UDP jest bezstanowe, 
- serwer QUIC musi odpowiadać na pakiety,
- fałszowanie IP jest możliwe.   
Jednak QUIC posiada także własne mechanizmy ochronne:
- tokeny, 
- konieczność potwierdzania przed dużą odpowiedzią, 
- rate limiting,
- szyfrowanie Connection ID.


