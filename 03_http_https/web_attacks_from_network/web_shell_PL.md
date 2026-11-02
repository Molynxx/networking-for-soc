# web_shell

## Cel
Poznanie i zrozumienie mechaniki działania ataku Web Shell, sposobów wykrycia go oraz zabezpieczania się przed nim. 

## Co to jest
To atak polegający na wgraniu złośliwego skryptu na serwer przez atakującego. Skrypt jest uruchamiany przez przeglądarkę i działa jak zdalny terminal - atakujący wpisuje polecenia w przeglądarce, a serwer je wykonuje. Często to kontynuacja File Upload - atakujący najpierw wgrywa plik, a potem go używa. 

## Cel
- zdalne wykonywanie poleceń (RCE) na serwerze, 
- utrzymanie dostępu (persistence) - webshell zostaje, nawet jeśli inne ślady zostaną usunięte, 
- kradzież danych, 
- dalsze ataki - pivot na inne systemy w sieci. 

## Działanie ataku
- atakujący za pomocą ataku File Upload lub wykorzystując inną lukę wgrywa shell na serwer (np. `shell.php` do `/uploads`), 
- powyższy plik zawiera kod, który przyjmuje polecenia z parametru i wykonuje je w systemie, np. `<?php system($_GET['cmd']); ?>`, 
- następnie atakujący wchodzi w przeglądarce na `https://example.com/uploads/shell.php?cmd=whoami`, 
- serwer wykonuje polecenie z parametru cmd i zwraca wynik whoami, 
- jeśli atak zakończy się powodzeniem, atakujący ma zdalny dostęp do serwera i może zrobić wszystko na co pozwalają uprawnienia aplikacji. 
- warianty webshell:
	- prosty - jedno polecenie w parametrze (`?cmd=...`) - łatwy do wykrycia, 
	- złożony - panel z formularzem, upload plików, edycja plików - trudny do wykrycia, 
	- obfuskacja - kod zakodowany (np. base64) - trudny do wykrycia w plikach. 

## Jak wykryć
w logach serwera, systemowych i aplikacji:
- żądania do plików wykonywalnych w katalogach upload - zwykle w takich katalogach są obrazki, pdfy, załączniki, jeśli jednak w tych katalogach pojawia się żądanie do pliku `.php`, `.jsp`, `.asp`, to oznacza, że ktoś wgrał tam kod wykonywalny - to klasyczny sygnał webshella, 
- parametry z nazwami poleceń w URL - parametry takie jak `cmd`, `exec`, `command`, `shell`, `run` wskazują na webshell, 
- powtarzające się żądania do tego samego pliku z różnymi parametrami - normalny ruch nie wysyła żądania do tego samego pliku wiele razy, jeśli to się pojawia to sygnał, że należy sprawdzić serwer pod kątem webshell, 
- odpowiedzi z dużą ilością tekstu - wynik polecenia systemowego (np. `ls`, `cat`, `whoami`) wraca jako tekst na stronie. Jeśli rozmiar odpowiedzi na żądanie do pliku w `/uploads` jest nietypowo duży (setki bajtów) to może być wynik polecenia, warto sprawdzić rozmiar odpowiedzi w `access.log`, 
- korelacja z File Upload - webshell prawie zawsze idzie w parze z wgraniem pliku. Więc jeśli w logach widać najpierw `POST /uploads`, a chwilę potem `GET /uploads/shell.php?cmd=...` z tego samego IP, to jest jeden łańcuch ataku, 
- nowe pliki w katalogu uploads - AIDE (file integrity monitoring), dla każdego nowego pliku `.php` w `/uploads` przychodzi powiadomienie, to najszybszy sposób wykrycia, bez czekania, aż atakujący użyje webshella, 
- nietypowe nazwy plików - atakujący często zmieniają nazwy plików, żeby obejść filtry, np. `shell.php%00.jpeg`, dlatego w logach i systemie plików warto zwracać uwagę na pliki, których nazwa nie pasuje do zawartości.

## Jak się bronić
- wyłączenie wykonywania kodu w katalogu uploads - to najskuteczniejsza obrona (apache -> `php_flag engine off` w `.htaccess` w katalogu `/uploads`, w nginx dodaje się regułę, blokującą wykonanie `.php` w tym katalogu. W wyniku takiego działania atakujący, nawet jeśli umieści webshell, to serwer go nie wykona, 
- biała lista rozszerzeń - ustawienie, by aplikacja przyjmowała tylko konkretne rozszerzenia, np. `.jpeg`, `.png`, `.pdf`, 
- sprawdzanie zawartości plików (magic bytes) - zamiast ufać rozszerzeniu, można sprawdzić, czy plik naprawdę jest obrazem. Obraz ma określone "magic bytes" na początku pliku (np. FF D8 FF dla JPEG). Jeśli plik ma rozszerzenie `.jpg` ale nie jest obrazem, powinien zostać odrzucony, 
- zmiana nazwy pliku po uploadzie - losowa nazwa bez rozszerzenia wykonawczego uniemożliwi atakującemu wywołanie pliku, bo nie będzie znał jego nazwy,
- zapis poza katalogiem www - wgrywane pliki powinny być przechowywane poza katalogiem, z którego serwer wykonuje kod (np. w `/var/uploads`, a nie w `/var/www/html/uploads/`). Dzięki temu nawet po wgraniu webshella atakujący nie będzie mógł go uruchomić przez przeglądarkę, 
- minimalne uprawnienia - aplikacja nie powinna działać jako root, 
- File Integrity Monitoring - np. AIDE, monitorują zmiany w plikach, powiadamiają o każdym nowym pliku `.php` w `/uploads`, 
- WAF - może blokować żądania z parametrami `cmd`, `exec`, `command` do plików w katalogach `/uploads`, 
- W SIEM warto ustawić monitoring i alerty na:
	- nowe pliki `.php` w `/uploads`, 
	- żądania do `.php` w `/uploads` z parametrem `cmd`, 
	- korelację: `POST /upload` + `GET /uploads/*.php` z tego samego IP. 

