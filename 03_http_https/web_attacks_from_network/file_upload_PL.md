# file_upload

## Cel 
Poznanie mechaniki działania ataku File Upload oraz sposobów wykrywania i obrony przed nim. 

## Co to jest
Jest to atak, w którym atakujący wgrywa na serwer złośliwy plik (np. skrypt PHP, JSP, ASP), a następnie uruchamia go przez przeglądarkę. 

## Cel:
- wgranie webshell - skrypt pozwalający uruchamiać polecenia przez przeglądarkę, 
- uzyskanie dostęp do wykonania kodu (RCE), 
- przejęcie kontroli nad serwerem, 
- utrzymanie dostępu (persistence) - webshell zostaje na serwerze, 

## Działanie ataku:
- aplikacja ma funkcję uploadu - np. "zmień awatar", "dodaj zdjęcie", "dodaj załącznik",
- serwer przyjmuje plik i zapisuje go w katalogu (np. `/uploads`), 
- zwykły użytkownik wgrywa `zdjęcie.jpeg`, 
- atakujący wygrywa plik `shell.php`, 
- jeśli serwer nie sprawdza typu pliku, zapisuje `shell.php` w `/uploads`,
- jeśli serwer sprawdza rozszerzenia, atakujący może spróbować oszukać serwer"
	- zmieniając nazwę `shell.php` na `shell.jpeg`, 
	- używając podwójnego rozszerzenia `shell.php.jpeg`,
	- wgrywając plik z zagnieżdżonym kodem PHP w metadanych obrazu, 
	- wykorzystując błąd w alidacji (np. sprawdzanie tylko `Content-Type`),
- atakujący może także w nazwie pliku użyć `../`, by zapisać plik poza katalogiem uploads - np w katalogu, z którego aplikacja wykonuje pliki, 
- atakujący wchodzi na `htpps://example.com/uploads/shell.php?cmd=whoami`, 
- serwer wykonuje polecenie `whoami` i zwraca wynik, 
- jeśli ten atak zakończy się powodzeniem atakujący ma możliwość zdalnego wykonywania poleceń. 

## Jak wykryć:
- w logach serwera: `POST /upload HTTP/1.1" 200` - jednak to nie pokazuje nazwy pliku, wiec sam log nie mówi nic, 
- logi aplikacji - tutaj nazwa, rozszerzenie i typ MIME będzie widoczny, należy szukać podejrzanych rozszerzeń i nazw,
- WAF - payload - nazwa pliku i jego zawartość będzie widoczna, jeśli WAF skanuje upload, podobnie jak wyżej, należy szukać czerwonych flag w nazwach i zawartości plików, 
- system plików - nowe pliki w `/uploads` z podejrzanymi rozszerzeniami jak `.php`, `jsp`, `.asp`, itp,
- File Integrity Monitoring (AIDE, Tropwire) - dobrą praktyką jest ustawienie alertów o nowych plikach w katalogach, gdzie nie powinno ich być, 
- czerwone flagi:
	- plik `.php`, `jsp`, `.asp`, `.phtml`  w `/uploads` - webshell, 
	- podwójne rozszerzenia jak np. `shell.php.jpeg`, `shell.joeg.php`, 
	- `../` w nazwie pliku - to próba zapisu poza katalogiem upload, 
	- nietypowy `Content-Type` np. `application/x-php` przy pliku jpeg, 
	- ten sam IP przy `POST /upload` + `GET /uploads/shell.php` - wgranie i uruchomienie webshella, 
	- bardzo mały plik .php (kilkaset bajtów) - typowy webshell. 

## Jak się bronić:
- biała lista rozszerzeń - czyli pozwolenie tylko na `.jpg`, `.png`, `.odf` - nic więcej, 
- sprawdzanie zawartości (magic bytes) - sprawdzenie, czy plik naprawdę jest obrazem, a nie tylko ma rozszerzenie `.jpg`, 
- zmiana nazwy pliku po uploadzie - zapisanie pliku pod losową nazwą np. UUID, bez rozszerzenia wykonawczego, 
- zapis poza katalogiem www - pliki uploadowane warto trzymać poza katalogiem, z którego serwer wykonuje kod, 
- wyłączenie wykonywania w `/uploads` - w konfigiracji serwera należy wyłączyć `php_flag engine` (apache), lub dodać regułę w nginx, 
- WAF - blokuje upload plików wykonywalnych i podwójnych rozszerzeń, 
- minimalne uprawnienia - katalog uploads nie powinien mieć prawa wykonywania, 
- monitoring - alert na nowe pliki `.php` w `/uploads`. 