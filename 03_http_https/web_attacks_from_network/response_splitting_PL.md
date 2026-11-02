# response_splitting

## Cel
Pznanie i zrozumienie mechaniki ataku Response Splitting (HTTP Response Splitting) oraz poznanie sposóbw jego wykrycia i zap[obieganiu mu.

## Co to jest 
To atak, w którym atakujący wstrzykuje znaki CRLF (`\r\n`) do wartości nagłówka HTTP, żeby "rozciąć" odpowiedz serwera i dodać własną. CRLF to znaki, które w protokole HTTP oznaczają koniec linii a podwójny CRLF (`\r\n\r\n`) oznacza koniec nagłówków i początek ciała. Jeśli atakujący wstrzyknie je do nagłówka, serwer uzna, że nagłówki się skończyły i wszystko co, atakujący dopisał dalej potraktuje jako odpowiedź. 

## Cel:
- wstrzyknięcie własnej odpowiedzi - np. fałszywą stronę logowania, komunikat, 
- zatrucie cache - jeśli proxy buforuje odpowiedź, kolejni użytkownicy dostaną wstrzykniętą odpowiedź, 
- ustawienie ciasteczka - np. nadpisać sesję ofiary, 
- wykonanie XSS - wstrzyknięcie `<script>` w ciało odpowiedzi, 
- przekierowanie ofiary na złośliwą stronę. 

## Działanie ataku:
- HTTP to tekst, a nagłówki i ciało oddziela pusta linia (`\r\n\r\n`)
	``` 
	HTTP/1.1 200 OK
	Content-Type: text/html
	Location: /strona

	<html>....</html>
	```
Jeśli aplikacja wstawia dane użytkownika do nagłówka (np. Location) bez sprawdzenia, atakujący może wpisać `\r\n` i zamknąć nagłówki przed czasem, 
- Aplikacja ma funkcję przekierowania, np. `?url=/strona` - aplikacja wykonuje:
	```
	HTTP/1.1 302 Found
	Location: /strona
	```
- atakujacy wpisyhe `\r\n\` w parametr: `?url=/strona%0d%0a%0d%0aHTTP/1.1%20200%20OK%%0d%0aContent-Type:%20text/htm;%0d%0a%0d%0a<html>fałszywastrona</html>` - `%0d%0a` - to zakodowane `\r\n\` (CRRLF), `%20` - zakodowana spacja, 
- serwer wstawia do to do nagłówka bez sprawdzenia:
	```
	HTTP/1.1 302 Found
	Location: /strona

	HTTP/1.1 200 OK
	Content-Type: text/html

	<html>fałszywastrona</html>
	```
- przeglądarka sprawdza pierwszą odpowiedz (`02 Found Location: /strona`) i przekierowuje na `/strona`. Ale w buforze zostaje druga odpowiedź `200 OK` z fałszywą stroną, 
- jeśli proxy buforuje odpowiedzi, druga odpowiedź trafia do cache. Następny użytkownik, który trafi na ten sam URL, dostanie fałszywą stronę, bo atakujący zatruł cache dla innych, 

## Jak wykryć:
- access/log, WAF, proxy cache, SIEM,
- zakodowane CRLF w parametrach - `%0d%0a`, `%0d`, `%0a` w URL to prawie zawsze próba wstrzyknięcia. Zwykły użytkownik nie wpisuje takich znaków w parametry nawet przypadkiem. Jeśli widać w logach takie znaki - należy sprawdzić na jaki endpoint trafiły i czy aplikacja je przetwarza, 
- surowe `\r\n` w żądaniach - czasem atakujący nie kodują znaków, a wysyłają surowe w URL lub nagłówkach. W logach widać to jak wieloliniowe żądanie - co nie zdarza się normalnie, to czerwona flaga, 
- parametry z `HTTP/1.1` lub `HTTP/1.0` w URL - oznacza, że atakujący próbuje dokleić nową odpowiedź. Zwykły użytkownik nie wpisuje w parametry wersji HTTP, 
- parametry z nazwami nagłówków - `Set-Cookie`, `Location`, `Content-Type` w URL to próba wstrzyknięcia nagłówka. Jeśli pokazują się w parametrach, należy sprawdzić, co aplikacja z nimi robi, 
- nietypowe odpowiedzi w cache proxy - jeśli proxy buforuje odpowiedzi i nagle w cache pojawia się fałszywa strona, to może być skutek Response Splitting - wtedy warto porównać cache z rzeczywistą odpowiedzią backendu, 
- korelacja - żądanie z CRLF w połączeniu z dziwną odpowiedzią dla innych może oznaczać zatrucie cache. 
## Jak się bronić:
- walidacja danych przed wstawieniem do nagłówka - to najskuteczniejsza obron. Aplikacje powinny odrzucać dane, w których są zawarte `\r`, `\n` albo ich zakodowane wersje przed wstawieniem do nagłówka. 
- escapowanie / sanityzacja -jeżeli dane muszą trafić do nagłówka, należy usunąć z nich `\r\n`. W wielu frameworkach są wbudowane funkcje, które to robią (np. `header()` w PHp od wersji 5.1.2 blokuje CTFL),
- WAF - nowoczesne WAF blokują payloady z `%0d%0a` i surowym CRLF, stanowią dobre uzupełnienie walidacji, 
- konfiguracja proxy - proxy nie powinno buforować odpowiedzi, które mogą być zatrute, albo normalizować odpowiedzi przed zapisem do cache, 
- aktualizacje - wiele podatności na Response Splitting wynikało z błędów w starych wersjach serwerów i frameworków, dlatego systematyczne aktualizacje łatają część problemów, 
- monitoring i alerty SIEM:
	- `%0d%0a`,  `%0d`, `%0a` w parametrach URL, 
	- `HTTP/1.1` w parametrach, 
	- `Set-Cookie`, `Location` w parametrach, 
	- nietypowe odpowiedzi w cache proxy. 