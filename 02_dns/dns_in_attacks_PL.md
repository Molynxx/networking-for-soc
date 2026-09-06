# dns_in_attacks

## Cel 
Zrozumienie jakie zagrożenia wynikają z używania serwerów DNS, które są konieczne dla ruchu www, jak je wykryć i jak się przed nimi bronić. 

## DNS TUNNELING
DNS tunnelling to sposób ukrywania danych w protokole DNS, który normalnie służy do tłumaczenia nazw domen na adres IP, co atakujący chętnie wykorzystują. DNS jest rzadko blokowany przez firewalle ponieważ jest konieczny do poruszania się po sieci. 
- atak: DNS tunnelling, 
- Cel: celem tego rodzaju ataku jest sterowanie malware (C2) oraz kradzież danych (eksfiltracja), 
- metoda: atak ten polega na ukrywaniu danych w zapytaniach i odpowiedziach DNS, 
- narzędzie: rekordy A, AAAA, TXT. 

### Jak działa ten typ ataku
- `eksfiltracja (A, AAAA)`:
	- malware na komputerze ofiary wyszukuje dane, np. plik z hasłami, 
	- następnie koduje dane w Base64, żeby uniknąć znaków, które są niedozwolone w domenie. Base64 to sposób kodowania danych, celem zabezpieczenia ich w przesyłanym tekście. Kodowanie to zamienia dowolne dane tekstowe na ciąg znaków, cyfr, liter i znaków `+`, `/`, `=`. Wszystkie znaki tego kodowania są dozwolone w domenach. 
	- malware dzieli zakodowane dane na mniejsze kawałki po ok 50 znaków, żeby zapytanie nie wyglądało na podejrzanie długie, 
	- każdy kawałek wstawia do nazwy subdomeny, o którą pyta DNS, np. aFGzwoJvxed.evil.com, 
	- wysyła normalne zapytanie o adres IP tej domeny do DNS, 
	- resolver przekazuje zapytanie dalej, aż dotrze do serwera evil.com, 
	- serwer atakującego odbiera zapytanie, wyciąga dane z subdomeny i zapisuje dane, 
	- atakujący dzięki temu składa te dane w jedną całość i rozkodowuje uzyskując informacje. 
- `C2 - command & Control (TXT)`:
	- malware umieszczone na urządzeniu ofiary wysyła zapytanie o rekord TXT do evil.com. Choć wygląda to jak zwykłe zapytanie w rzeczywistości jest to zapytanie do atakującego o to co malware ma robić. TXT jest rekordem, w którym można umieścić dowolny tekst przypisany do domeny, Może służyć do przesyłania konkretnych informacji dla innych usług i serwerów. Jednak ten rekord daje też duże możliwości atakującym. 
	- serwer atakującego odpowiada rekordem TXT z zakodowanym poleceniem, 
	- malware odczytuje TXT, dekoduje i wykonuje to polecenie, 
	- całość tej komunikacji wygląda jak normalny ruch DNS, dzięki czemu trudno jest to wykryć na pierwszy rzut oka. 

### Jak rozpoznać
To na co należy zwracać uwagę podczas monitorowania ruchu na DNS to:
- długie subdomeny - normalne domeny nie posiadają długich subdomen, więc jeśli występują w logach długie, losowe subdomeny należy to sprawdzić, 
- duża ilość zapytań TXT z komputerów użytkowników - urządzenia użytkowników nie pytają TXT, pytają o to serwery pocztowe. Więc widoczne zapytanie z komputera użytkownika powinno wzbudzić podejrzenia,  
- beaconing - regularne i powtarzalne odpytywanie serwera C2 przez malware. Zapytania powtarzające się co 60 - 300 sekund powinny naturalnie budzić podejrzenie, ponieważ to wyraźny sygnał, że coś jest nie tak. Normalnie DNS nie dostają od jednego komputera zapytań o tą samą domenę z tak dużą częstotliwością, zwłaszcza jeśli to nie jest dobrze znana domena, 
- nietypowe domeny - atakujący używają nietypowych końcówek jak `.xyz`, `.top` czy `.tk`, ponieważ są tanie i łatwo dostępne, 
- Base64 w subdomenie - czyli długi ciąg przypadkowych znaków. Zakodowany w Base64 tekst nie ma logiki ani sensu, jednak po odkodowaniu daje konkretne i sensowne dane. Jeśli subdomena składa się z przypadkowych znaków, warto sprawdzić czy to na pewno nie jest Base64. 

### Jak się bronić
- należy wymusić resolver (w przypadku sieci firmowych) - wszystkie komputery w firmie powinny korzystać tylko z firmowego serwera DNS. Jest to istotne ze względu na możliwość monitorowania i wykrywania ataków DNS tunnelling, administrator systemu ma do niego logi. Ponadto firmowy resolver daje możliwości blokowania podejrzanych domen oraz ograniczania ruchu (rate limiting), który daje dużo możliwości 
	- rate limiting - ograniczenie liczby zapytań z jednego IP pozwala ustawić takie parametry jak:
		- response-per-second: określa, ile odpowiedzi na sekundę może wysłać serwer do pojedynczego klienta, 
		- window: definiuje okres czasu w sekundach, w którym zliczane są zapytania. Jeśli limit jest ustawiony na np. 15, klient w danym przedziale czasowym nie może wysłać szesnastego w tym oknie czasowym, 
		- burst: pozwala na chwilowy skok powyżej limitu, aby nie zablokować normalnych lecz nagle pojawiających się wzrostów ruchu.   
		Dobrze skonfigurowany rate limiting chroni przed beaconingiem, w przypadku ataku DNS tunnelling. 
- należy blokować na resolverze podejrzane, nietypowe TLD (Top-Level Domain). Typowe domeny to `.com`, `.pl`, `.org`, `.com.pl`, itp. Natomiast nietypowe domeny takie jak `.xyz`, `top`, itp. są chętnie wybierane przez atakujących bo są tanie i łatwe do zdobycia. 
- monitoring DNS - to właśnie to działanie daje możliwości wykrycia podejrzanego ruchu wskazującego na tunnelling, administrator ma pełny dostęp do logów firmowego DNS. Należy zwracać uwagę na długość subdomen, częstotliwość zapytań oraz rekordy TXT. 
- Blokada DoH/DoT: 
	- DoT (DNS over TLS) - ruch pomiędzy klientem a resolverem jest szyfrowany, a to znacząco utrudnia monitoring i wykrywanie tunnellingu. Dlatego w środowiskach produkcyjnych po wymuszeniu firmowego resolvera warto wyłączyć DoT. DoT działa na porcie 853, którego nie dzieli z żadną inną usługą więc łatwo można go zablokować na firewallu, można także skonfigurować resolver tak aby nie odpowiadał na zapytania na porcie 853, 
	- DoH (DNS over HTTPS) - HTTPS to szyfrowany protokół internetowy, wiec zapytania poprzez ten protokół podobnie jak DoT utrudnia monitorowanie ruchu DNS. Jednak tu zabezpieczenie przed tym protokołem jest trudniejsze, bo DoH dzieli port z HTTPS, więc zablokowanie go na firewallu odetnie cały bezpieczny ruch internetowy. Tutaj należy sięgnąć po inne rozwiązania:
		- polityki grupowe (GPO) - narzędzie do zarządzania w domenie Active Directory, dzięki któremu administrator może wymusić ustawienia na wszystkich komputerach w firmie. Jednak ustawienia te dotyczą wyłącznie przeglądarek zintegrowanych z systemem. Więc by metoda była skuteczna trzeba by ograniczyć możliwości instalowania innych przeglądarek niż Chrome czy Edge,
		- modyfikacja rejestru Windows, tak by wyłączyć DoH dla pozostałych przeglądarek. Ta metoda uzupełnia GPO, zamykając możliwość obejścia zabezpieczeń, lecz ma swoje wady. Aby upewnić się, że pozostałe przeglądarki nie będą korzystały z DoH należałoby dodać odpowiedni wpis w rejestrze dla każdej przeglądarki osobno. To czasochłonna, mozolna i obarczona ryzykiem błędu we wpisie do rejestru metoda. Jednak można ją zautomatyzować - odpowiednio napisany skrypt może automatycznie wprowadzić wymagane zmiany w rejestrze. Można także ustawić rejestr na jednym urządzeniu a potem po prostu skopiować plik rejestru na pozostałe (.reg) to ogranicza możliwości popełnienia błędu podczas zmiany rejestru. 
		- należy pamiętać, że GPO dotyczy wyłącznie WINDOWS/AD, a firmy bez AD muszą radzić sobie inaczej, np. MDM. 
- stosowanie alertów - należy ustawić reguły detekcyjne dla SIEM tak aby pokazywały alerty dla rekordów TXT z komputerów użytkowników oraz dla długich subdomen. 

## DGA (Domain Generation algorithm)
Atak ten polega na utrzymaniu komunikacji C2 nawet jeśli zostanie zblokowana domena atakującego. 
- atak: DGA, 
- cel: utrzymanie komunikacji z malware, mimo blokad domen, 
- metoda: generowanie losowych nazw domen, 
- narzędzie: rekord A, AAAA. 

### Jak to działa
- malware posiada w sobie algorytm generujący setki nazw domen, np. `as4awf.evil.com`, `rgfad3.evil.com`, 
- ten algorytm zna atakujący i rejestruje tylko jedną domenę dziennie, 
- w takim przypadku większość domen nie istnieje więc resolver zwraca `NXDOMAIN`,
- jednak jedna z nich istnieje i zwraca adres IP serwera C2, 
- mimo że ofiara zablokuje podejrzaną domenę, to dla atakującego nie ma wielkiego znaczenia bo kolejnego dnia malware używa już innej.

### Jak rozpoznać 
- w logach widać wiele komunikatów NXDOMAIN - jeden komputer w sieci generuje wiele zapytań o nieistniejące domeny, 
- subdomeny wyglądające na zupełnie przypadkowy ciąg znaków, 
- kiedy po serii NXDOMAIN pojawia się odpowiedź, 
- ten sam scenariusz powtarzający się każdego dnia. 

### Jak się bronić
- należy monitorować komunikaty NXDOMAIN, jeśli jeden host pyta wiele razy o nieistniejące domeny, to sygnał, że to potencjalne DGA, 
- należy używać threat intelligence - zbiory informacji o znanych zagrożeniach, np. lista złośliwych IP, domen, wzorców ataków. Dobrą praktyką jest sprawdzanie domen, które pojawiają się w logach właśnie w tych źródłach, 
- należy blokować wzorce DGA - narzędzia bezpieczeństwa (zaawansowane Firewalle DNS, SIEM, platformy zabezpieczeń DNS, Modele ML/AI) potrafią wykryć losowość nazw, 
- DNS sinkhole - technikę defensywną, polegającą na przekierowaniu złośliwego ruchu w bezpieczne miejsce. Mechanizm działania sinkhole:
	- wymaga dostępnej listy złośliwych domen, np. C2, 
	- gdy urządzenie zapyta o taką domenę, serwer DNS (skonfigurowany jako sinkhole) zamiast prawdziwego adresu IP podaje fałszywy (np. 0.0.0.0 lub adres serwera analitycznego). 
	- to pozwala administratorom analizować logi i identyfikować, które urządzenia próbowały łączyć się ze złośliwymi serwerami. Zidentyfikowane zainfekowane urządzenie może wtedy zostać odłączone z sieci na czas usuwania złośliwego oprogramowania. 

## FAST FLUX
To rodzaj ataku bardzo podobny do DGA, ma ten sam cel, jednak nie domeny się zmieniają lecz IP. 
- atak: Fast Flux, 
- cel: utrzymanie serwera C2 mimo blokad IP, 
- metoda: stała domena, rotacja IP i NS - rekord NS to informacja dla resolvera, który DNS jest serwerem autorytatywnym dla danej domeny, 
- narzędzia: A, NS. 

### Jak to działa
- Atakujący ma jakąś domenę, np. evil.com oraz serwer C2, 
- ma także botnet - tysiące zainfekowanych komputerów, 
- na domenie ustawia sobie TTL np. na 60 sek - TTL to jest czas w jakim są przechowywane w pamięci DNS dane o domenie, jeśli w tym czasie zostanie wysłane ponowne zapytanie serwer DNS już nie rozpytuje tylko podaje dane z cache, 
- skutkiem takiego działania jest, że w jednej minucie evil.com wskazuje na IP 1.2.3.4, a po 60 sekundach, kiedy resolver nie ma już w pamięci danych o domenie, odpytuje ponownie i wskazuje na inny adres IP, 
- adresy IP, na które wskazuje domena to komputery z botnetu atakującego, które działają jak proxy dla serwera C2, 
- więc nawet IP 1.2.3.4 zostanie zbanowane, to za minutę ruch pójdzie już przez inne IP.

### Jak rozpoznać
Czynniki wskazujące na potencjalny Fast Flux to:
- ta sama domena lecz różne IP, to nienaturalne kiedy jedna domena co chwilę ma inny adres IP. 
- niskie TTL - krótki rekord zawsze powinien budzić podejrzenia, 
- wysoka częstotliwość zapytań - częściej niż normalnie.

### Jak się bronić
- należy blokować domeny Fast Flux - dodać je do czarnej listy, 
- Monitoring TTL - może wskazać na potencjalnie złośliwe domeny, 
- DNS sinkhole - przechwytywanie zapytań do podejrzanych domen, 
- Threat intelligence - należy korzystać z dostępnych list znanych złośliwych domen. 

## CACHE POISONING
Atak ten polega na zatruciu przez atakującego pamięci podręcznej resolvera, poprzez przekazanie fałszywej odpowiedzi o adres domeny, zanim przyjdzie ona z prawdziwego DNS.
- atak: Cache poisoning, 
- cel: przekierowanie ruchu, kradzież danych, 
- metoda: podrobiona odpowiedź DNS,
- narzędzia: A, AAAA, NS. 

### Jak to działa
- użytkownik w sieci pyta o adres domeny bank.pl, 
- atakujący, który znajduje się w tej samej sieci np. MitM, przechwytuje zapytanie, 
- wysyła fałszywą odpowiedź, zanim nadejdzie prawdziwa z DNS, to trudny wyścig ale wciąż możliwy, ponieważ odpowiedź DNS musi pokonać drogę przez wiele routerów, a atakujący przebywa w tej samej sieci co ofiara, 
- resolver zapisuje dane o domenie w swojej pamięci podręcznej (na czas TTL podany w fałszywej odpowiedzi atakującego). Atakujący może podać dowolny TTL, jednak ma świadomość, że zbyt długi TTL może wywołać alert na programach SIEM. W czasie, w którym dane są przechowywane w pamięci resolvera (TTL), każde zapytanie z sieci o domenę bank.pl otrzymuje tą samą fałszywą odpowiedź, którą wysłał atakujący, 
- użytkownicy wchodzą na fałszywą stronę i podają dane. 

### Jak rozpoznać
Symptomy wskazujące na potencjalny Cache poisoning:
- nagła zmiana IP - SIEM może pokazać alert, ponieważ systemy bezpieczeństwa porównują odpowiedzi DNS z różnych źródeł (np. zewnętrzne resolvery). Jeśli wewnętrzny resolver zwraca inny adres IP dla domeny niż zewnętrzny, generowany jest alert. 
- odpowiedź z nieautorytatywnego serwera - SIEM może to wykryć na trzy sposoby:
	- na podstawie listy zaufanych serwerów z listy IP autorytatywnych serwerów DNS, którą zdefiniuje administrator i jeśli odpowiedź nie przychodzi z IP z tej listy, SIEM generuje alert, 
	- na podstawie niewłaściwej flagi w odpowiedzi (AA) - autorytatywna odpowiedź DNS ma ustawioną flagę AA (Authoritative Answer). Jeśli SIEM zarejestruje odpowiedź bez flagi AA, ale ma fałszywy adres IP, to może wydać alert, 
	- na podstawie korelacji odpowiedzi na zapytania - SIEM może porównać odpowiedź z oczekiwaną (np. z listy zaufanych IP). Jeśli odpowiedź na zapytanie o bank.pl przyszła szybciej niż zwykle przychodzi od autorytatywnego serwera, fakt ten może zostać wykryty. SIEM może również przechwycić nietypowe odpowiedzi z nieprawidłowym kodem błędu (rcode) lub oznaczonego jako sfałszowane (FORGED). 
	- brak DNSSEC - czyli zestaw specyfikacji, które dodają warstwę uwierzytelniania kryptograficznego i integralności danych do protokołu DNS. Jego podstawowe funkcje to:
		- uwierzytelnienie odpowiedzi - dzięki podpisom cyfrowym, resolver DNS może sprawdzić, czy odpowiedź przychodzi rzeczywiście od autorytatywnego serwera dla danej domeny, 
		- zapewnienie integralności - gwarantuje, że dane nie zostały zmodyfikowane podczas przesyłania. 
		- jeśli DNSSEC jest wyłączone, to atak jest możliwy. 

### Jak się bronić
- należy włączyć DNSSEC - dzięki czemu odpowiedzi są kryptograficznie podpisywane, 
- należy losować query ID oraz porty źródłowe - zabezpieczenie, które znacznie utrudnia poisoning. Na czym polega:
	- query ID - to szesnastobitowy numer identyfikacyjny, który pozwala resolverowi dopasować odpowiedź do konkretnego zapytania. Jest umieszczany w nagłówku każdego zapytania i odpowiedzi DNS,
	- port źródłowy - to numer portu, z którego jest wysyłane zapytanie (domyślnie 53) jednak resolvery mogą go losować. Serwer DNS wysyła odpowiedź na ten konkretny port.
	- jak to działa:
		- atakujący, który chce sfałszować odpowiedź DNS, musi trafić nie tylko we właściwy port ale także w poprawny query ID zanim nadjedzie prawdziwa odpowiedź z serwera DNS, 
		- bez losowania, jest to dość łatwe do odgadnięcia bo w starych, prymitywnych resolverach domyślny query ID to 12345 a domyślny port to 53. 
		- z losowaniem to jest praktycznie niemożliwe. 
		- jeśli odpowiedź przyjdzie na niewłaściwy port i lub z niewłaściwym query ID -> resolver odrzuca ją.    
	Należy jednak zaznaczyć, że nowoczesne resolvery losują query ID dla każdego zapytania, losują port źródłowy z szerokiego zakresu, używają losowości kryptograficznej.  
- należy ograniczać ACL resolvera - tylko zaufane serwery mogą pytać resolver. ACL (Access Control List) to lista dozwolonych i zabronionych zapytań dla określonych użytkowników, adresów IP lub procesów. Np. serwer DNS odpowiada wyłącznie na zapytania użytkowników należących do określonej sieci, a zapytania innych są odrzucane, 
- obowiązkowa systematyczna aktualizacja oprogramowania - wszelkie dziury w resolverze to główna droga ataków. 
- ten atak może się udać tylko w przypadkach gdy:
	- resolver jest stary - nie losuje query ID i portów, 
	- MitM - atakujący widzi zapytanie, więc nie musi zgadywać ID i portu, 
	- DNSSEC jest wyłączone, 
	- zła konfiguracja, np. zbyt wąski zakres portów. 

## DNS AMPLIFICATION
To atak polegający na zalaniu ofiary ruchem w celu uniemożliwienia działania usługi, np. strony www. 
- atak: DNS Amplification, 
- cel: DDoS (Distributed Denial of Service) - zalanie ofiary ruchem,
- metoda: amplifikacja (wzmocnienie) - małe zapytanie, duża odpowiedź, 
- narzędzia: ANY, TXT. 

### Jak to działa
- atakujący znajduje otwarty resolver DNS, czyli taki, który odpowiada każdemu, 
- następnie wysyła do niego małe zapytanie, np. ANY (prosi o wszystkie rekordy dla danej nazwy),
- w swoim zapytaniu podaje fałszywy adres źródłowy IP (IP ofiary), 
- resolver generuje dużą odpowiedź i wysyła ją pod sfałszowany adres IP, 
- efektem jest, że na małe zapytanie przychodzi bardzo duża odpowiedź (nawet kilka tysięcy bajtów), 
- przy tysiącach takich zapytań ofiara zostaje zalana ruchem i zablokowana. 

### Jak rozpoznać
- po zapytaniach ANY widniejących w logach - ponieważ normalnie nikt ich nie używa, 
- sfałszowane IP źródłowe - w logach można sprawdzić, czy adres IP odbiorcy zgadza się z adresem źródłowym, 
- duże odpowiedzi na małe zapytania - jeśli odpowiedź jest większa niż zapytanie i scenariusz ten się powtarza, to bardzo mocno wskazuje na amplifikację, 
- ruch z jednego resolvera do jednej ofiary - normalnie to się nie zdarza, sama częstotliwość dużych odpowiedzi do jednego odbiorcy powinna natychmiast budzić podejrzenia. 

### Jak się bronić 
- należy wyłączyć rekurencję dla zewnętrznych użytkowników i pozostawić na rekurencję tylko dla komputerów z lokalnej sieci, np. komputery w biurze. Wszystkie pozostałe zapytania resolver zignoruje. 
- należy ograniczać ANY - w konfiguracji DNS można wyłączyć obsługę zapytań ANY "deny-any",
- dobrą praktyką jest ustawienie rate limiting - ogranicza to liczbę zapytań w ogóle z jednego IP, 
- ważny jest także monitoring wolumenu - każdy nagły skok odpowiedzi to sygnał ataku. 

## DNS RECONNAISSANCE
To atak polegający na zdobywaniu przez atakującego danych firmy takich jak serwery pocztowe, serwery DNS, mail admina, kontroler domeny czy też co kryje się pod danym adresem IP. To są informacje, umożliwiające atakującemu zaplanowanie odpowiedniego ataku. 
- atak DNS reconnaissance, 
- cel: zebranie informacji przed atakiem, 
- metoda: odpytywanie odpowiednich rekordów,
- narzędzia: MX, NS, SOA, SRV, PTR.

### Jak to działa
Atakujący odpytuje rekordy np. dla firma.pl:
- MX? firma.pl - poznaje serwery pocztowe, 
- NS? firma.pl - poznaje serwery DNS, 
- SOA? firma.pl - poznaje adres mail admina, 
- SRV? _ldap._tcp.firma.pl - poznaje kontroler domeny, 
- PTR? IP - sprawdza, co kryje się pod danym adresem.  
- Porządkuje wszystkie powyższe informacje i planuje atak.

### Jak rozpoznać
- MX z komputera użytkownika - zwykłe komputery nie pytają o MX, robią to natomiast serwery pocztowe, 
- zapytania SOA, SRV - zwykli użytkownicy ich nie potrzebują, 
- próby AXFR - polecenie umożliwiające próbę pobrania całej strefy, czyli wszystkich informacji. 

### Jak się bronić 
- należy ograniczać AXFR - transfer strefy tylko dla zaufanych serwerów, 
- ukrywanie wrażliwych danych SOA - nie należy wstawiać adresu admina,
- monitorowanie nietypowych zapytań - należy zwrócić uwagę na alerty na MX, SOA, SRV z dziwnych hostów, 
- stosowanie ACL - pomoże to ograniczyć transfer strefy wyłącznie dla zaufanych serwerów.

## NXDOMAIN ATTACK
To atak, w którym atakujący zalewa resolver DNS wielką liczbą zapytań o nieistniejące domeny. Każde takie zapytanie zwraca NXDOMAIN, a resolver marnuje swoje zasoby, bo nie może odpowiedzieć z cache - musi szukać w hierarchii. 
- atak: NXDOMAIN attack,
- cel: DDoS na resolver DNS - spowolnienie lub całkowite zatrzymanie usługi DNS, 
- metoda: zmęczenie zasobów - zapchanie kolejek zapytań, 
- narzędzia: A, AAAA, TXT (najczęstsze, choć nie ogranicza się tylko do tych rekordów).  
Atak ten różni się od DNS amplification kierunkiem ruchu (NXDOMAIN - ruch do resolvera, amplification - z resolvera do ofiary), celem (NXDOMAIN - zmęczyć resolver, amplification - zalać ofiarę) oraz IP ofiary (NXDOMAIN - prawdziwe, amplification - fałszywe). 

### Jak to działa
- atakujący wysyła dużą ilość zapytań o losowe, nieistniejące domeny, np. fsdfew.xyz.com,
- każde z tych zapytań wymaga od resolvera pełnego przejścia przez hierarchię DNS, 
- resolver nie może udzielić odpowiedzi z cache ponieważ domeny są unikalne, 
- odpowiedzi NXDOMAIN powodują obciążenie resolvera, 
- przy wystarczająco dużej ilości tego typu zapytań resolver ulega przeciążeniu. 

### Jak rozpoznać
- nienaturalnie duża ilość zapytań NXDOMAIN,
- duża ilość losowych subdomen z jednego IP, 
- zapytania o domeny, które nigdy nie istniały. 

### Jak się bronić
- ustawienie rate limit na resolverze - ograniczy ilość zapytań z jednego IP, 
- blokowanie IP generujących masowe NXDOMAIN - pozwoli uniknąć ataku z danego IP (już podejrzanego) w przyszłości, 
- monitorowanie NXDOMAIN i alerty - pozwala wykryć i zablokować szkodliwy adres IP i powstrzymać atak, 
- ograniczenie rekurencji do zaufanych klientów - zamknięcie resolvera na zapytania spoza zaufanej sieci pozwoli ograniczyć atakującemu możliwość tego typu ataku. 
