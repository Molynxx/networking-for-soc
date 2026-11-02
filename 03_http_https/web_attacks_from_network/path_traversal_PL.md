# path_traversal

## Cel 
Zapoznanie się i zrozumienie mechaniki działania ataku Path Traversal (Directory Traversal). 

## Co to jest
Jest to atak, w którym atakujący wychodzi poza dozwolony katalog i odczytuje pliki systemowe lub inne, do których nie powinien mieć dostępu. 

## Cel 
- odczytać pliki systemowe np. `/etc/passwd`, `/etc/shadow`, 
- odczytać pliki konfiguracyjne np. hasła i klucze API, 
- uzyskać dostęp do plików poza katalogiem publicznym.

## Działanie ataku
- w normalnym scenariuszu aplikacja ma katalog `/uploads` i pozwala pobierać pliki `GET /download?file=raport.pdf` - serwer czyta plik i wysyła go klientowi, 
- atakujący może edytować ścieżkę, tak by zamiast `raport.pdf` było: `GET /download?file=../../etc/passwd`   
	- co to oznacza: 
		- `..` - idź katalog wyżej, 
		- `../` - idź poziom wyżej, 
		- `../../` - idź dwa poziomy wyżej, 
		- `../../etc/passwd` - wyjdź dwa poziomy wyżej, wejdź do `/etc` i odczytaj plik `passwd`.  
		Serwer odczytuje plik i wysyła go atakującemu. 
- są sytuacje kiedy atakujący chcąc ominąć filtry używa różnych zapisów - mają to samo znaczenie lecz inny zapis:
	- `../` - klasyczny zapis, 
	- `..%2f` - zapis z zakodowanym slash, 
	- `%2e%2e%2f` - zapis z zakodowanymi dwiema kropkami i slash, 
	- `....//` - zapis z podwójnymi kropkami i slash - próba ominięcia filtra który usuwa `../`, 
	- `..\` - stosowany w Windows gdzie ścieżka jest podzielona backslash a nie slash. 

## Jak wykryć
- monitorowanie parametrów występujących w żądaniach, jeśli występuje w nich:
	- `../`, 
	- `..%2f`, `%2e%2e%2f`, 
	- `/etc/passwd`, `/etc/shadow`, `C:\Windows\`, 
	- `../../`,    
	mamy do czynienia z Path Traversal.
- przykłady:
	- `GET /download?file=../../etc/passwd HTTP/1.1`, 
	- `GET /download?file=..%2f..%2fetc%2fpasswd HTTP/1.1`

## Jak się bronić
- walidacja ścieżki - sprawdzanie czy ścieżka nie zawiera niedozwolonych znaków, znajdujących się na blacklist, lub sprawdza czy użyte znaki są na whitelist. Jeśli ścieżka zawiera niedozwolone znaki, serwer blokuje żądanie, 
- whitelist plików - pozwalanie na dostęp tylko do określonych na liście plików, 
- Chroot/sandbox - zamknięcie aplikacji w katalogu, z którego nie da się wyjść, 
- brak bezpośredniego dostępu do plików - czyli serwerowanie plików przez ID, a nie przez ścieżkę,  
- WAF (Web Application Firewall) - analizuje wszystkie żądania i blokuje wszystkie podejrzane żądania, w tym Path Traversal. Uwaga: równolegle z WAF należy ustawić czarną listę znaków, żeby nie można było obejść filtrowania WAF, 
- monitoring - ustawione alerty na `../` w parametrach. 