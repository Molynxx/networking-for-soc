# dns_basics

## Cel: 
Zrozumienie czym jest protokół DNS, jak działa, jaka jest hierarchia i w jaki sposób ułatwia użytkownikom korzystanie z internetu. 

## Czym jest DNS (Domain Name System)
Jest to usługa tłumaczenia nazw domenowych na adresy IP i jest to jego jedyne zadanie. 
- ludziom trudno jest zapamiętać wiele adresów IP oraz serwisów, pod którymi się znajdują, 
- komputery natomiast potrzebują adresu w postaci numerów, żeby móc odnaleźć daną stronę www, 
- tu wkracza DNS, użytkownik pamięta nazwę domeny google.com, co jest dla niego łatwe do zapamiętania, a DNS tłumaczy tę nazwę na adres IP. 

## Dlaczego DNS jest ważny dla SOC
- każde połączenie w internecie rozpoczyna się właśnie od tego protokołu działającego w warstwie aplikacji. Zanim host połączy się z serwerem, najpierw wysyła zapytanie do DNS o adres serwera, 
- DNS jest często wykorzystywany przez atakujących do komunikacji C2, tunelowania danych, amplifikacji,
- DNS jest też jednym z najważniejszych źródeł logów dla analityków SOC. Widząc anomalie w DNS, widzisz atak na bardzo wczesnym etapie.

## Hierarchia DNS 
DNS nie jest po prostu jedną ogromną listą adresów i ich domen, jest podzielony na warstwy, aby rozłożyć obciążenie i odpowiedzialność.

- `Root`:
	- co to jest: to 13 głównych serwerów (klastrów) na świecie
	- co wiedzą: wiedzą, gdzie są serwery TLD (.com, .pl, .org)
	- przykład: odpowiada na pytanie resolvera: "kto obsługuje .com?"
- `TLD`;
	- co to jest: Top Level Domain - jakie obsługuje końcówki domen
	- co wiedzą: wiedzą gdzie są serwery autorytatywne dla domen o danej końcówce 
	- przykład: .com wie gdzie jest google.com
- `Autorytatywny`:
	- co to jest: to serwer konkretnej domeny
	- co wiedzą: znają wszystkie rekordy swojej domeny i podają konkretne IP, 
	- przykład: google.com wie, że www = 142.250.203.206
- `Resolver (rekursywny)`:
	- co to jest: serwer, który pyta w imieniu użytkownika
	- co wiedzą: sam w zasadzie nic nie wie, jego zadaniem jest przejście całej hierarchii by znaleźć odpowiedź
	- przykład: 1.1.1.1 (DNS Cloudflare), 8.8.8.8 (DNS googla)

## Jak działa zapytanie DNS
Gdy użytkownik wpisuje w pasku przeglądarki np. "google.com", zaczyna się cały proces:
- komputer -> resolver (np. 8.8.8.8) pyta: "Jaki jest IP google.com?"
- resolver -> root: "Kto obsługuje .com?"
- root -> TLD .com: "Kto obsługuje google.com?"
- TLD .com -> resolver: "Nie wiem, ale oto adresy serwerów autorytatywnych google.com"
- resolver -> autorytatywny google.com: "Jaki jest adres IP dla google.com?"
- autorytatywny -> resolver: "142.250.203.206 TTL 3600"
- resolver -> komputer użytkownika: "142.250.203.206"
- komputer użytkownika łączy się z adresem IP 142.250.203.206

## Typy DNS
- `Resolver (rekursywny)`:
	- rola: pyta innych w imieniu użytkownika, nie zna odpowiedzi ale wie gdzie ich szukać, przechodzi przez całą hierarchię. To serwer dostawcy internetu lub publiczny (np. google 8.8.8.8)
	- zawartość: cache - zapamiętuje odpowiedzi na czas TTL
		- TTL (Time To Live) - to czas życia rekordu w sekundach, oznacza jak długo  będzie przechowywany adres IP dla domeny. W tym czasie resolver nie będzie pytał, tylko poda dane z pamięci cache. Po upływie tego czasu, resolver znów będzie przechodzić przez całą hierarchię. 
			- Niskie TTL (60-300s) + wysoka częstotliwość zapytań może wskazywać na DGA lub Fast Flux (więcej w pliku `dns_in_attacks_PL.md`),
			- Wysokie TTL (86400s) -> zmiana rekordu będzie długo niewidoczna. Np. w przypadku migracji serwera na inne IP, przez 24 h resolver będzie pamiętał poprzedni rekord (poprzednie IP), więc zamiast połączyć się z nowym adresem będzie łączył na nieaktualny adres. 
- `Autorytatywny`:
	- rola: ma wszystkie odpowiedzi na temat swojej domeny, nie musi nikogo pytać. Root i TLD to szczególne typy serwerów autorytatywnych
	- zawartość: strefa - plik z rekordami domeny  

## Strefa DNS i transfer strefy

### Strefa DNS 
Strefa to plik tekstowy na serwerze autorytatywnym, zawierający wszystkie rekordy domeny.  
Żeby zrozumieć dobrze jak to działa, warto użyć przykładu: Administrator dużej firmy, ma jeden główny segregator (Primary Server DNS) z plikiem Strefa (np. dla domeny mojafirma.com). Co jeśli pierwszy segregator ulegnie zniszczeniu? Lub będzie tak często przeglądany przez wiele osób, że zrobi się gigantyczna kolejka? -> to powód dlaczego zawsze warto mieć kopię. To właśnie jest rola serwerów Primary (Master) i Secondary (Slave).
- `Primary DNS`: to miejsce, w którym administrator może edytować dane, wprowadzać zmiany w pliku strefa. 
- `Secondary DNS`: zawiera plik wyłącznie do odczytu, to jest kopia primary. Administrator nigdy nic na nim nie zmienia. Jego zadaniem jest pobrać kopię strefy od Primary i trzymać ją u siebie. Dzięki temu jest redundancja - czyli jeśli primary padnie, to Secondary odpowiada na pytania, 
	- obciążenie jest rozłożone - pytania klientów mogą trafiać zarówno na primary jak i Secondary.

### Transfer strefy
To proces, w którym Secondary DNS kopiuje dane z Primary DNS. Najpierw jednak należy zrozumieć czym jest numer seryjny. To liczba na początku pliku określająca wersję danych. Za każdym razem podczas edycji administrator ręcznie inkrementuje ją o 1. Secondary co jakieś czas pyta Primary "Jaki jest aktualny numer seryjny?". Jeśli Primary ma większy numer seryjny, Secondary wie, że jego dane są nieaktualne i musi zrobić transfer.  
Transfer może odbywać się na dwa sposoby:
- AXFR (Full Zone Transfer) - gdzie cały plik jest ściągany od nowa linijka po linijce. To była pierwsza i przez długi okres jedyna metoda, lecz chociaż jest prosta to przy większych strefach jest nieefektywna, bo po co pobierać np. 10 000 rekordów jeśli zmienił się tylko jeden, 
- IXFR (Incremental Zone Transfer) - pobierane są wyłącznie zmiany. Jest to znacznie szybsze i oszczędzające łącze, może zarówno dodawać jak i usuwać rekordy. Jest jednak warunek, Primary musi być na tyle "inteligentny", by pamiętać historię zmian. 
- SOC perspective: AXFR powinien być dozwolony tylko pomiędzy Primary a Secondary, więc zapytanie tego AXFR z zewnętrznego IP to zawsze alert, oznacza że atakujący próbuje pobrać wszystkie rekordy domeny (reconnaissance)

## Forward DNS vs Reverse DNS
- Forward DNS - tłumaczy nazwę domeny na adres IP. Np. "Jaki jest adres example.com?" -> w odpowiedzi DNS podaje adres IP np. 93.184.216.34. To zapytanie występuje zawsze gdy użytkownik otwiera stronę. 
- Reverse DNS - odwraca proces, tłumaczy adres IP na nazwę domeny, np "Jaka domena kryje się pod adresem 93.184.216.34?". Serwer odpowiada np. example.com. Reverse DNS jest używany do weryfikacji, do sprawdzenia czy IP jest rzeczywiście tym za kogo się podaje.  
Przykład:  
Jak to działa na poczcie mailowej, pomagając ocenić czy wiadomość, która przyszła jest spamem czy nie. Użytkownik dostaje maila z serwera 93.184.216.34. Serwer odbiorcy wykonuje kolejno:
- Reverse DNS: "Jaka nazwa jest przypisana do IP 93.184.216.34?"
- Odpowiedź: "mail.example.com"
- Forward DNS: "Jaki IP ma mail.example.com"
- Odpowiedź: "93.184.216.34"
- Sprawdzenie: Czy IP z forward DNS zgadza się z IP, które wysłało maila?
	- jeśli tak - to wszystko jest w porządku i mail ląduje w skrzynce odbiorczej a nie w spamie,
	- jeśli nie - wiadomość zostaje uznana za spam, mail odrzucony.   

## Reverse DNS w SOC
Analityk SOC widzi alert: "Host 192.168.1.100 łączy się z IP 185.220.101.34". Ta sytuacja zawsze jest warta sprawdzenia:
- Reverse DNS: "jaka nazwa jest przypisana do IP 185.220.101.34?"
- odpowiedź: "tor-exit-node.torproject.org"
- wniosek: Ruch przez sieć TOR - to nie jest normalne zachowanie, żaden legalny program nie używa TOR-a bez wyraźnej woli użytkownika.
- zalecane zablokowanie na firewall podejrzanego IP.   
Inna sytuacja:
- odpowiedź reverse DNS: "c2-malware.badguy.xyz"
- wniosek: dość oczywisty, C2, malware, to nie jest normalny ruch, a wielka czerwona flaga. 
Oba powyższe przypadki mogą wskazywać na reverse shell lub C2, w obu tych przypadkach zalecane jest zablokowanie na firewall podejrzanego IP. 

## Rekordy DNS
Każda domena ma w swojej strefie różne rodzaje rekordów. To nie są wyłącznie rekordy "nazwa -> IP", ale także informacje o poczcie, serwerach i usługach. Każdy typ rekordu ma własne literowe oznaczenie, które mówi co dokładnie przechowuje dany wpis i jak ma odpowiadać na zapytania. 

### Rekord A (Address)
- cel: tłumaczy nazwę na IPv4
- Przykład: example.com -> 93.184.216.34
- SOC: To najczęstszy typ zapytania, zupełnie normalny ruch. Podejrzane w przypadku, gdy pojawia się bardzo dużo zapytań A do jednej domeny (może wskazywać na beaconing - malware na zainfekowanym urządzeniu regularnie (np. co 60 lub 120 sekund) pyta o serwer C2).

### Rekord AAAA (Address dla IPv6)
- cel: tłumaczy nazwę na adres IPv6
- przykład: example.com -> 2606:2800:220:1:248:1893:25c8:1946,
- SOC: IPv6 jest rzadziej monitorowany. Malware czasem używa go, żeby ominąć filtry, które sprawdzają tylko IPv4.

### Rekord CNAME (Canonical Name)
- cel: alias - przekierowanie jednej nazwy na drugą, np. jeśli ktoś wpisze adres bez "www"
- przykład: example.com -> www.example.com
- działanie: gdy użytkownik pyta o example.com, dostaje odpowiedź: "To jest alias dla www.example.com. Zapytaj jeszcze raz o www.example.com"
- SOC: łańcuchy CNAME mogą być długie i prowadzić do ukrytych domen. Atakujący może mieć wiele domen i gdy jedna zostanie zablokowana, jego własny DNS mówi: "Nie szukaj mnie, po prostu idź pod ten adres". To alias dla złośliwej strony, który zapewnia ciągłość jej działania. Dlatego też należy weryfikować gdzie ostatecznie prowadzi CNAME, np. za pomocą polecenia: `dig zły.link +trace`.

### Rekord MX (Mail Exchanger)
- cel: wskazuje serwer pocztowy domeny
- przykład: example.com MX 10 mail.example.com (10 to priorytet serwera pocztowego - im niższa liczba, tym wyższy priorytet. Liczba ta mówi klientowi wysyłającemu maila, do którego serwera ma próbować dostarczyć wiadomość w pierwszej kolejności).
- SOC: 
	- host, który nigdy nie wysyła maili, a nagle wysyła zapytanie MX to zachowanie podejrzane (może przygotowywać phishing). Prawdopodobnie komputer został zainfekowany przez malware, które pyta o rekordy maila, np. banku ofiary. To pozwala mu poznać na jakich serwerach bank ma konta mailowe, co pozwala ocenić zabezpieczenia i dostosować mail phishingowy tak aby wyglądał na prawdziwą korespondencję z banku. 
	- malware na komputerze ofiary zebrało wszystkie maile i kontakty z książki kontaktów ofiary, a teraz wysyła zapytanie MX, podczas wysyłania maila ze skradzionymi danymi na własny adres mailowy.   
	- należy pamiętać, że sam komputer użytkownika nie wysyła maili bezpośrednio. Używa aplikacji (np. Outlook, przeglądarka), która łączy się z firmowym serwerem poczty. To serwer poczty a nie IP komputera użytkownika, pyta DNS o rekord MX adresata. Więc jeśli w logach pojawia się, że IP komputera użytkownika pyta o rekord MX to zawsze jest podejrzane i najprawdopodobniej to malware. 

### Rekord NS (Name Server)
- cel: wskazuje serwery autorytatywne domeny
- przykład: example.com NS ns1.example.com
- SOC: zmiana rekordu NS na nietypowy serwer może oznaczać przejęcie domeny (domain hijacking). Atakujący mógł włamać się na konto właściciela domeny (np. w panelu hostingowym domeny) i zmienić rekord NS z legalnego na swój własny. 

### Rekord TXT (Text)
- cel: przechowuje dowolny tekst związany z domeną
- legalne przykłady użycia:
	- SPF: `v=spf1 include:_spf.google.com -all` - określa, które serwery mogą wysyłać maile z tej domeny
	- DKIM: klucz publiczny do weryfikacji podpisów maili, 
	- DMARC: polityka co zrobić z mailami, które nie przejdą SPF/DKIM
	- weryfikacja domeny: Google, Microsoft każą dodać konkretny tekst żeby udowodnić, że użytkownik jest właścicielem domeny
- SOC: rekord TXT może być nadużywany do DNS tunnellingu (więcej w pliku dns_in_attacks_PL.md). 

### Rekord SOA (Start Of Authority)
- cel: znajdują się tu informacje administracyjne o strefie
- zawiera: 
	- Primary NS (główny serwer autorytatywny)
	- Email administratora
	- Numer seryjny (do synchronizacji primary-secondary)
	- Timery: refresh, retry, expire, minimum TTL
- SOC: SOA zdradza adres mailowy admina, więc atakujący może to wykorzystać do socjotechniki lub prób włamania się na konto admina. Warto więc upewnić się, że hasło jest bezpieczne, a polityki haseł dobrze ustawione. 

### Rekord PTR (Pointer)
- cel: reverse DNS - IP->NAZWA (odwrotność A/AAAA)
- przykład: 34.216.184.93.in-addr.arpa -> example.com
- SOC: omówiony w sekcji Forward vs Reverse DNS

### Rekord SRV (Service)
- cel: wskazuje gdzie działa konkretna usługa
- przykład: `_ldap._tcp.example.com SRV 10 5 389 dc1.example.com`
- SOC: Zapytania SRV zdradzają jakie usługi działają w firmie (VoIP, LDAP, chat). Np. z powyższego przykładu można dowiedzieć się, że w firmie działa usługa LDAP na example.com jest na serwerze dc1.example.com, na porcie 389. Atakujący więc często używają go do reconnaissance przed atakiem, by wiedzieć jakich usług i na jakim porcie szukać. Zapytanie SRV z komputera, który nie jest serwerem i nie ma do tego prawa, to prawie zawsze ktoś, kto szuka drogi do krytycznych zasobów firmy. 

### Rekord ANY
- cel: żądanie wszystkich rekordów domeny na raz
- SOC: normalny ruch prawie nigdy nie używa ANY. Zapytania ANY to są z dużym prawdopodobieństwem amplifikacja DNS. Dlatego wielu administratorów wyłącza tę opcję na swoich serwerach DNS. Więc jeśli widać w logach zapytanie ANY to oznacza, albo dany DNS jest źle skonfigurowany, albo jest celowo ustawiony przez atakującego, by służył jako wzmacniacz w ataku DDoS.

### Rekord CAA (Certification Authority Authorization)
- cel: określa, które CA mogą wydać certyfikat SSL dla domeny 
- przykład: `example.com CAA 0 issue "letsencrypt.org`
- SOC: atakujący sprawdzają CAA żeby znaleźć słabe punkty - jeśli domena nie ma CAA, mogą spróbować wygenerować certyfikat przez nieautoryzowane CA.
 
## DNS OVER HTTPS (DoH) i DNS OVER TLS (DoT)
DNS działa na porcie 53, nie ma szyfrowania, więc każdy może zobaczyć jakie strony odwiedza użytkownik. To w środowiskach produkcyjnych pozwala na dokładne monitorowanie zagrożeń. Problem dla monitoringu stanowi jednak DoH i DoT. 
- DoT (DNS over TLS)
	- port: 853
	- DNS jest szyfrowany przez TLS
	- nie widać w logach zapytań DNS
	- łatwy do zablokowania, można po prostu zablokować na firewall port 853
- DoH (DNS over HTTPS)
	- port: 443
	- DNS ukryty w ruchu HTTPS, szyfrowany
	- nie widać w logach zapytań DNS,
	- malware używa DoH do komunikacji C2 i omija firmowe resolvery
	- trudny do wyłączenia i wymaga kilku kroków od admina:
		- wyłączenie DoH dla dozwolonych w firmie przeglądarek 
		- wyłączenie DoH dla Windows
		- zablokowanie przeglądarek innych niż dozwolone
		- zablokowanie znanych IP DoH (znanych serwerów DNS)
		- wymuszenie firmowego DNS (przy warunku, że DoH jest na nim wyłączony)
		- po wykonaniu tych kroków ruch dla SOC powinien być w pełni widoczny. Pominięcie któregoś z tych kroków niesie ryzyko, że blokada DoH lub wymuszenie DNS firmowego zostanie pominięte. 

## Narzędzia DNS

### dig (Domain Information Groper)
To najpotężniejsze narzędzie, używane do debugowania i analizy DNS. Za pomocą ręcznych zapytań do serwerów DNS można sprawdzić co dokładnie odpowiada:
- dig example.com A -> zapytanie o rekord A
- dig example.com MX -> zapytanie o serwery pocztowe
- dig example.com ANY -> zapytanie o wszystkie rekordy
- dig -x 93.184.216.34 -> reverse DNS
- dig AXFR example,com @ns1 -> próba transferu strefy
- +short example.com -> tylko odpowiedź, bez dodatkowych informacji
- dig example.com +trace -> pokazuje całą drogę zapytania DNS od roota, przez TLD do autorytatywnego (np. gdy zachodzi potrzeba prześledzenia pełnego łańcucha CNAME)

### nslookup
Proste narzędzie wbudowane w system (Windows, Linux) do ręcznego odpytywania serwerów DNS. Działa podobnie do polecenia dig, lecz jest mniej szczegółowe.
- nslookup example.com -> odpowiada z jakiego serwera DNS skorzystano i jakie IP ma domena example.com
- nslookup -type=MX example.com -> gdzie domena odbiera maile
- nslookup -type=NS example.com -> kto obsługuje DNS tej domeny
- nslookup example.com 1.1.1.1 -> pyta serwer Cloudflare zamiast domyślnego 
- nslookup -type=TXT example.com -> jakie serwery mogą legalnie wysyłać maile z tej domeny

### host
To proste konsolowe zapytanie do DNS. Jest bardzo proste, zwraca tylko najważniejsze informacje, bez zbędnych szczegółów.
- host example.com -> odpowiada adresem IP domeny
- host 185.220.101.34 -> odpowiada nazwą domeny, można tutaj wychwycić domeny podszywające się pod IP
- host -t MX example.com -> rekord MX, gdzie znajdują się serwery poczty
- host -t NS example.com -> gdzie znajdują się serwery autorytatywne 

## Normalny vs podejrzany ruch DNS - PODSUMOWANIE
- Normalny ruch DNS:
	- głównie zapytania A/AAAA
	- małe pakiety poniżej 100 bajtów
	- regularne, przewidywalne wzorce
	- ruch na porcie 53 UDP
	- zapytania do popularnych domen (np. google.com, microsoft.com, itp.)
- Podejrzany ruch DNS:
	- duże pakiety w logach - DNS tunnelling
	- długie odpowiedzi TXT - DNS tunnelling (dane w rekordzie TXT)
	- wysoka częstotliwość zapytań do jednej domeny - Beaconing (C2)
	- losowo wyglądające subdomeny (a8f3b.example.com) - DGA (Domain Generation Algorithm)
	- zapytania ANY - amplifikacja DNS
	- zapytania AXFR z zewnętrznych IP - reconnaissance
	- host pyta o MX, a nigdy nie wysyła maili - przygotowanie do phishingu
	- niskie TTL (30-60s) + wysoka częstotliwość - Fast Flux C2 (odpowiedź jest krótko na resolverze, więc atakujący może często zmieniać IP aby ominąć blokady, zanim stare IP zostanie zablokowane)
	- ruch DNS na porcie 443 (DoH) zamiast na 53.