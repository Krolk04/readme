# Zynced – komunikator desktopowy (Python) + serwer API i strona (PHP)

Kompletny projekt: **aplikacja na Windows** (Python + CustomTkinter), **bezpieczne API w PHP** (działa na zwykłym hostingu, bez VPS),
**strona główna z pobieraniem**, **instalator .exe z licencjami**, **automatyczne aktualizacje** i **panel administratora**.

> ⚠️ **Ważne – bezpieczeństwo:** hasło do bazy MySQL wkleiłeś w wiadomości do mnie, więc traktuj je jako *skompromitowane*.
> **Zmień hasło bazy w panelu hostingu** i wpisz nowe tylko do `web/includes/config.php`. W aplikacji desktopowej **nie ma** żadnych haseł do bazy.

---

## 1. Jak to działa (architektura)

```
 [Aplikacja Zynced.exe]  --HTTPS+token-->  [web/api/index.php]  --MySQL-->  [Baza danych]
        (klient)                              (jedyne miejsce z hasłem do bazy)
```

* Aplikacja **nigdy** nie łączy się z bazą. Rozmawia z API po HTTPS, a API sprawdza tokeny, uprawnienia i limity.
* Użytkownik końcowy nie ma jak zdobyć haseł do bazy – leżą tylko w `web/includes/config.php` (folder zablokowany `.htaccess`).
* **Wiadomości „na żywo”** – bez VPS i bez WebSocketów: aplikacja odpytuje API co ~1,5 s (lekki *polling*, tylko nowe dane). W tle zwalnia do 4,5 s.
* **Połączenia i udostępnianie ekranu** – bez własnego serwera głosowego: używamy **Jitsi (WebRTC)**. Aplikacja tworzy pokój i otwiera go w przeglądarce.
  Jakość ekranu (720p/30 → 1080p/60 → 4K/60) ustawia serwer według planu użytkownika.

## 2. Struktura plików

```
zynced/
├─ README.md                  <- ten plik
├─ db/db.sql                  <- schemat bazy (wklej w phpMyAdmin → SQL). Nowe zmiany dopisuję na dole.
├─ web/                       <- CAŁOŚĆ wrzucasz na hosting PHP (public_html)
│  ├─ index.php               strona główna + pobieranie (PL/EN)
│  ├─ legal.php               regulamin, polityka prywatności, licencja
│  ├─ file.php                bezpieczne pobieranie plików (podpisane linki)
│  ├─ payment_webhook.php     automatyczne przyznawanie planów po płatności
│  ├─ api/index.php           API dla aplikacji
│  ├─ includes/               biblioteki + config.php (ZABLOKOWANE przez .htaccess)
│  ├─ lang/                   tłumaczenia strony (dodaj plik xx.php = nowy język)
│  ├─ legal/                  teksty prawne (.txt) – uzupełnij dane firmy!
│  ├─ storage/                pliki użytkowników (ZABLOKOWANE, poza dostępem z sieci)
│  ├─ data/release.json       informacja o najnowszej wersji aplikacji (ZABLOKOWANE)
│  ├─ downloads/              tu trafia Zynced-Setup.exe
│  └─ cron/cleanup.php        sprzątanie wygasłych plików/subskrypcji
├─ client/                    <- aplikacja Python
│  ├─ configplik.py           WSZYSTKIE ustawienia klienta (API_URL, wersja, kolory, interwały…)
│  ├─ main.py                 start aplikacji, aktualizacje
│  ├─ core.py                 API, sesja, języki, cache, updater
│  ├─ ui.py                   widgety, motyw, animacje
│  ├─ pages_auth.py           logowanie/rejestracja (japoński klimat)
│  ├─ pages_chat.py           główny ekran: serwery, czat, DM, GIFy, emotki, pliki
│  ├─ pages_panels.py         ustawienia, ustawienia serwera, sklep, plany, znajomi
│  ├─ pages_admin.py          panel administratora
│  └─ lang/pl.json, en.json   tłumaczenia aplikacji (dodaj plik xx.json = nowy język)
└─ installer/                 <- instalator
   ├─ build.bat               dwuklik: buduje aplikację + instalator
   ├─ build_installer.py      cały proces (ikony, RTF z licencji, PyInstaller, Inno Setup, SHA-256)
   └─ zynced.iss              skrypt Inno Setup (licencja → regulamin → prywatność → instalacja)
```

## 3. Wdrożenie krok po kroku

### Krok 1 – baza danych
1. W panelu hostingu otwórz **phpMyAdmin** → wybierz swoją bazę → zakładka **SQL**.
2. Wklej całą zawartość `db/db.sql` → **Wykonaj**. Powstaną tabele z prefiksem `zy_` (nie ruszają Twoich starych tabel).
3. Po rejestracji swojego konta nadaj sobie admina (zapytanie SQL):
   ```sql
   UPDATE zy_users SET is_admin=1 WHERE email='twoj@email.pl';
   ```
   Albo ustaw `bootstrap_admin_email` w `config.php` – konto z tym e-mailem dostanie admina już przy rejestracji.

### Krok 2 – strona i API (PHP 8.0+, Apache/LiteSpeed)
1. Wgraj **cały folder `web/`** na hosting (do `public_html` lub podkatalogu) – FTP/menedżer plików.
2. W `web/includes/` skopiuj `config.example.php` → **`config.php`** i uzupełnij:
   * `db` – host, nazwa bazy, użytkownik, **nowe** hasło,
   * `app_url` – adres strony (np. `https://zynced.pl`),
   * `secret`, `cron_key`, `webhook_secret` – losowe długie ciągi (`php -r "echo bin2hex(random_bytes(32));"`),
   * `mail` – **SMTP** skrzynki z hostingu (polecane) – bez tego nie przyjdzie kod weryfikacyjny,
   * `gif_provider` + `gif_key` – darmowy klucz **Tenor** (Google Cloud) lub **Giphy**,
   * `jitsi_domain` – domyślnie `meet.jit.si`.
3. **Włącz HTTPS** (certyfikat Let's Encrypt w panelu) i odkomentuj przekierowanie w `web/.htaccess`.
4. Upewnij się, że folder `web/storage/` jest zapisywalny dla PHP (chmod 750/755) – tu lądują pliki użytkowników.
5. **Cron** (co 15 min): `php /sciezka/do/web/cron/cleanup.php` albo URL `https://twojadomena.pl/cron/cleanup.php?key=TWÓJ_CRON_KEY`.
   Usuwa pliki po 5/10/25/30/90 dniach i kończy wygasłe subskrypcje. (Dodatkowo sprzątanie odpala się losowo przy ruchu.)
6. Sprawdź: `https://twojadomena.pl/api/index.php?a=ping` → powinno zwrócić `{"ok":true,...}`.

> Limity wysyłki: pliki idą w porcjach po 4 MB (`chunk_size`), więc **nie** musisz podnosić `upload_max_filesize` ponad 8 MB.
> Jeśli hosting ma PHP-FPM, użyj dołączonego `.user.ini`.

### Krok 3 – konfiguracja klienta
Otwórz `client/configplik.py` i ustaw:
```python
API_URL  = "https://twojadomena.pl/api/index.php"
SITE_URL = "https://twojadomena.pl"
APP_VERSION = "1.0.0"
```
Tam też zmienisz interwały odświeżania, kolory, emotki, domyślne ustawienia użytkownika itd.

### Krok 4 – uruchom i przetestuj
```
cd client
run_dev.bat          (instaluje zależności i startuje aplikację)
```
Zarejestruj się → kod przyjdzie e-mailem → po weryfikacji jesteś zalogowany.

### Krok 5 – zbuduj instalator
1. Zainstaluj **Python 3.10+** (z opcją *Add to PATH*) i **Inno Setup 6** ( https://jrsoftware.org/isdl.php ).
2. Uzupełnij dane firmy w `web/legal/*.txt` (miejsca `[ ... ]`) – to wchodzi do instalatora i na stronę. **Skonsultuj teksty z prawnikiem.**
3. Dwuklik **`installer/build.bat`**. Wynik: `installer/output/Zynced-Setup.exe`
   (kopia ląduje też w `web/downloads/`, a sumę SHA-256 dostajesz w konsoli oraz w `web/data/release.json.new`).
4. Instalator: ładna grafika, wybór języka (PL/EN), **3 osobne strony z dokumentami (Licencja, Regulamin, Polityka prywatności) – trzeba przewinąć i zaznaczyć „Akceptuję”**, skróty, autostart (opcjonalnie), deinstalator.

### Krok 6 – publikacja
1. Wgraj `Zynced-Setup.exe` do `web/downloads/` (jeśli nie skopiował się automatycznie).
2. Wklej dane z `release.json.new` do **`web/data/release.json`**.
3. Strona główna automatycznie pokaże przycisk „Pobierz dla Windows” z numerem wersji.

## 4. Aktualizacje aplikacji – co wrzucać

Aplikacja przy starcie pyta API (`a=release`) o najnowszą wersję. Jeśli jest nowsza niż `APP_VERSION`, pokazuje pasek
„Dostępna aktualizacja” → pobiera instalator, **sprawdza SHA-256**, uruchamia go po cichu (`/SILENT`) i restartuje aplikację.

**Wydanie nowej wersji (np. 1.1.0):**
1. W `client/configplik.py` zmień `APP_VERSION = "1.1.0"` (i kod aplikacji, jeśli coś zmieniałeś).
2. Uruchom `installer/build.bat`.
3. Wgraj nowy `web/downloads/Zynced-Setup.exe` (nadpisz).
4. Zaktualizuj `web/data/release.json`: `version`, `sha256`, `notes`, opcjonalnie `"mandatory": true` (wymusza aktualizację – brak przycisku „Później”).
5. Jeśli są zmiany w bazie – wklej nowe fragmenty z końca `db/db.sql` w phpMyAdmin.
6. Jeśli zmieniałeś PHP – wgraj zmienione pliki z `web/` (NIE nadpisuj `includes/config.php`).

## 5. Funkcje – przewodnik

| Funkcja | Jak używać |
|---|---|
| Rejestracja / logowanie | Kod 6-cyfrowy przychodzi e-mailem (ważny 15 min, 5 prób). Hasła bcrypt (stare hashe `$2y$` z PHP działają). |
| Serwery | Pasek po lewej: **+** (utwórz / dołącz kodem), **🧭** odkrywaj publiczne. ⚙ przy nazwie serwera = ustawienia. |
| Ustawienia serwera | Przegląd (nazwa, opis, kolory, publiczny, powitanie), Kanały (tekstowe/głosowe, temat, tryb powolny), Role (6 uprawnień), Członkowie (pseudonim, role, wyrzucanie), Zaproszenia, Strefa zagrożenia. |
| Czat na żywo | Nowe wiadomości, edycje i usunięcia pojawiają się same. Wskaźnik „X pisze…” z animowaną kreską. Nieprzeczytane na serwerach/kanałach/DM, dźwięk, toast, miganie paska zadań. |
| Wiadomości prywatne | Kliknij znajomego → 💬. Możesz pisać do znajomych i osób z wspólnego serwera. |
| Emotki / GIFy | 😊 – kolorowe emotki (Twemoji, pobierane i cache’owane), 🎞 – biblioteka GIFów (Tenor/Giphy przez Twój serwer – klucz nie wycieka). Wiadomość z samych emotek = duże obrazki. |
| Pliki | ＋ → wybór pliku → wysyłka porcjami z paskiem postępu i wznawianiem. Limity: Free 3 GB, Basic 6 GB, Elite 10 GB. Auto-usuwanie: 5/10/25/30 dni (Basic/Elite także 90) – ustawiasz w *Ustawienia → Pliki*. |
| Połączenia / ekran | 📞 w DM – dzwoni do rozmówcy (baner „Połączenie przychodzące”). 🖥 – od razu udostępnianie ekranu. Kanał głosowy – kliknij, by dołączyć. |
| Subskrypcje | **Zynced Basic**: 23,50 zł/mies. lub 200 zł/rok (zamiast 282 zł). **Zynced Elite**: 55 zł/mies. lub 576 zł/rok (zamiast 660 zł). Basic/Elite: 1000 ZnCoins co miesiąc, więcej GB, 90 dni plików, ekran 1080p/60 (Basic) lub 4K/60 (Elite). Ceny/limity edytujesz w `web/includes/plans.php`. |
| ZnCoins i sklep | Dekoracje awatara, banera i efekty nicku – za ZnCoins lub PLN. Produkty dodajesz w panelu admina. |
| Języki | *Ustawienia → Język*. PL/EN. **Nowy język:** skopiuj `client/lang/pl.json` jako `de.json`, zmień `"_name"` i wartości – pojawi się sam (strona: `web/lang/de.php`). Pliki można też wrzucić do `%APPDATA%\Zynced\lang\`. |
| Wygląd | Motyw jasny/ciemny, kolor akcentu, skala interfejsu, tryb kompaktowy, ograniczenie animacji. |

### Panel administratora
Widoczny tylko dla kont z `is_admin=1` (ikona 🛡 na pasku serwerów). Wejście z animacją kurtyny „管理”. **Uprawnienia sprawdza serwer** przy każdej akcji.
* **Pulpit** – statystyki na żywo, ping bazy, wolne miejsce.
* **Zgłoszenia** – odrzuć, usuń wiadomość, ostrzeż, wycisz, zbanuj.
* **Użytkownicy** – ostrzeżenia, wyciszenia (godziny/dni/na stałe), bany, **ręczne nadanie planu Basic/Elite**, **dodawanie ZnCoins**, wylogowanie sesji, weryfikacja e-maila, nadawanie/odbieranie admina.
* **Sklep** – dodawanie/edycja/usuwanie produktów.
* **Baza danych** – test połączenia, **reset połączenia**, czyszczenie wygasłych danych, optymalizacja, limity zapytań, lista połączeń. Operacje niebezpieczne (wylogowanie wszystkich, kasowanie starych wiadomości) wymagają wpisania słowa `RESET`.
* **Dziennik audytu** – każda akcja admina jest zapisana (kto, co, IP).

### Płatności prawdziwymi pieniędzmi
1. W `config.php` ustaw `checkout_url` – link do bramki z miejscami `{order}` i `{amount}` (np. Payment Link Stripe/PayU/Przelewy24 z polem referencyjnym).
2. Po opłaceniu bramka (lub Twój skrypt pośredni) woła `payment_webhook.php` metodą POST z JSON `{"order":123,"status":"paid"}` i nagłówkiem
   `X-Signature = HMAC-SHA256(treść, webhook_secret)` → plan/produkt przyznawany **automatycznie**.
3. Bramki mają własne formaty webhooków (np. Stripe) – dopasuj parsowanie w `payment_webhook.php`. Dopóki `checkout_url` jest puste, płatności są wyłączone, a plany możesz nadawać ręcznie w panelu admina.

## 6. Zabezpieczenia (co jest w środku)
* Aplikacja bez dostępu do bazy; hasła DB tylko w `config.php` (`.htaccess` blokuje `includes/`, `storage/`, `data/`, `lang/`).
* Zapytania **PDO prepared statements** (brak SQL injection), hasła **bcrypt**, tokeny sesji w bazie tylko jako SHA-256.
* **Limity zapytań** (anty-brute-force i anty-DDoS warstwy aplikacji): globalnie na IP, logowanie, rejestracja, wysyłanie wiadomości, kody e-mail itd.
* Kody weryfikacyjne: hash, wygasanie, limit prób. Logowanie: stała czasowo weryfikacja hasła (bez zdradzania, czy konto istnieje).
* Pliki: losowe nazwy poza webrootem, **podpisane czasowo linki (HMAC)**, `nosniff`, `Content-Security-Policy: sandbox`, wyłączone wykonywanie PHP w `storage/`.
* Nagłówki bezpieczeństwa, brak CORS, limit rozmiaru JSON, walidacja kolorów/tekstów, GIFy tylko z domen Tenor/Giphy.
* **Uczciwa uwaga o DDoS:** PHP na zwykłym hostingu nie zatrzyma dużego ataku sieciowego. Włącz **Cloudflare** (darmowy plan) przed domeną i ustaw `behind_cloudflare => true`.
* Rób regularne **kopie zapasowe** bazy i `storage/`.

## 7. Znane ograniczenia (uczciwie)
* **Połączenia przez Jitsi** otwierają się w przeglądarce. Publiczny `meet.jit.si` bywa wymagający logowania moderatora – dla stabilności użyj własnej instancji Jitsi lub `8x8.vc` i wpisz domenę w `jitsi_domain`.
* **Odświeżanie** to szybki polling (nie WebSocket) – wystarcza dla setek jednoczesnych użytkowników na przeciętnym hostingu; przy tysiącach rozważ VPS z WebSocketami.
* **Emotki** w tekście wiadomości mieszanej z tekstem są czcionką systemową; kolorowe (Twemoji) są w panelu wyboru i jako „duże emotki”.
* Teksty prawne to **szablony** – uzupełnij dane i daj je do przeglądu prawnikowi przed publicznym startem.
* Aplikacja GUI była pisana bez możliwości uruchomienia jej na Windows w moim środowisku – jeśli zobaczysz błąd, wklej treść z `%APPDATA%\Zynced\zynced.log`, a poprawię.

## 8. Rozwiązywanie problemów
| Objaw | Co zrobić |
|---|---|
| „Serwer niedostępny” na ekranie logowania | Sprawdź `API_URL` w `configplik.py` i otwórz `.../api/index.php?a=ping` w przeglądarce. |
| `server_not_configured` | Brak pliku `web/includes/config.php`. |
| `db_unavailable` | Złe dane bazy w `config.php` / baza wyłączona. Włącz na chwilę `'debug' => true`, sprawdź `error_log`. |
| Nie przychodzi kod e-mail | Skonfiguruj SMTP w `config.php` (`mail.smtp`). Sprawdź spam. |
| GIFy: „nie skonfigurowana” | Wpisz `gif_key` (Tenor/Giphy) w `config.php`. |
| Błąd przy wysyłaniu dużych plików | Sprawdź uprawnienia `storage/` i wolne miejsce; `chunk_size` ≤ `upload_max_filesize`. |
| `Inno Setup nie znaleziony` | Zainstaluj Inno Setup 6 lub ustaw zmienną `ISCC` na ścieżkę do `ISCC.exe`. |
| Błąd w aplikacji | Log: `%APPDATA%\Zynced\zynced.log`. |

---
© Zynced Studio
