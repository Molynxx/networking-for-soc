# xss

## Cel
Poznanie i zrozumienie mechaniki działania ataku XSS (Cross-Site Scripting), sposobów jego wykrycia oraz zabezpieczania się przed nim. 

## Co to jest
XSS jest to atak polegający na wstrzyknięciu JavaScript do strony, którą widzi ofiara. Przeglądarka ofiary wykonuje ten kod, myśląc, że to kod do strony. By zrozumieć dobrze działanie tego ataku należy wyjaśnić czym jest i jak działa JacaScript. JavaScript to język programowania, który jest instrukcją dla przeglądarki jak ma wyświetlać stronę. JS ma dostęp do:
- ciasteczek (sesji),
- treści strony (DOM), 
- adresu URL, 
- danych, które użytkownik widzi.   
Kiedy użytkownik otwiera stronę, przeglądarka pobiera HTML i wykonuje zawarty w nim JavaScript. To klucz do ataku typu XSS - jeśli atakującemu uda się wstrzyknąć swój kod, to wykona się on w przeglądarce ofiary. 

## Cel
- kradzież ciasteczek sesji (przejęcie sesji ofiary),
- przekierowanie ofiary na fałszywą stronę (phishing),
- wykonanie akcji w imieniu ofiary (przelew, zmiana hasła),
- wyświetlenie fałszywego formularza logowania i kradzież hasła, 
- keylogging - przechwytywanie, co ofiara wpisuje. 

## Działanie ataku (w zależności od typu):
- Stored XSS (zapisany):
	- strona ma miejsce gdzie użytkownicy mogą coś napisać, np. sekcja komentarzy pod artykułem, 
	- serwer zapisuje każdy komentarz w bazie,
	- serwer wyświetla komentarz innym użytkownikom, 
	- atakujący wpisuje w komentarz kod JavaScript zamiast zwykłego tekstu,
	- serwer zapisuje ten komentarz w bazie i pokazuje wszystkim odwiedzającym stronę, 
	- przeglądarka każdej ofiary, która otworzy tę stronę z artykułem wykona kod zapisany w komentarzu. Należy pamiętać, że ten kod w komentarzu jest dla przeglądarki częścią instrukcji wyświetlania strony, przeglądarka nie wie, że to tylko komentarz, rozpoznaje składnię i wykonuje kod w niej zawarty. Ten rodzaj XSS jest najbardziej niebezpieczny, ponieważ dotyczy każdej ofiary otwierającej stronę. 
- Reflected XSS (odbity):
	- atakujący wysyła link do ofiary z kodem w parametrze, 
	- przeglądarka wysyła zapytanie do serwera, 
	- serwer odbija ten kod w odpowiedzi czyli wstawia go do HTML, który wysyła z powrotem, 
	- przeglądarka otrzymuje w odpowiedzi HTML z kodem i wykonuje go, 
	- Ważne: serwer sam nie wykonuje kodu tylko wstawia go do HTML i odsyła, a ponieważ kod nie jest zapisany w bazie działa tylko dla tego, kto kliknie w link. 
- DOM-based XSS:
	- tutaj istotne jest wyjaśnienie co to jest DOM: jest to struktura strony, którą przeglądarka trzyma w pamięci. To drzewo elementów `<html>`, `<body>`, `<div>`, `<p>`, itp. Kiedy przeglądarka dostaje HTML od serwera, buduje z niego DOM, czyli wewnętrzną reprezentację strony. JavaScript może czytać i zmieniać DOM (np. zmienić tekst w `<p>`, dodać nowy `<div>` lub usunąć jakiś element). 
	- strona ma JavaScript napisany przez developera, który czyta dane z URL i wstawia je do strony. Np. `document.getElementById('output').innerHTML = location.hash;`. 
		- wyjaśnienie kodu: weź wszystko po `#` z URL i wstaw do elementu  `<p id="output">`,
	- atakujący wysyła link z kodem po `#`, np: `https://example.com/#<script>...</script>`, 
	- ofiara klika w link, 
	- przeglądarka otwiera stronę, a serwer nie widzi nic co znajduje się po znaku `#`, 
	- serwer wysyła normalną stronę, 
	- JavaScript na stronie czyta kod z URL i wstawia go do DOM, 
	- przeglądarka wykonuje ten kod.
	- Kluczowe: kod nie przechodzi przez serwer, to przeglądarka wstawia go sama, bo taki ma JavaScript na stronie. Atak ten jest niebezpieczny ponieważ serwer nie widzi ataku, a w logach jest tylko `GET /`, więc nie ma nic podejrzanego. 

## Jak wykryć
- Stored XSS: w logach serwera jest trudny do wykrycia, ponieważ zwykle nie zaleca się logowania treści komentarzy, jednak można wykryć ten atak za pomocą odpowiednich narzędzi takich jak WAF, Raportowanie CSP, Skanowanie bazy danych, SIEM i korelacja. 
	- WAF (Web Application Firewall): analizuje ruch HTTP i loguje wzorce, które wywołują reguły XSS. W logach WAF można znaleźć pole o nazwie `terminatingRuleMatchDetails`, które zawiera dokładny payload (złośliwy kod), np. `<script>alert(1)</script>` oraz identyfikator żądania, 
	- CSP (Content Security Policy) - to nagłówek, który mówi przeglądarce, jakie skrypty może wykonywać. Jeśli przeglądarka ofiary napotka zablokowany skrypt (np. ten wstrzyknięty przez atakującego), wysyła raport o naruszeniu na serwer. Ten raport zawiera informacje o zablokowanym skrypcie i stronie, na której wystąpił problem. Jednak to jest informacja już po fakcie, gdy atak już zadziałał, 
	- Skanowanie bazy danych (prewencja) - okresowe skanowanie pól w bazie danych (np. komentarze, opisy profili) w poszukiwaniu znaków `<script>`, `onerror`, `javascript`, itp pozwala wykryć złośliwe payloady, zanim ofiara je zobaczy. 
	- SIEM i korelacja - można ustawić reguły, które korelują alerty z WAF i CSP. Np. jeśli WAF zgłosi próbę XSS, a następnie CSP zgłosi naruszenie z tego samego IP to atak jest potwierdzony. 
- Reflected XSS: jest to łatwiejsze do wykrycia ponieważ jest widoczne w logach serwera, np:   
	- `192.168.1.50 - - [18/Oct/2026:10:05:00] "GET /search?q=<script>alert(1)</script> HTTP/1.1" 200`  
	URL jest logowany w całości, dlatego Reflected XSS jest łatwiejszy do wykrycia w logach - kod jest w adresie, a adres trafia do logów. 
- DOM-based XSS: 
	- raportowanie CSP - jeśli przeglądarka ma nagłówek CSP z dyrektywą report-uri lub report-to, przeglądarka ofiary automatycznie wyśle raport za każdym razem gdy zablokuje wykonanie skryptu niezgodnego z polityką. raport zawiera m.in adres URL strony, na której doszło do naruszenia oraz szczegóły zablokowanego skryptu,
	- WAF może wyłapać eksfiltrację - ale to skutek nie samo wstrzyknięcie.
- czerwone flagi:
	- `<script>` w parametrach - próba wstrzyknięcia kodu JS, 
	- `javascript:` w URL - próba wykonania kodu po kliknięciu, 
	- `onerror=`, `onload=`, `onclick=` - atrybuty HTML z kodem JS, 
	- `<img src=x onerror...>` - klasyczny wektor XSS bez `<script>`, 
	- `document.cookie` w parametrach - próba kradzieży sesji, 
	- `%3Cscript%3E` - zakodowany `<script>`, próba obejścia filtrów. 

## Jak się bronić
- escapowanie danych: zamienia znaki specjalne na encje HTML np: `<` -> `&lt;`, `>` -> `&gt;`. Dzięki temu przeglądarka widzi tekst a nie kod. 
- Walidacja danych: sprawdzanie, czy dane mają sens, czyli czy id jest liczbą, czy email wygląda jak email. Dzięki temu atakujący nie może wpisać `<script>` w pole, które oczekuje tylko liter, 
- Content Security Policy (CSP): to nagłówek HTTP mówiący przeglądarce, jakie skrypty może wykonać. Dzięki temu nieautoryzowane skrypty są blokowane a raporty o próbach naruszenia są wysyłane do serwera. Jednak CSP można obejść, jeśli używa białej listy domen (atakujący może wykorzystać zaufany skrypt np: JSONP). Dlatego lepsze jest `nonce` lub `hash` zamiast listy domen, 
- HttpOnly na ciasteczkach: to flaga ciasteczka, która mówi przeglądarce: "JavaScript nie może odczytać tego ciasteczka". Dzięki temu nawet jeśli XSS się wykona, atakujący nie ukradnie sesji. To najważniejsza obrona przed skutkiem XSS. 
- WAF: blokuje podejrzane żądania na podstawie wzorców, dzięki czemu payload nie dociera do aplikacji. Jednak nie wykryje DOM-based XSS ponieważ payload występuje po `#`, 
- prewencja (po stronie developera): analiza kodu przed wdrożeniem, żeby znaleźć niebezpieczne wzorce takie jak:
	- niebezpieczne źródła w kodzie (skąd biorą się dane):
		- `location.hash` - to, co po `#`, 
		- `location.search` - to, co po `?`, 
		- `decument.URL`,
		- `document.referrer`, 
		- `window.name`,
	- niebezpieczne ujścia (gdzie wstawiane są dane):
		- `innerHTML`, 
		- `outerHTML`, 
		- `document.write()`,
		- `eval()`,
		- `setTimeout()` z tekstem,
		- `setInterval()` z tekstem. 
	- podatność powstaje, gdy jedno łączy się z drugim, np: `document.getElementById('output').innerHTML = location.hash` - to jest klasyczny DOM-based XSS. 
	- bezpieczne alternatywy:
		- zamiast `innerHTML` można użyć `textContent`, 
		- zamiast `document.write()` można użyć `createElement()` + `appendChild()`,
		- zamiast `eval()` można użyć funkcji bez `eval`, 
		- zamiast wstawiania danych z URL bez sprawdzania można zastosować walidację i escapowanie. 