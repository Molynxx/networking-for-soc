# session_hijacking

## Cel
Poznanie i zrozumienie mechaniki działania ataku Session Hijacking (przejęcie sesji) oraz sposobów jego wykrycia i zabezpieczania się przed nim.

## Co to jest
Atak ten polega na kradzieży przez atakującego ciasteczka sesji ofiary i użyciu go by podszyć się pod nią. 

## Cel ataku
- uzyskanie dostępu do konta bez znajomości hasła, 
- wykonanie w imieniu ofiary akcji np. zlecenie przelewu, zmiana hasła, kradzież danych.

## Działanie ataku 
- gdy ofiara loguje się na swoje konto dostaje ciasteczko sesji np. session=abc123, 
- atakujący kradnie to ciasteczko za pomocą np.:
	- XSS - wstrzyknięcie JavaScript, który odczytuje `document.cookie` i wysyła do atakującego, 
	- MitM - jeśli sieć nie jest zabezpieczona atakujący może podsłuchiwać ruch i przechwycić ciasteczko,
	- Malware - na urządzeniu ofiary jest zainstalowany złośliwy program, który czyta ciasteczka i wysyła je do atakującego, 
	- Przewidywanie sesji - jeśli ID sesji jest słabe (np. 1234) atakujący może je zgadnąć, 
	- Physical access - jeśli atakujący ma dostęp do komputera ofiary, może odczytać ciasteczka, 
- gdy atakujący posiada już ciasteczko wysyła do serwera żądanie z tym samym ciasteczkiem, 
- po tych działaniach atakujący ma dostęp do konta ofiary, bez znajomości hasła. 

## Jak wykryć
- jeśli ta sama sesja z dwóch różnych IP (to samo ciasteczko) - powinno budzić podejrzenia, bo jedna osoba nie może być w dwóch miejscach na raz, 
- ta sama sesja z różnych `User-Agentów` (różne przeglądarki) - przeglądarki nie dziedziczą sesji, więc to mało prawdopodobne, że użytkownik uruchomił tę samą sesję w dwóch różnych przeglądarkach, 
- nastąpiła nagła zmiana lokalizacji sesji - np. IP początkowo z Polski a potem z USA może świadczyć o przejęciu sesji, choć to może być nagle włączony VPN, 
- jeśli akcje sesji są wykonywane poza godzinami pracy ofiary, może to wskazywać na to, że ktoś inny używa jej sesji. 

## Jak się bronić
- HTTPS - protokół ten szyfruje ciasteczka w tranzycie, więc MitM ich nie widzi, 
- HttpOnly - ciasteczko jest niedostępne dla JavaScript, więc atak XSS nie może go odczytać. To flaga, którą dodaje się do ciasteczka - ustawia się po stronie serwera w nagłówku `Set-Cookie` podczas wysyłania ciasteczka, 
- Secure - ciasteczko jest wysyłane wyłącznie przez szyfrowany protokół HTTPS. To kolejna flaga, którą ustawia się w tym samym miejscu co HttpOnly,
- SameSite - ustawienie za pomocą flagi (ustawiana jak dwie poprzednie), która mówi przeglądarce: "Wysyłaj to ciasteczko tylko wtedy, gdy żądanie przychodzi z tej samej strony", 
- Regeneracja ID sesji po logowaniu - po udanym logowaniu serwer unieważnia stare ID i tworzy nowe. To funkcja w kodzie aplikacji, 
- weryfikacja IP / User-Agent - serwer sprawdza czy każde żądanie w ramach sesji pochodzi z tego samego IP i tego samego User-Agent, co przy logowaniu. To rozwiązanie ma jednak wady, w przypadku mobilnych użytkowników, którzy często zmieniają IP (przełączanie Wi-Fi/LTE lub VPN czy proxy, które zmieniają IP) zabezpieczenie to utrudnia funkcjonowanie sesji. Podobnie jak wyżej, to jest funkcja w kodzie aplikacji. 