# brute_force

## Cel 
Zrozumienie mechaniki działania siłowego ataku brute force, sposobów jego wykrycia oraz jak się przed nim bronić. 

## Co to jest 
- co to jest: brute force to atak polegający na zgadywaniu hasła wypróbowując wiele kombinacji, aż któraś będzie trafiona. Atak może mieć różne warianty:
	- Brute force - próba wszystkich kombinacji, 
	- Dictionary attack - próby słów ze słownika (np. hasło, qwerty, admin),
	- Credential stuffing - próba wykorzystania loginów i haseł wykradzionych z innych stron, 
	- Password spraying - próba jednego hasła na wielu kontach. 

## cel ataku:
-  dostanie się na konto, do bazy danych, SSH, poczty.

## jak działa: 
- atakujący może znać lub zgadnąć login np. admin, 
- wysyła setki lub tysiące prób logowania z różnymi hasłami,
- aż w końcu trafi na właściwe.  
	Wszystko to przez formularz logowania (najczęściej POST /login). 

## jak wykryć: 
- jeśli analiza logów pokaże wiele `401 Unauthorized` z jednego IP w krótkim czasie na to samo konto, wskazuje to atak brute force. Jeśli na końcu zdarzeń `401` pokaże się wpis `200 OK` to atak zakończył się powodzeniem. Inne czerwone flagi w tym ataku to także nietypowe godziny oraz narzędzia typu `hydra`, `curl`, `python-requests`. 

## jak się bronić: 
- Rate limiting - ograniczenie liczby prób logowania z jednego IP, 
- Blokada konta - ustawienie ilości nieudanych prób, po której przekroczeniu konto zostaje zablokowane, 
- CAPTHCHA - pozwala odróżnić prawdziwego użytkownika od bota, 
- MFA - wieloskładnikowe uwierzytelnianie. Nawet jeśli atakujący odgadnie hasło, wymagany będzie jeszcze drugi czynnik by logowanie się powiodło, 
- Silne hasło - dłuższe i bardziej złożone hasła są trudniejsze to złamania, 
- Monitoring - powtarzające się alerty 401 z jednego IP w krótkim czasie.