# brute_force

## Cel 
Zrozumienie mechaniki działania siłowego ataku brute force, sposobów jego wykrycia oraz jak się przed nim bronić. 

## Co to jest 
Brute force to atak polegający na zgadywaniu hasła, wypróbowując wiele kombinacji, aż któraś będzie trafna. Atak może mieć różne warianty:
- Brute force - próba wszystkich kombinacji, 
- Dictionary attack - próby słów ze słownika (np. hasło, qwerty, admin),
- Credential stuffing - próba wykorzystania loginów i haseł wykradzionych z innych stron, 
- Password spraying - próba jednego typowego hasła na wielu kontach. Typowego, bo w password spraying atakujący używa popularnego hasła, np. "Lato2026!", a nie losowego, żeby  nie zablokować kont. 

## Cel ataku
- dostanie się na konto, do bazy danych, SSH, poczty.

## Działanie ataku 
- atakujący może znać lub zgadnąć login, np. admin, 
- wysyła setki lub tysiące prób logowania z różnymi hasłami,
- aż w końcu trafi na właściwe.  
Wszystko to przez formularz logowania (najczęściej POST /login). 

## Jak wykryć
- jeśli analiza logów pokaże wiele `401 Unauthorized` z jednego IP w krótkim czasie na to samo konto, wskazuje to atak brute force. Jeśli na końcu zdarzeń `401` pokaże się wpis `200 OK`, to atak zakończył się powodzeniem. Inne czerwone flagi w tym ataku to także nietypowe godziny oraz narzędzia typu `hydra`(program do brute force), `curl` (program wiersza poleceń do wysyłania żądań HTTP), `python-requests` (biblioteka Pythona do wysyłania żądań HTTP) - w logach widać je jako `User-Agent`, np. `User-Agent: curl/7.88.1`.  

## Jak się bronić 
- Rate limiting - ograniczenie liczby prób logowania z jednego IP, 
- Blokada konta - ustawienie ilości nieudanych prób, po której przekroczeniu konto zostaje zablokowane, 
- CAPTCHA - pozwala odróżnić prawdziwego użytkownika od bota, 
- MFA - wieloskładnikowe uwierzytelnianie. Nawet jeśli atakujący odgadnie hasło, wymagany będzie jeszcze drugi czynnik, by logowanie się powiodło, 
- Silne hasło - dłuższe i bardziej złożone hasła są trudniejsze do złamania, 
- Monitoring - powtarzające się alerty 401 z jednego IP w krótkim czasie.