# https_and_tls

## Cel
Zrozumienie na czym polega różnica pomiędzy protokołami HTTP a HTTPS, czym jest TLS, jak przebiega handshake i co z tego widzi SOC. 

## Czym jest HTTPS i czym różni się od HTTP
HTTP to protokół, który przesyła dane jawnie, dlatego każdy kto jest po drodze (np. router, MitM, itp) może te dane zobaczyć. HTTPS powstało by ten problem rozwiązać, to HTTP opakowane w TLS (szyfrowanie). Główne różnice:
- HTTP: 
	- port: 80,
	- szyfrowanie: brak,
	- widoczność treści: pełna, 
	- uwierzytelnienie serwera: brak. 
- HTTPS:
	- port: 443,
	- szyfrowanie: TLS,
	- widoczność treści: brak, 
	- uwierzytelnienie serwera: certyfikat.   
HTTPS zapewnia więc nam poufność, integralność (nikt nie zmieni danych po drodze) oraz uwierzytelnienie (daje pewność, że łączysz się z prawdziwym serwerem). HTTPS nie tyle jest osobnym protokołem, co połączeniem HTTP z szyfrowaniem w warstwie prezentacji. Można to zobrazować na podstawie warstw OSI:
- HTTP: 
	- warstwa aplikacji: HTTP, 
	- warstwa transportowa: TCP, 
	- warstwa sieciowa: IP. 
- HTTPS:
	- warstwa aplikacji: HTTP, 
	- `warstwa prezentacji`: TLS (szyfrowanie),
	- warstwa transportowa: TCP, 
	- warstwa sieciowa: IP. 
Szyfrowanie pozwala ukryć wszystkie informacje przed osobami trzecimi, treść danych jest w tym przypadku niejawna. SOC może zobaczyć jedynie:
- połączenie TCP na porcie 443,
- handshake TLS (wersja, SNI, certyfikat), 
- rozmiar i częstotliwość pakietów.  
To czego nie widzi to treść HTTP. 

## TLS handshake
Podobnie jak TCP, TLS ma swój handshake i zanim zostaną przesłane dane serwer i klient muszą uzgodnić, jak będą szyfrować. 
- Krok 1 - ClientHello:
	- klient wysyła wersje TLS (np. 1.2, 1.3), 
	- listę cipher suites (czyli algorytmy szyfrowania), 
	- SNI - czyli nazwę domeny, do której chce się połączyć, 
	- client random (losowe dane do klucza).   
	W tym pakiecie widać SNI, to jest jedno z niewielu miejsc gdzie w HTTPS można zobaczyć nazwę domeny. 
- Krok 2 - ServerHello - serwer odpowiada:
	- wybraną wersję TLS, 
	- wybrany cipher suite, 
	- swój certyfikat, 
	- server random.  
	Jeśli serwer wybierze wersję TLS starszą mimo, że klient wspierał nowszą - powinno to budzić podejrzenia. 
- Krok 3 - Weryfikacja certyfikatu - klient sprawdza:
	- datę ważności certyfikatu, czy nie jest wygasły, 
	- czy jest podpisany przez zaufane CA (Certificate Authority - czyli Urząd Certyfikacji), 
	- czy nazwa domeny się zgadza.   
	Jeśli cokolwiek się nie zgadza - klient przerwie połączenie. 
- Krok 4 - Wymiana kluczy:
	- klient i serwer ustalają wspólny klucz sesji (symetryczny), który będzie używany do szyfrowania całej komunikacji. 
- Krok 5 - zakończenie:
	- obie strony potwierdzają, że handshake zakończył się pomyślnie i od tego momentu zaczyna się szyfrowana transmisja danych. 

## Rodzaje szyfrowania TLS 
W TLS występują dwa rodzaje szyfrowania, które spełniają inne zadania:
- Asymetryczne:
	- dwa klucze (publiczny i prywatny)
	- jest dość wolne, 
	- służy do wymiany klucza sesji.
- Symetryczne:
	- jeden klucz, 
	- jest szybkie - przy większych ilościach danych bardzo przyspiesza komunikację, 
	- służy do szyfrowania danych.   
Więc najpierw za pomocą szyfrowania asymetrycznego serwer i klient ustalają klucze, a następnie następuje wymiana danych już za pomocą szyfrowania symetrycznego. 

## Certyfikaty i łańcuch zaufania
- Certyfikat to dokument cyfrowy, który potwierdza, że dana domena należy do danej firmy. Certyfikat zawiera: 
	- nazwę domeny, 
	- klucz publiczny, 
	- właściciela domeny, 
	- datę ważności, 
	- podpis CA.
- Łańcuch zaufania:    
	`Root CA -> Intermediate CA -> Certyfikat serwera`    
	- Root CA - główny urząd certyfikacji, który jest wbudowany w przeglądarki/systemy. Co to oznacza: Przeglądarki i systemy mają wbudowaną listę Root CA, więc nie musi sprawdzać czy dany urząd jest zaufany, tą listę już ma. 
	-  Intermediate CA - pośrednik, upoważniony do wystawienia certyfikatu przez Root CA. Lista Intermediate CA nie jest wbudowana w systemy czy przeglądarki, więc klient nie wie, który Intermediate jest zaufany. 
	- Certyfikat serwera - podpisany przez Intermediate CA. Ponieważ system/przeglądarka nie wie, który Intermediate jest zaufany, serwer wysyła klientowi swój certyfikat oraz certyfikat Intermediate CA podpisany przez Root CA. Klient ma wbudowaną listę Root CA więc sprawdza czy certyfikat Intermediate jest podpisany przez zaufany Root CA. Jeśli tak - ufa Intermediate CA i sprawdza czy certyfikat serwera jest podpisany przez ten Intermediate CA i jeśli wszystko się zgadza ufa serwerowi. 

## SNI - Server Name Indication 
SNI to nazwa domeny, którą klient podaje w ClientHello. Jest to istotne ponieważ jeden serwer może obsługiwać wiele domen i to właśnie SNI mówi serwerowi, której domeny klient chce zobaczyć certyfikaty. Jest widoczne w handshake nawet jeśli dane są szyfrowane. Dzięki temu można sprawdzić z jaką domeną łączy się klient, 

## Co SOC może zobaczyć w HTTPS 
- Co jest widoczne: 
	- SNI (nazwa domeny), 
	- IP serwera,
	- port 443,
	- rozmiar pakietów, 
	- certyfikat, 
	- wersja TLS 
- co jest niewidoczne: 
	- treść żądania HTTP, 
	- metody HTTP, 
	- kody odpowiedzi, 
	- nagłówki HTTP, 
	- cookies, 
	- dane logowania. 

## Zagrożenia związane z TLS
Chociaż TLS sam w sobie jest bezpieczny, to jeśli nie jest poprawnie skonfigurowany, atakujący mogą wykorzystać:
- fałszywe certyfikaty - czyli podszywanie się pod serwer, 
- downgrade - czyli zmuszenie do użycia starszej, słabszej wersji TLS, 
- self-signed certy - certyfikat wystawiony przez serwer samemu sobie - często w malware, 
- SNI spoofing - czyli podszywanie się pod domenę.  
Szczegóły tych ataków i sposób ich wykrywania są opisane w folderze 04_tls. 
