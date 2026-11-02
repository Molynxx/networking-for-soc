# csrf

## Cel
Poznanie mechaniki działania ataku Cross-Site Request Forgery (CSRF) sposobów jego wykrywania i zabezpieczenia się przed nim. 

## Co to jest:
- to atak, w którym atakujący zmusza zalogowaną ofiarę do wykonania niechcianej akcji na stronie, na której jest zalogowana. Atakujący nie kradnie ciasteczka, lecz wykorzystuje to, że przeglądarka automatycznie dołącza ciasteczko do każdego żądania skierowanego do danej strony. 

## Cel ataku (przykład dla poczty email)
- zmiana hasła ofiary,
- ustawienie przekierowania (forwarding) wszystkich maili na adres atakującego, 
- dodanie filtra, który kasuje lub ukrywa maile (np. od banku), 
- wysłanie maila z konta ofiary (np. phishing do jej kontaktów),
- dodanie reguły auto-odpowiedzi, 
- zmiana numeru telefonu do odzyskania hasła, 
- atakujący nie czyta skrzynki na żywo, ale ustawia coś, co działa długo i po cichu. 

## Jak działa
- ofiara jest zalogowana na poczcie, do czasu wylogowania w przeglądarce jest aktywne ciasteczko sesji. UWAGA: nawet jeśli karta poczty nie jest otwarta a użytkownik nie wylogował się z poczty lub ma ustawione "zostań zalogowany" ciasteczko nadal działa, 
- ofiara klika w link atakującego lub wchodzi na stronę, na której został ukryty kod atakującego, 
- na stronie istnieje ukryty formularz, np.
	``` 
	<form action="https://mail.example.com/settings/forwarding" method="POST">
		<input type="hidden" name="forward_to" value="atakujacy@evil.com">
		<input type="hidden" name="enabled" value="true">
	</form>
	<script>document.forms[0].submit();</script>
	```
Powyższy kod tworzy formularz, który ustawia przekierowanie wszystkich maili na `atakujacy@evil.com` i automatycznie wysyła go przez JavaScript. Ofiara tego nie widzi, strona może wyglądać jak artykuł, oferta, reklama, cokolwiek. Choć atakujący nie zna adresu email ofiary, to aktywne ciasteczko dokładnie wie o jaką sesję chodzi i jaki jest mail, 
- przeglądarka ofiary wysyła żądanie do poczty, ponieważ formularz wskazuje na `mail.example.com ` - więc przeglądarka automatycznie dołącza ciasteczko sesji poczty:
	```
	POST /settings/forwarding HTTP/1.1
	hist: mail.example.com
	Cookie: session=abc123

	forward_to=atakujacy@evil.com&enabled=true
	```
- serwer poczty widzi prawidłową sesję - więc zapisuje nowe ustawienie, 
- od tej chwili atakujący otrzymuje kopię każdego maila, który otrzymuje ofiara, lecz ofiara tego nie widzi. Atakujący w takiej sytuacji może czytać wszystko włącznie z mailami z banku, pracy, maile z resetami haseł, prywatną korespondencję.

## Jak wykryć
- referer z obcej domeny przy wrażliwej akcji - np. podczas zmiany ustawień na poczcie referer wskazuje na stronę poczty - to jest sytuacja normalna. Jednak jeśli zmieniane są ustawienia na poczcie a referer wskazuje na nieznaną domenę, np. `zlastrona.com` to jest to podejrzane i może wskazywać na to, że żądanie zostało zainicjowane przez inną stronę - to klasyczny sygnał CSRF, 
- brak referer przy POST, który normalnie go ma - zwykle przeglądarki wysyłają referer przy żądaniach POST, jeśli więc nagle pojawia się POST do wrażliwej akcji i nie ma w nim referer (jest pusty) - to podejrzane. Atakujący może celowo go nie ustawiać, żeby uniknąć wykrycia, normalny użytkownik nie wysyła POST bez referer, 
- zmiana wrażliwych ustawień z obcego referer - zmiany hasła, przekierowanie poczty, zmiana danych do odzyskania, dodanie admina, przelew - to bardzo prawdopodobne, że to CSRF, 
- seria żądań POST z jednego IP do różnych ustawień - zwykły użytkownik nie zmienia zwykle wielu ustawień na raz w ciągu kilku sekund, więc jeśli widoczna jest seria POST z jednego IP do kilku różnych ustawień w krótkim czasie, to może wskazywać na zautomatyzowany atak, zwłaszcza jeśli referer jest z obcej domeny, 
- nietypowe godziny - jeśli użytkownik loguje się zwykle w godzinach pracy, a nagle pojawia się o 3:00 w nocy i zmienia przekierowanie poczty, to może być CSRF, przejęcie sesji - warto to sprawdzić,
- zmiana IP w trakcie jednej sesji - jeśli ta sama sesja nagle pojawia się z innego IP niż zwykle to zawsze jest podejrzane, to może być CSRF z innej lokalizacji albo session hijacking. 

## Jak się bronić
- token CSRF - działa to tak, że serwer generuje token dla każdej sesji i wstawia go do formularza jako ukryte pole. Przy odbiorze żądania sprawdza, czy token się zgadza. Atakujący nie może odczytać tokenu z formularza ofiary, bo blokują go same-origin policy - przeglądarka nie pozwala stronie atakującego czytać treści z innej domeny. Więc atakujący nie może wstawić go do żądania, a bez tego tokenu serwer nie zaakceptuje zmian. Zalogowany użytkownik na swojej poczcie nie widzi tego tokenu, jest on ukrytym polem w formularzu, więc gdy użytkownik zapisuje ustawienia token leci razem z żądaniem. Formularz atakującego nie zawiera tokenu więc wszelkie zmiany nie zostaną zatwierdzone na serwerze, 
- SameSite na ciasteczkach - serwer wysyła ciasteczko z flagą `Set-Cookie: session=abc123; SameSite=Strict` - to oznacza, że ciasteczko nie jest wysyłane przy żadnym żądaniu z obcej strony, (np. ze strony atakującego), 
- sprawdzanie `Origin`/`Referer` - serwer sprawdza, czy żądanie przyszło z tej samej domeny, jeśli nie, odrzuca, 
- potwierdzenie przy wrażliwych akcjach - np. przy zmianie hasła, dodaniu przekierowania lub filtra - wymagane jest ponowne wpisanie hasła lub kodu SMS, 
- powiadomienia o zmianach - mail do ofiary "zmieniono ustawienie x..", co pozwala ofierze na reakcję, 
- WAF - blokuje żądania bez prawidłowego referer / origin lub bez tokenu CSRF, 
- krótkie sesje  - sesja wygasa po C minutach bezczynności, 
- wylogowywanie się - to najprostsza metoda obrony przed tym atakiem, ponieważ atak nie zadziała jeśli użytkownik nie jest zalogowany. 