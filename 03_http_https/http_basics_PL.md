# http_basics

## Cel
Zrozumienie czym jest protokół http, jakich używa metod oraz na jakie zagrożenia jest narażony.

## Czym jest http
HTTP (Hypertext Transfer Protocol) jest protokołem warstwy aplikacji, służący do komunikacji pomiędzy klientem (aplikacja, przeglądarka) a serwerem www. To jeden z najważniejszych protokołów dla SOC, ponieważ:
- niemal cały ruch webowy przechodzi przez protokół HTTP, 
- większość aplikacyjnych ataków takich jak SQLi, SXX, phishing czy brute force, można wykryć w logach HTTP, 
- zrozumienie kodów, metod oraz nagłówków to podstawa triange'u.   

## Jak działa HTTP
Działanie tego protokołu opera się na żądaniach i odpowiedziach -> klient wysyła żądanie, serwer odpowiada na nie. HTTP to nie tunel jak SSH, to seria niezależnych pytań i odpowiedzi. Po tej wymianie zależnie od wersji HTTP następują działania:
- HTTP 1.0 (close) - po każdym pytaniu i odpowiedzi połączenie zostaje zamykane, więc przy kolejnym musi nastąpić ponowne three way handshake,  to mało wydajne dlatego ta wersja HTTP jest w zasadzie martwa, nieużywana, 
- HTTP 1.1 (keep-alive) - problem braku wydajności został rozwiązany poprzez podtrzymywanie połączenia. Po każdym pytaniu i po każdej odpowiedzi serwer czeka jeszcze chwilę, czy nie nadejdzie kolejne. Dopiero gdy przez jakiś czas jest cisza, połączenie jest zamykane. Jednak żądania czekają w kolejce w jednym połączeniu, 
- HTTP 2.0 