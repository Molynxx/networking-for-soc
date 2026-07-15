# tcp_vs_udp_in_attacks

## Cel
Zrozumienie różnic pomiędzy protokołami TCP i UDP, oraz przyczyn dlaczego atakujący wybierają właśnie te protokoły. 

## TCP vs UDP - perspektywa atakującego
Atakujący zwykle starannie wybiera protokół, ze względu na cele ataku oraz ograniczenia obrony. W atakach sieciowych protokoły transportowe (TCP/UDP) mogą być albo celem ataku (np. SYN flood uderza w mechanizm handshake TCP), albo tylko nośnikiem dla ataku na wyższej warstwie (np. reverse shell używa TCP do komunikacji, ale sam atak dotyczy aplikacji). 

### Niezawodność i szybkość 
Te dwie cechy nie idą w parze, tu konieczne są kompromisy albo niezawodnie, albo szybko. 
- TCP: 
	- gwarantuje niezawodność, wszystkie dane muszą dotrzeć do odbiorcy w odpowiedniej kolejności i bez duplikacji. Jednak ma zabezpieczenia: Handshake, kontrolę przeciążenia, retransmisję jeśli coś zaginie.
	- nadaje się do ataków, w których trzeba mieć pewność, że dane dotarły do celu, np. reverse shell, C2, exfiltracja. Jednak to jest wolniejsze i łatwiejsze do śledzenia przez analityka - ponieważ jest stanowe.
- UDP: 
	- szybki, bez gwarancji dostarczenia, bez pilnowania kolejności, bez kontroli przepływu, czyli po prostu "wyślij i zapomnij". 
	- dobry do ataków tam gdzie liczy się szybkość i wolumen (objętość danych / Gbps), np: DDoS, amplification lub w przypadkach, w których atakujący chce uniknąć wykrycia - UDP jest bezstanowy, to jest trudniejsze do logowania.

### Stanowość 
- TCP:
	- to protokół stanowy, serwer śledzi połączenia (SYN, ESTABLISHED, FIN). Blue Team może monitorować stan połączeń, gdyż atakujący musi utrzymywać stan, a to zwiększa jego ślad.
	- protokół TCP jest wybierany w przypadku, gdy atakujący chce się ukryć w normalnym ruchu, np: C2 przez HTTPS na porcie 443, który zawsze używa TCP. Taki atak nie rzuca się w oczy analitykowi. 
- UDP:
	- to protokół bezstanowy, serwer nie śledzi połączeń UDP bo nie ma sesji. Każdy pakiet jest niezależny, a zespołowi Blue Team jest trudniej odróżnić normalny ruch od ataku. 
	- atakujący wybiera ten protokół, kiedy chce ominąć firewalle stanowe, uważnie śledzące sesje TCP, lecz UDP traktują bardziej wyrozumiale. 

### Amplifikacja 
- TCP:
	- ten protokół mówi stanowcze "nie" dla amplifikacji. Ten atak jest niemożliwy na tym protokole, ponieważ handshake uniemożliwia SYN-ACK przy sfałszowanym IP (IP spoofing). 
- UDP:
	- protokół nie ma zabezpieczeń, nie jest odporny na sfałszowane IP, co umożliwia amplifikację. Atakujący wysyła małe zapytanie do serwera DNS albo NTP, podając sfałszowany adres IP, a ofiara otrzymuje duże odpowiedzi, co wyczerpuje możliwości jej łącza internetowego. DDoS amplifikacyjne to zawsze UDP. 

### DDoS wolumetryczny (volumetric)
To inny rodzaj ataku niż amplifikacja, jednak cel ataku jest podobny, wyczerpują zasoby ofiary. W tym ataku wysyłana jest ogromna ilość pakietów, które w sumie zapychają dostępne pasmo ofiary. 
- TCP:
	- źle skonfigurowany firewall, wyłączone SYN cookies, brak rate limiting, SYN proxy, sprawia, że TCP jest wrażliwy na atak SYN flood.
- UDP:
	- brak odpowiedniej konfiguracji, brak rate limiting, uwrażliwiają protokół na ataki UDP flood. 

### Omijanie Firewalla
- TCP:
	- Firewalle stanowe analizują flagi TCP i śledzą stan połączeń, co utrudnia atakującemu ominięcie ich bez wzbudzenia podejrzeń. Jednak sama analiza stanu to za mało - potrzebny jest IDS/IPS (systemy wykrywające i blokujące ataki) do wykrycia anomalii w ruchu.
- UDP:
	- Często firewalle są skonfigurowane mniej restrykcyjnie dla UDP, ze względu na DNS, NTP, streaming, które korzystają z tego protokołu. Atakujący wykorzystują UDP do tunelowania (np. DNS tunneling), bo port 53 jest zwykle otwarty na firewallach. 

## Przykładowe ataki i ich protokoły

| Atak | Protokół | Dlaczego ten protokół |
|------|----------|-----------------------|
| SYN flood | TCP | Wykorzystuje handshake - wysyła masę SYN, lecz nie kończy handshake |
| UDP flood | UDP | Szybko, bezstanowy, trudniejszy do odfiltrowania |
| DNS amplification | UDP | Amplifikacja możliwa jest tylko w UDP, wykorzystuje fałszowanie IP|
|Reverse shell | TCP | Potrzebuje niezawodności, utrzymuje interaktywną sesję |
| C2 beaconing | TCP (często HTTPS) | Ukrywa się w normalnym ruchu, wymaga niezawodności | DNS tunneling | UDP (port 53) | Firewalle często przepuszczają DNS, trudno blokować |
| TCP reset attack | TCP | Wykorzystuje flagę RST i przewidywanie numerów sekwencyjnych |
| Session hijacking | TCP | Przejmuje istniejącą sesję TCP, wymaga przewidzenia SEQ/ACK |

## Podsumowanie 
Ten plik miał na celu porównanie obu protokołów z perspektywy atakującego. Szczegółowa analiza konkretnych ataków znajduje się w plikach:
- ataki na TCP i UDP: opisane są w `tcp_handshake_and_flags_PL.md` i `udp_basics_PL.md`, 
- ataki wykorzystujące DNS: omówione są w folderze `02_dns`, 
- ataki aplikacyjne (C2, Exfiltracja, reverse shell): omówione są w folderze `06_network_attacks_and_detection`.

