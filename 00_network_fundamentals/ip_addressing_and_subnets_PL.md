# ip_addressing_and_subnets

## Cel
Zrozumienie jak działa adresowanie w sieciach, jak obliczyć ilość hostów, jakie są adresy zarezerwowane dla sieci oraz broadcast (rozgłoszeniowe), czym jest gateway (brama) oraz czym są podsieci i jakie są powody podziału sieci na mniejsze podsieci. 

## Czym jest adres IP
Adres IP sieci to 4 liczby (oktety) oddzielone kropkami, np. `192.168.1.10`. Każda z tych 4 liczb zajmuje 1 bajt, a 1 bajt to 8 bitów (0-255). Adres ten składa się z dwóch części:
- część sieciowa, która identyfikuje sieć,
- część hosta, która identyfikuje konkretne urządzenie w sieci (stacja robocza, drukarka, itp).   
Która część jest która określa maska podsieci. 

## Maska podsieci  
Maska podsieci określa, która część adresu identyfikuje sieć, a która konkretne urządzenia. Rozróżnić to można na podstawie wartości oktetu:
- 255 oznacza, że ta część adresu jest zarezerwowana dla sieci, 
- 0 - oznacza, że ta część adresu jest zarezerwowana dla hosta, 
- inna wartość np 128 oznacza, że sieć jest podzielona na mniejsze sieci.   
Jednak najłatwiej to zrozumieć rozpisując wartości maski na system binarny:
- bity o wartości 1 oznaczają bity przeznaczone na sieć,
- bity o wartości 0 są przeznaczone dla hosta.   

Przykład 1:
```
Maska podsieci w zapisie dziesiętnym: 255.255.255.0
Maska podsieci w zapisie binarnym: 11111111.11111111.11111111.00000000
Stąd wynika, że dla sieci przeznaczone jest 24 bity (jedynki) oraz 8 bitów na adres.
```
Przykład 2: 
``` 
Maska podsieci w zapisie dziesiętnym: 255.255.128.0
Maska podsieci w zapisie binarnym: 11111111.11111111.10000000.00000000
Czyli: 17 bitów dla sieci oraz 15 dla hosta.
```
Przykład 3:
```
Maska podsieci w zapisie dziesiętnym: 255.248.0.0
Maska podsieci w zapisie binarnym: 11111111.11111000.00000000.00000000
Dla sieci 13 bitów, dla hosta 19 bitów. 
```
Obecnie celem uproszczenia zapisu maski podsieci stosuje się zapis CIDR. CIDR to zapis maski w postaci liczby bitów (od 0 do 32), które mówi, która część adresu IP jest stała (sieć), a które zmienna (hosty):
``` 
255.0.0.0 ->/8 	  255.255.0.0 ->/16    255.255.255.0 ->/24 	  255.255.255.255 ->/32
254.0.0.0 ->/7 	  255.254.0.0 ->/15	   255.255.254.0 ->/23    255.255.255.254 ->/31
252.0.0.0 ->/6 	  255.252.0.0 ->/14	   255.255.252.0 ->/22    255.255.255.252 ->/30
248.0.0.0 ->/5 	  255.248.0.0 ->/13	   255.255.248.0 ->/21    255.255.255.248 ->/29
240.0.0.0 ->/4 	  255.240.0.0 ->/12	   255.255.240.0 ->/20    255.255.255.240 ->/28
224.0.0.0 ->/3 	  255.224.0.0 ->/11	   255.255.224.0 ->/19    255.255.255.224 ->/27
192.0.0.0 ->/2 	  255.192.0.0 ->/10	   255.255.192.0 ->/18    255.255.255.192 ->/26
128.0.0.0 ->/1 	  255.128.0.0 ->/9	   255.255.128.0 ->/17    255.255.255.128 ->/25
```
W teorii nie ma sztywno określonej minimalnej liczby bitów sieci, można użyć maski `/0` lub `/1`, lecz w praktyce:
- w sieciach lokalnych (LAN) nie używa się masek mniejszych niż `/8`,
- najmniejsza użyteczna maska do łączenia dwóch urządzeń to `/30`, która daje 4 adresy łącznie w tym 2 dla hostów. Jest zwykle używana do łączeń point-to-point miedzy routerami. 
- maski poniżej `/8` to ogromne sieci, które są niepraktyczne ze względu na:
	- bezpieczeństwo - jedna wielka sieć to jeden wielki obszar ataku. Nie można jej podzielić na mniejsze,
	- wydajność - ruch broadcast zalewa całą sieć,
	- zarządzanie - w dużej sieci trudniej znaleźć błędy i awarie.  

Dzięki zastosowaniu odpowiednich masek można podzielić jedną sieć na kilka mniejszych (subnetting), np. żeby każdy dział (IT, księgowość, itp) miał swoją sieć odizolowaną od sieci innych działów. To właśnie maska określa, na ile części dzielą się bajty adresu:
- `128` - 2 części sieci, każda po 126 hostów, 
- `192` - 4 części, każda po 62 hostów, 
- `224` - 8 części, każda po 30 hostów, 
- `240` - 16 części, każda po 14 hostów, 
- `248` - 32 części, każda po 6 hostów, 
- `252` - 64 części, każda po 2 hosty.  
Im większa liczba w masce tym więcej podsieci i mniej hostów w każdej z nich. Podział ten w zasadzie można wprowadzić w każdym bajcie:
- 255.255.255.128,
- 255.255.192.0,
- 255.224.0.0,
- 128.0.0.0.
Każdy powyższy podział jest możliwy jednak rzadko stosuje się go w 1 bajcie, ponieważ np. 128.0.0.0 to byłyby dwie ogromne sieci, taki podział stosuje się zwykle tylko w sieciach eksperymentalnych.  
UWAGA: kiedyś stosowano klasy adresów zamiast CIDR, obecnie niestosowany jednak warto go znać:

| Klasa | Pierwszy oktet | Domyślna maska | CIDR |
|-------|----------------|----------------|------|
| A | 1-126 | 255.0.0.0 | /8 |
| B | 128-191 | 255.255.0.0 | /16 |
| C | 192-223 | 255.255.255.0 | /24 |

## Obliczanie adresów i hostów 
Należy pamiętać, że w każdej sieci lub podsieci występują dwa adresy, których nie można przypisać hostom, należy je wyjąć z puli adresów hostów:
- adres sieci - pierwszy adres w sieci, np 192.168.0.0 przy masce `/24` to identyfikator sieci, 
- broadcast - ostatni adres (192.168.0.255) to adres rozgłoszeniowy, na którym komunikują (ogłaszają, pytają o adres MAC, odpowiadają) się urządzenia.  
Na obliczenie ilości adresów hostów w sieci jest prosty wzór:
- Bity hosta = 32 (wszystkie bity adresu) - CIDR (bity sieci), 
- łącznie adresów = 2^bity hosta, 
- hostów = 2^bity hosta - 2 (dwa bity - adres sieci i adres rozgłoszeniowy).   
Przykład: 

| CIDR | Adresów | Hostów |
|------|---------|--------|
| /24 | 256 | 254 |
| /25 | 128 | 126 |
| /26 | 64 | 62 |
| /27 | 32 | 30 |
| /28 | 16 | 14 |

## Adresy publiczne vs prywatne 
- Adres publiczny to unikalny adres na cały świat, widoczny w Internecie, dostarczony od dostawcy internetu,
- adres prywatny - to wewnętrzny adres sieci, nie jest widoczny w internecie (np. 192.168.0.0).

## Gateway 
Jest to brama - router, przez który można wejść do internetu lub innej sieci. Router zawsze ma adres hosta sieci, nie jest on z góry narzucony jednak w praktyce nadaje się mu pierwszy adres hosta dostępny w danej sieci, np. 192.168.0.1.