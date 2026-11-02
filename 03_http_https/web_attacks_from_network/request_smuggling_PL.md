# request_smuggling

## Cel 
Poznać oraz zrozumieć mechanikę działania ataku Request Smuggling (HTTP Request Smuggling), sposobów jego wykrycia oraz zapobiegania mu. 

## Co to jest
Jest to atak, w którym atakujący wysyła jedno żądanie HTTP, ale dwa serwery po drodze (proxy i backend) widzą je inaczej - każdy widzi inną granicę żądania. W efekcie tego ataku, zostaje "przemycony" fragment żądania wysłanego przez atakującego i backend traktuje go jako początek następnego żądania. 

## Cel
- obejście kontroli dostępu na proxy (dostanie się do endpointów, które proxy blokuje), 
- zatrucie cache - wstrzyknięcie własnej odpowiedzi, którą dostaną inni użytkownicy, 
- przejęcie żądania innego użytkownika - wstrzyknięcie nagłówka do jego sesji, 
- wstrzyknięcie własnej odpowiedzi - np. fałszywy formularz logowania. 

## Działanie ataku 
- HTTP ma dwa sposoby określania, gdzie kończy się żądanie:
	- Content-Length - mówi, że żądanie ma dokładnie X bajtów, 
	- Transfer-Encoding: chunked - mówi, że żądanie jest podzielone na kawałki, a każdy z nich ma własną długość.   
	Więc jeśli w jednym żądaniu występują oba nagłówki jednocześnie, to różne serwery mogą się "nie dogadać" w kwestii, który z nich jest ważniejszy. Proxy może patrzeć na Content-Length - i to właśnie uznać za koniec żądania, a backend może patrzeć na Transfer-Encoding i to on będzie wyznaczał dla niego koniec żądania.
- atakujący wysyła żądanie, np:
	```
	POST / HTTP/1.1
	Host: example.com
	Content-Length: 13
	Tranfser-Encoding: chunked

	0

	GET /admin HTTP/1.1
	Host: example.com
	```
- proxy w takiej sytuacji widzi: `Content-Length: 13` i oblicza 13 bajtów od początku ciała, tam określa koniec żądania i uznaje je za zakończone i przekazuje do backendu. Dla proxy żądanie wygląda tak:
	```
	POST / HTTP/1.1
	Content-Length: 13
	
	0
	```
- co widzi backend: patrzy na `Transfer-Encoding: chunked` i rozumie to ciało inaczej -> jako żądanie podzielone na kawałki. Wg niego żądanie kończy się dopiero po `0` a potem zaczyna się nowe żądanie: 
	```
	GET /admin HTTP/1.1
	Host: example.com
	```
Backend ten fragment żądania widzi jako kolejne żądanie i wykonuje je jako żądanie kolejnego użytkownika, który trafił na to samo połączenie. 
- efekt: atakujący "przemycił" żądanie `GET /admin` - proxy go nie zauważyło, a backend wykonał -> kontrola proxy została ominięta, 
- warianty:
	- CL.TE - proxy używa `Content-Length`, backend używa `Transfer-Encoding`, 
	- TE.CL - proxy używa `Transfer-Encoding`, backend używa `Content-Length`, 
	- TE.TE - oba używają `Transfer-Encoding`, ale jeden ignoruje go z powodu błędu np:  `Transfer-Encoding: xchunked`.

## Jak wykryć 
- w access.log proxy, access.log backendu, WAF, SIEM podczas monitorowania HTTP,
- żądania z jednoczesnym `Content-Length` i `Transfer-Encoding` - normalne żądania mają jeden z tych nagłówków, oba jednocześnie, to sygnał, że ktoś usiłuje wykorzystać różnicę w parsowaniu - warto to sprawdzić, 
- nietypowe wartości `Transfer-Encoding` - poprawna wartość to `chunked`, więc jeśli zamiast tego widać `xchunked`, `chunked, chunked`, `identity, chunked` lub inne kombinacje - to oznacza próbę obejścia parsowania. Niektóre serwery ignorują nieznane wartości a niektóre je akceptują - to właśnie wykorzystują atakujący, 
- niespójność liczby żądań między proxy a backendem - jeśli proxy widzi 100 żądań, a backend 106, to znaczy, że 6 żądań zostało przemyconych. To najbardziej wyraźny sygnał i można go wykryć w SIEM, korelując logi z obu warstw, 
- żądania do wrażliwych endpointów z nietypowych źródeł - jeśli w logach pojawia się `GET /admin` z IP, które wcześniej nie pojawiało się w logach, a proxy tego żądania nie zarejestrowało - to może być przemycone żądanie, 
- powtarzające się dziwne żądania na tym samym połączeniu - Request Smuggling często wymaga wielu prób, bo atakujący nie wie, jak dokładnie parsuje proxy, więc w logach backendu widać całą serię dziwnych żądań z jednego IP. 

## Jak się bronić
- ujednolicenie parsowania HTTP - najskuteczniejsza obrona to proxy i backend, które korzystają z tej samej logiki parsowania. Jeśli oba traktują `Content-Length` i `Transfer-Encoding` w ten sam sposób, atak się nie uda. 
- odrzucanie żądań zawierających oba nagłówki jednocześnie - jeśli żądanie ma oba nagłówki, serwer powinien odrzucić takie żądania albo zignorować `Content-Length`, jeśli jest `Transfer-Encoding`. 
- odrzucać nietypowe wartości `Transfer-Encoding` - akceptować tylko `chunked`, wszystko inne powinno być odrzucane,
- WAF - nowoczesne WAF-y (Cloudflare, AWS WAF) potrafią wykrywać Request Smuggling i blokować żądania z obydwoma nagłówkami. Nie zastępuje to spójnego parsowania, ale stanowi dodatkową warstwę zabezpieczeń, 
- korelacja logów SIEM - ustawienie alertów na różnicę liczby żądań między proxy a backendem z tego samego IP w krótkim czasie, 
- ograniczenie dostępu do backendu - backend powinien przyjmować ruch tylko z proxy, nie bezpośrednio z internetu. Dzięki temu atakujący nie może ominąć proxy i wysyłać żądań bezpośrednio do backendu. 
