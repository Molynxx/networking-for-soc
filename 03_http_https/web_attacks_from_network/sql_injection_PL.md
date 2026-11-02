# sql_injection

## Cel
Pozananie i zrozumienie mechaniki działania ataku SQL Injection (SQLi), sposóbów jego wykrywania oraz zabezpieczania się przed nim. 

## Co to jest: 
SQL Injection to atak, w którym atakujący wstrzykuje kod SQL w miejsce, w którym aplikacja oczekuje zwykłych danych takich jak np. login, hasło, parametr URL.

## Cel ataku: 
- zalogowanie się na konto bez znajomości hasła, 
- kradzież danych z bazy (np. hasła, adresy mailowe, dane kart klientów), 
- zmiana lub modyfikacja danych na serwerze. 

## Działanie ataku:
- jeśli np. aplikacja ma formularz logowania, to w normalnym scenariuszu użytkownik w pola login i hasło wpisuje swoje dane. Serwer wtedy buduje zapytanie:   
	`SELECT * FROM users WHERE login = 'jan' AND password = 'jan1234':`   
Następnie baza szuka wiersza, w którym występuje login 'jan' i hasło 'jan1234'. jeśli takie dane zostaną znalezione użytkownik zostaje zalogowany. 
- atakujący może spróbować zamiast danych podać fragment kodu , który zostaje wstrzyknięty do zapytania bazy: `' OR 1=1--`, 
- ten kod zostanie wstawiony w zamiast loginu i hasła w zapytanie bazy:   
	`SELECT * FROM users WHERE login = '' OR 1=1-- AND password = 'cokolwiek':`  
	- co tutaj zaszło:
		- `login = ''` pusty, nic nie zostanie znalezione, 
		- `OR 1=1` - zawsze prawda, 
		- `--` - to znak komentarza, wszystko co występuje po nim jest ignorowane, 
		- co więc mówi polecenie: szukaj we wszystkich kolumnach w tabeli users, gdzie login jest pusty lub 1=1 (zdanie logiczne (tautologia) - zawsze prawda), ignoruj wszystko co jest dalej.
- aplikacja bierze pierwszy wynik ze zwróconej przez bazę listy, (często admina, z powodu alfabetycznego ustawienia wpisów) i loguje na nim atakującego. Atakujący dostaje sesję, nie dane do logowania. 

## Jak wykryć:
- w logach ukazuje to jako zwykłe żądanie HTTP lecz z dziwnymi parametrami:  
	`GET /login.php?user=' OR 1=1--pass=cokolwiek HTTP/1.1`
- czerwone flagi dla parametrów:
	- pojedynczy cudzysłów `'`, 
	- słowa SQL: `OR`. `UNION`, `SELECT`, `DROP`, `UPDATE`,
	- komentarze: `--`, `#`,
	- zdania logiczne: `1=1`, `'a'='a'`, 
	- serie podobnych żądań z jednego IP.  

## Jak się bronić: 
- Prepared statements (przygotowane zapytania): jest to sposób budowania zapytań SQL, w którym dane są oddzielone od kodu. Zamiast wstawiać wprowadzone dane do zapytania    
	- `SELECT * FROM users WHERE login = 'jan';`   
aplikacja najpierw tworzy szablon zapytania,:   		
	- `SELECT * FROM users WHERE login = ?;`   
W takiej sytuacji baza wie, że `?` to miejsce na dane, a nie na kod, więc nawet jeśli atakujący wpisze ` ' OR 1=1--` to baza potraktuje to jako zwykły tekst a nie kod SQL. Dzięki temu atakujący nie ma możliwości wstrzyknięcia kodu SQL. Większość języków programowania ma wbudowane prepared statements.
- Walidacja danych: Metoda ta polega na sprawdzaniu czy dane wpisane przez użytkownika mają sens i nie zawierają niedozwolonych znaków. Zanim parametr zostanie użyty aplikacja sprawdza parametr przed użyciem, czy:
	- Id jest liczbą?
	- Login zawiera tylko litery i cyfry, 
	- email wygląda jak emai.  
	Jeśli walidacja się nie powiedzie aplikacja odrzuca żądanie, więc atakujący nie może wpisać `' OR 1=1--` w pole, które oczekuje liczby lub liter. 
- WAF (Web Application Firewall): to jest aplikacyjna zapora - urządzenie lub oprogramowanie stojące przed aplikacją i analizujące ruch HTTP. WAF zawiera reguły i wzorce, które opisują znane ataki, np: 
	- jeśli w parametrze znajduje się `' OR 1=1--` -> zablokuj, 
	- jeśli w URL znajduje się `<script>` -> zablokuj, 
	- jeśli w żądaniu znajduje się `INION SELECT` -> zablokuj.  
	Każde żądanie jest analizowane przez WAF i porównywane ze wzorcami, a każde podejrzane żądanie jest blokowane, zanim dotrze do aplikacji. WAF jest cennym źródłem dla SOC, ponieważ  loguje zablokowane żądania. 
- zasada minimalnych uprawnień: konto bazy danych, którego używa aplikacja powinno mieć tylko te uprawnienia, która są mu niezbędne. Dlatego też tworzy się konto z ograniczeniami, które może używać `SELECT` (czytać) oraz `INSERT` (doładować), ale nie może używać `DROP` (usuwać tabel) i `UPDATE` (zmieniać danych bez potrzeby). Więc nawet jeżeli atakujący wstrzyknie `DROP TABLE users`, baza odmówi wykonania bo konto nie ma do tego uprawnień, jednak nadal może wyciągać dane przez `SELECT`. 
- logowanie i monitoring: zbieranie logów z aplikacji, serwera WWW i bazy danych oraz analiza ich pod kątem podejrzanych zdarzeń. 
