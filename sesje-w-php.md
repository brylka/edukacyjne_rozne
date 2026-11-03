# Obsługa sesji w skryptach PHP
### Technik Programista | Kwalifikacja INF.03 | Programowanie aplikacji internetowych

---

## Plan materiału

1. Problem: HTTP „nie pamięta” użytkownika
2. Ciasteczka (cookies) – szybkie przypomnienie
3. Czym jest sesja i jak działa pod spodem
4. `session_start()` – pierwszy krok i najczęstsza pułapka
5. Zapisywanie, odczyt i usuwanie danych z `$_SESSION`
6. Przykład 1 – licznik odwiedzin
7. Przykład 2 – koszyk w sklepie
8. Logowanie z użyciem sesji (PDO + `password_verify()`)
9. Ochrona podstron – plik „strażnik”
10. Wylogowanie – poprawne niszczenie sesji
11. Komunikaty jednorazowe (flash messages)
12. Bezpieczeństwo sesji
13. Najczęstsze błędy

---

## 1. Problem: HTTP „nie pamięta” użytkownika

Protokół **HTTP jest bezstanowy**. Każde żądanie przeglądarki do serwera jest traktowane jak zupełnie nowe – serwer nie wie, że ta sama osoba sekundę wcześniej się zalogowała albo dodała produkt do koszyka.

```
Żądanie 1:  przeglądarka → serwer:  POST /logowanie.php  (login, hasło)
            serwer:  "OK, zalogowano!"

Żądanie 2:  przeglądarka → serwer:  GET /panel.php
            serwer:  "A kim ty właściwie jesteś?"
```

Zmienne PHP żyją tylko **do końca wykonania skryptu**. Po wysłaniu strony do przeglądarki wszystkie znikają.

Potrzebujemy więc mechanizmu, który pozwoli **przechować dane pomiędzy żądaniami**. W PHP są dwa podstawowe:

| Mechanizm | Gdzie są dane? | Przykłady użycia |
|---|---|---|
| **Ciasteczka** (cookies) | W przeglądarce użytkownika | Wybrany język, motyw jasny/ciemny, „zapamiętaj mnie” |
| **Sesja** | Na serwerze | Zalogowany użytkownik, koszyk, uprawnienia |

---

## 2. Ciasteczka – szybkie przypomnienie

Ciasteczko to mały kawałek tekstu, który serwer każe przeglądarce zapamiętać. Przeglądarka **odsyła je automatycznie** przy każdym kolejnym żądaniu do tej samej strony.

```php
<?php
// Ustawienie ciasteczka na 30 dni
setcookie("motyw", "ciemny", time() + 60 * 60 * 24 * 30);

// Odczyt (UWAGA: dopiero przy NASTĘPNYM żądaniu!)
$motyw = $_COOKIE['motyw'] ?? 'jasny';
```

### Dlaczego nie trzymamy wszystkiego w ciasteczkach?

| Cecha ciasteczka | Konsekwencja |
|---|---|
| Przechowywane u użytkownika | Użytkownik może je **dowolnie zmienić** (narzędzia deweloperskie F12) |
| Wysyłane przy każdym żądaniu | Zwiększają ruch sieciowy |
| Limit ok. 4 KB | Nie zmieścisz dużego koszyka |

> ⚠️ Gdyby informacja „jestem zalogowany jako admin” była w ciasteczku `rola=admin`, każdy mógłby zmienić ją na `admin` w przeglądarce. **Danym z ciasteczek nie wolno ufać.**

Dlatego ważne dane trzymamy **na serwerze** – w sesji.

---

## 3. Czym jest sesja i jak działa pod spodem?

**Sesja** to dane przechowywane **na serwerze**, przypisane do konkretnego użytkownika (przeglądarki). Użytkownik dostaje tylko **identyfikator sesji** – długi, losowy ciąg znaków.

```
1. Użytkownik wchodzi na stronę.
   PHP: session_start() → nie ma sesji → generuje ID, np. "k3f9a1c0e8b7..."
   PHP tworzy plik na serwerze:   sess_k3f9a1c0e8b7...
   PHP wysyła ciasteczko:         PHPSESSID=k3f9a1c0e8b7...

2. Skrypt zapisuje dane:  $_SESSION['login'] = 'jan';
   Po zakończeniu skryptu PHP zapisuje je do pliku sesji.

3. Kolejne żądanie – przeglądarka sama odsyła ciasteczko PHPSESSID.
   PHP: session_start() → odczytuje plik sess_k3f9a1c0e8b7...
   → $_SESSION['login'] znowu ma wartość 'jan'
```

### Gdzie fizycznie są te dane?

W XAMPP pliki sesji znajdziesz zwykle w katalogu `C:\xampp\tmp`. Zawartość przykładowego pliku:

```
login|s:3:"jan";rola|s:4:"user";
```

To dane zapisane w specjalnym formacie (serializacja). Możesz zajrzeć do takiego pliku – to dobre ćwiczenie, żeby zobaczyć, że sesja to naprawdę „zwykły plik na serwerze”.

### Porównanie

| | Ciasteczko | Sesja |
|---|---|---|
| Miejsce przechowywania | Przeglądarka | Serwer |
| Czy użytkownik może zmienić dane? | **Tak** | Nie (widzi tylko ID) |
| Rozmiar | ok. 4 KB | Praktycznie bez limitu |
| Czas życia | Dowolny (ustawiany w `setcookie`) | Domyślnie do zamknięcia przeglądarki / ok. 24 min bezczynności |
| Dostęp w PHP | `$_COOKIE` | `$_SESSION` |

> **Ważne:** Sesja **też korzysta z ciasteczka** – ale przechowuje w nim tylko identyfikator, a nie dane. Kto pozna ten identyfikator, „staje się” danym użytkownikiem (więcej w punkcie 12).

---

## 4. `session_start()` – pierwszy krok i najczęstsza pułapka

Każdy skrypt, który chce korzystać z sesji, **musi** wywołać `session_start()`:

```php
<?php
session_start();

// od tego miejsca możesz używać $_SESSION
```

Funkcja:
- tworzy nową sesję (jeśli jej nie ma) **albo**
- wznawia istniejącą (jeśli przeglądarka przysłała `PHPSESSID`),
- wypełnia tablicę `$_SESSION` danymi.

### Pułapka: „headers already sent”

`session_start()` wysyła ciasteczko, czyli **nagłówek HTTP**. Nagłówki muszą zostać wysłane **przed jakąkolwiek treścią strony**. Ten kod spowoduje ostrzeżenie:

```php
<!DOCTYPE html>
<html>
<?php
session_start();   // [X] ZA PÓŹNO – HTML już poszedł do przeglądarki
?>
```

Komunikat będzie podobny do:

```
Warning: session_start(): Session cannot be started after headers have already been sent
```

Treścią jest **wszystko** poza `<?php ... ?>` – także pojedyncza spacja albo pusta linia przed `<?php`!

```php
 <?php            // [X] spacja przed <?php to już wysłana treść
session_start();
```

### [OK] Poprawnie

```php
<?php
session_start();
?>
<!DOCTYPE html>
<html>
...
```

> **Zasada:** `session_start()` umieszczamy w **pierwszej linii** pliku, zaraz po `<?php`. Tak samo `header()` i `setcookie()` – zawsze przed HTML-em.

> **Wskazówka:** Plik zapisuj w kodowaniu **UTF-8 bez BOM**. Znacznik BOM to niewidoczne bajty na początku pliku – PHP traktuje je jak treść i pojawia się ten sam błąd.

### Sprawdzanie stanu sesji

Jeśli plik może być dołączany w kilku miejscach, a nie chcesz uruchamiać sesji dwa razy:

```php
if (session_status() === PHP_SESSION_NONE) {
    session_start();
}
```

| Stała | Znaczenie |
|---|---|
| `PHP_SESSION_DISABLED` | Sesje wyłączone na serwerze |
| `PHP_SESSION_NONE` | Sesje działają, ale żadna nie jest uruchomiona |
| `PHP_SESSION_ACTIVE` | Sesja jest już uruchomiona |

---

## 5. Zapisywanie, odczyt i usuwanie danych z `$_SESSION`

`$_SESSION` to zwykła **tablica asocjacyjna** – pracujemy z nią tak samo jak z każdą inną tablicą.

### Zapis

```php
<?php
session_start();

$_SESSION['imie'] = "Anna";
$_SESSION['wiek'] = 17;
$_SESSION['ulubione'] = ["PHP", "SQL", "CSS"];   // można zapisać nawet tablicę
```

### Odczyt

```php
<?php
session_start();

echo $_SESSION['imie'];               // Anna
echo $_SESSION['ulubione'][0];        // PHP
```

Gdy klucza nie ma, dostaniesz ostrzeżenie `Undefined array key`. Dlatego przed odczytem sprawdzamy, czy wartość istnieje:

```php
// Sposób 1 – isset()
if (isset($_SESSION['imie'])) {
    echo "Witaj, " . htmlspecialchars($_SESSION['imie']);
} else {
    echo "Witaj, gościu";
}

// Sposób 2 – operator ?? (null coalescing) – krócej
$imie = $_SESSION['imie'] ?? "gościu";
echo "Witaj, " . htmlspecialchars($imie);
```

### Usuwanie jednej wartości

```php
unset($_SESSION['wiek']);
```

### Usuwanie wszystkich danych (ale bez niszczenia sesji)

```php
$_SESSION = [];
```

### Podsumowanie funkcji

| Funkcja / operacja | Co robi |
|---|---|
| `session_start()` | Uruchamia lub wznawia sesję |
| `$_SESSION['klucz'] = wartość` | Zapisuje daną |
| `isset($_SESSION['klucz'])` | Sprawdza, czy dana istnieje |
| `unset($_SESSION['klucz'])` | Usuwa jedną daną |
| `$_SESSION = []` | Czyści wszystkie dane sesji |
| `session_destroy()` | Usuwa sesję (plik na serwerze) – patrz punkt 10 |
| `session_id()` | Zwraca identyfikator bieżącej sesji |
| `session_name()` | Zwraca nazwę ciasteczka sesji (domyślnie `PHPSESSID`) |
| `session_regenerate_id(true)` | Nadaje sesji nowy identyfikator – patrz punkt 12 |
| `session_status()` | Zwraca stan sesji |

---

## 6. Przykład 1 – licznik odwiedzin

Najprostszy przykład pokazujący, że dane „przeżywają” między odświeżeniami strony.

```php
<?php
// licznik.php
session_start();

if (!isset($_SESSION['odwiedziny'])) {
    $_SESSION['odwiedziny'] = 0;
}

$_SESSION['odwiedziny']++;
?>
<!DOCTYPE html>
<html lang="pl">
<head><meta charset="UTF-8"><title>Licznik</title></head>
<body>
    <p>Odwiedziłeś tę stronę <?= $_SESSION['odwiedziny'] ?> raz(y) w tej sesji.</p>
    <p><a href="licznik.php">Odśwież</a></p>
</body>
</html>
```

Sprawdź:
1. Odśwież stronę kilka razy – licznik rośnie.
2. Otwórz stronę w **trybie incognito** lub innej przeglądarce – licznik zaczyna od 1. To inna sesja, inny `PHPSESSID`.
3. Otwórz narzędzia deweloperskie (F12 → Aplikacja/Storage → Cookies) i znajdź ciasteczko `PHPSESSID`.

> `<?= ... ?>` to skrót od `<?php echo ... ?>`.

---

## 7. Przykład 2 – koszyk w sklepie

Koszyk to klasyczne zastosowanie sesji. Przechowujemy w nim tablicę: **ID produktu → ilość**.

```php
<?php
// koszyk.php
session_start();

// Upewniamy się, że koszyk istnieje
$_SESSION['koszyk'] = $_SESSION['koszyk'] ?? [];

// Dodawanie produktu: koszyk.php?dodaj=5
if (isset($_GET['dodaj'])) {
    $id = (int)$_GET['dodaj'];          // rzutowanie – ID ma być liczbą
    if ($id > 0) {
        $_SESSION['koszyk'][$id] = ($_SESSION['koszyk'][$id] ?? 0) + 1;
    }
}

// Usuwanie produktu: koszyk.php?usun=5
if (isset($_GET['usun'])) {
    unset($_SESSION['koszyk'][(int)$_GET['usun']]);
}

// Opróżnianie koszyka: koszyk.php?wyczysc=1
if (isset($_GET['wyczysc'])) {
    $_SESSION['koszyk'] = [];
}
?>
<!DOCTYPE html>
<html lang="pl">
<head><meta charset="UTF-8"><title>Koszyk</title></head>
<body>
    <h1>Twój koszyk</h1>

    <?php if (empty($_SESSION['koszyk'])): ?>
        <p>Koszyk jest pusty.</p>
    <?php else: ?>
        <ul>
        <?php foreach ($_SESSION['koszyk'] as $id => $ilosc): ?>
            <li>
                Produkt nr <?= $id ?> – sztuk: <?= $ilosc ?>
                <a href="?usun=<?= $id ?>">[usuń]</a>
            </li>
        <?php endforeach; ?>
        </ul>
        <p><a href="?wyczysc=1">Opróżnij koszyk</a></p>
    <?php endif; ?>

    <p>
        <a href="?dodaj=1">Dodaj produkt 1</a> |
        <a href="?dodaj=2">Dodaj produkt 2</a>
    </p>
</body>
</html>
```

> W prawdziwym sklepie nazwy i ceny produktów pobierasz z bazy (PDO, `WHERE id IN (...)`). W sesji trzymamy tylko **ID i ilość** – **nigdy ceny**, bo cena może się zmienić i powinna zawsze pochodzić z bazy.

> **Uwaga do ćwiczeń:** modyfikowanie danych przez link `GET` jest tu uproszczeniem. W poprawnie zaprojektowanej aplikacji operacje zmieniające stan (dodaj, usuń) wysyła się formularzem metodą `POST`.

---

## 8. Logowanie z użyciem sesji

Łączymy wiedzę z materiału o PDO: zapytanie parametryzowane + hash hasła + sesja.

Tabela (jak w materiale o PDO, z dodaną rolą):

```sql
CREATE TABLE uzytkownicy (
    id INT AUTO_INCREMENT PRIMARY KEY,
    login VARCHAR(50) NOT NULL UNIQUE,
    haslo VARCHAR(255) NOT NULL,
    rola VARCHAR(20) NOT NULL DEFAULT 'user'
);
```

### Formularz i obsługa logowania – `logowanie.php`

```php
<?php
session_start();
require "db.php";   // połączenie PDO – plik z materiału o PDO

$blad = "";

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $login = trim($_POST['login'] ?? '');
    $haslo = $_POST['haslo'] ?? '';

    $stmt = $pdo->prepare("SELECT id, login, haslo, rola FROM uzytkownicy WHERE login = :login");
    $stmt->execute(['login' => $login]);
    $user = $stmt->fetch();

    if ($user && password_verify($haslo, $user['haslo'])) {
        session_regenerate_id(true);          // nowy identyfikator sesji – patrz punkt 12

        $_SESSION['user_id'] = $user['id'];
        $_SESSION['login']   = $user['login'];
        $_SESSION['rola']    = $user['rola'];

        header("Location: panel.php");
        exit;                                  // po header() ZAWSZE exit
    } else {
        $blad = "Błędny login lub hasło";
    }
}
?>
<!DOCTYPE html>
<html lang="pl">
<head><meta charset="UTF-8"><title>Logowanie</title></head>
<body>
    <h1>Logowanie</h1>

    <?php if ($blad): ?>
        <p style="color:red"><?= htmlspecialchars($blad) ?></p>
    <?php endif; ?>

    <form method="post">
        <label>Login: <input type="text" name="login" required></label><br>
        <label>Hasło: <input type="password" name="haslo" required></label><br>
        <button type="submit">Zaloguj</button>
    </form>
</body>
</html>
```

### Co trzymać w sesji, a czego nie?

| [OK] W sesji | [X] Nie w sesji |
|---|---|
| ID użytkownika | Hasło (nawet zahashowane) |
| Login (do wyświetlenia) | Numer karty, PESEL itp. |
| Rola / uprawnienia | Dane, które często się zmieniają w bazie (np. stan konta) |

> **Zasada:** w sesji trzymamy **minimum** potrzebne do rozpoznania użytkownika. Resztę pobieramy z bazy, gdy jest potrzebna.

### Dlaczego `exit` po `header("Location: ...")`?

`header()` tylko **wysyła przeglądarce informację** o przekierowaniu. Skrypt PHP **wykonuje się dalej**! Bez `exit` kod poniżej (np. treść chronionej strony) i tak zostanie wykonany i wysłany.

---

## 9. Ochrona podstron – plik „strażnik”

Każda strona dostępna tylko dla zalogowanych musi to sprawdzać. Zamiast kopiować ten sam kod do każdego pliku, tworzymy jeden plik i go dołączamy.

### `auth.php`

```php
<?php
// auth.php – dołącz na początku każdej chronionej strony
if (session_status() === PHP_SESSION_NONE) {
    session_start();
}

if (!isset($_SESSION['user_id'])) {
    header("Location: logowanie.php");
    exit;
}
```

### `panel.php`

```php
<?php
require "auth.php";   // jeśli niezalogowany – tu skrypt się zakończy
?>
<!DOCTYPE html>
<html lang="pl">
<head><meta charset="UTF-8"><title>Panel</title></head>
<body>
    <h1>Witaj, <?= htmlspecialchars($_SESSION['login']) ?>!</h1>
    <p><a href="wyloguj.php">Wyloguj</a></p>
</body>
</html>
```

### Strona tylko dla administratora – `admin.php`

```php
<?php
require "auth.php";

if ($_SESSION['rola'] !== 'admin') {
    http_response_code(403);
    die("Brak uprawnień.");
}
?>
<h1>Panel administratora</h1>
```

> ⚠️ Ukrycie linku „Panel admina” w menu **nie jest zabezpieczeniem**. Ktoś może wpisać adres `admin.php` ręcznie. Uprawnienia sprawdzamy **zawsze po stronie serwera, w samym pliku**.

---

## 10. Wylogowanie – poprawne niszczenie sesji

Częsty błąd: samo `session_destroy()`. Zobaczmy, co dokładnie robią poszczególne kroki:

| Operacja | Co usuwa |
|---|---|
| `$_SESSION = []` | Dane w tablicy `$_SESSION` w bieżącym skrypcie |
| `session_destroy()` | Plik sesji na serwerze (ale **nie** czyści `$_SESSION` w bieżącym skrypcie i **nie** usuwa ciasteczka) |
| `setcookie(session_name(), '', time() - 42000, ...)` | Ciasteczko `PHPSESSID` w przeglądarce |

### Pełne wylogowanie – `wyloguj.php`

```php
<?php
session_start();              // żeby zniszczyć sesję, trzeba ją najpierw uruchomić

// 1. Czyścimy dane
$_SESSION = [];

// 2. Usuwamy ciasteczko sesji (ustawiamy datę wygaśnięcia w przeszłości)
if (ini_get("session.use_cookies")) {
    $p = session_get_cookie_params();
    setcookie(session_name(), '', time() - 42000,
        $p["path"], $p["domain"], $p["secure"], $p["httponly"]
    );
}

// 3. Niszczymy sesję na serwerze
session_destroy();

header("Location: logowanie.php");
exit;
```

> Ten schemat pochodzi z oficjalnej dokumentacji PHP (opis funkcji `session_destroy()`). Na egzaminie i w praktyce wystarczy często `session_start(); session_unset(); session_destroy();` – ale warto wiedzieć, że ciasteczko w przeglądarce wtedy zostaje.

---

## 11. Komunikaty jednorazowe (flash messages)

Po dodaniu rekordu chcemy wyświetlić „Zapisano!”, ale **tylko raz** – nie przy każdym odświeżeniu. Sesja nadaje się do tego idealnie.

Przy okazji stosujemy wzorzec **PRG** (*Post / Redirect / Get*): po przetworzeniu formularza przekierowujemy, żeby odświeżenie strony (F5) nie wysłało formularza drugi raz.

```php
<?php
// dodaj.php
session_start();

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // ... zapis do bazy ...

    $_SESSION['komunikat'] = "Produkt został dodany.";   // zapisujemy komunikat
    header("Location: dodaj.php");                         // Redirect
    exit;
}

// Get – odczytujemy komunikat i od razu go usuwamy
$komunikat = $_SESSION['komunikat'] ?? null;
unset($_SESSION['komunikat']);
?>
<!DOCTYPE html>
<html lang="pl">
<head><meta charset="UTF-8"><title>Dodaj produkt</title></head>
<body>
    <?php if ($komunikat): ?>
        <p style="color:green"><?= htmlspecialchars($komunikat) ?></p>
    <?php endif; ?>

    <form method="post">
        <input type="text" name="nazwa" required>
        <button>Dodaj</button>
    </form>
</body>
</html>
```

```
POST dodaj.php  →  zapis + komunikat w sesji  →  302 Location: dodaj.php
GET  dodaj.php  →  wyświetl komunikat, usuń go z sesji
F5              →  GET dodaj.php  →  komunikatu już nie ma, nic się nie dubluje
```

---

## 12. Bezpieczeństwo sesji

Identyfikator sesji działa jak **klucz do konta**. Kto go zdobędzie, jest zalogowany jako ofiara – bez znajomości hasła.

### Zagrożenia

| Atak | Na czym polega | Obrona |
|---|---|---|
| **Przechwycenie sesji** (*session hijacking*) | Atakujący kradnie `PHPSESSID` – np. przez XSS albo podsłuch niezaszyfrowanego połączenia | `httponly`, HTTPS + `secure`, `htmlspecialchars()` na wyjściu |
| **Narzucenie sesji** (*session fixation*) | Atakujący podsuwa ofierze znany sobie ID sesji, ofiara się loguje, a atakujący używa tego samego ID | `session_regenerate_id(true)` po zalogowaniu |
| **Pozostawiona sesja** | Ktoś nie wylogował się na wspólnym komputerze w pracowni | Wylogowanie po czasie bezczynności |

### 1. Nowy identyfikator po zalogowaniu

```php
if ($user && password_verify($haslo, $user['haslo'])) {
    session_regenerate_id(true);    // true = usuń stary plik sesji
    $_SESSION['user_id'] = $user['id'];
}
```

Identyfikator sprzed logowania (który atakujący mógł znać) przestaje być ważny.

### 2. Bezpieczne ustawienia ciasteczka sesji

Ustawiamy je **przed** `session_start()`:

```php
<?php
session_set_cookie_params([
    'lifetime' => 0,                       // do zamknięcia przeglądarki
    'path'     => '/',
    'secure'   => isset($_SERVER['HTTPS']),// tylko przez HTTPS (gdy jest dostępne)
    'httponly' => true,                    // JavaScript nie odczyta ciasteczka
    'samesite' => 'Lax',                   // ogranicza wysyłanie z obcych stron
]);
session_start();
```

| Opcja | Chroni przed |
|---|---|
| `httponly` | Kradzieżą ID sesji przez wstrzyknięty JavaScript (`document.cookie`) |
| `secure` | Wysłaniem ID sesji przez niezaszyfrowane HTTP |
| `samesite` | Częścią ataków CSRF (wysyłaniem żądań z cudzej strony) |

### 3. Wylogowanie po bezczynności

```php
<?php
// fragment auth.php
$limit = 15 * 60;   // 15 minut

if (isset($_SESSION['ostatnia_aktywnosc'])
    && time() - $_SESSION['ostatnia_aktywnosc'] > $limit) {
    $_SESSION = [];
    session_destroy();
    header("Location: logowanie.php?wygasla=1");
    exit;
}

$_SESSION['ostatnia_aktywnosc'] = time();
```

### 4. Nie pokazuj ID sesji w adresie URL

Adres typu `panel.php?PHPSESSID=k3f9a1...` może trafić do historii, zakładek albo zostać komuś wysłany. Domyślna konfiguracja PHP (`session.use_only_cookies = 1`) przesyła identyfikator wyłącznie w ciasteczku – nie zmieniaj tego.

> **Dwa poziomy ochrony:**
> - **Hasło** chronimy hashem (`password_hash()`) i zapytaniami parametryzowanymi.
> - **Sesję** chronimy: `session_regenerate_id()`, `httponly`, HTTPS, limitem czasu i ochroną przed XSS (`htmlspecialchars()`).
>
> XSS jest groźny podwójnie – wstrzyknięty skrypt może wykraść sesję. Dlatego każde `echo` danych od użytkownika przechodzi przez `htmlspecialchars()`.

---

## 13. Najczęstsze błędy

| Błąd | Dlaczego źle | Poprawnie |
|---|---|---|
| Brak `session_start()` w pliku | `$_SESSION` jest puste, „sesja nie działa” | `session_start()` w **każdym** pliku korzystającym z sesji |
| `session_start()` po HTML-u / spacja przed `<?php` | Ostrzeżenie *headers already sent*, sesja nie działa | `session_start()` w pierwszej linii, plik UTF-8 bez BOM |
| `header("Location: ...")` bez `exit` | Skrypt wykonuje się dalej – chroniona treść może wyciec | Zawsze `exit;` po przekierowaniu |
| Odczyt `$_SESSION['x']` bez sprawdzenia | Ostrzeżenie *Undefined array key* | `isset()` albo `??` |
| Samo `session_destroy()` przy wylogowaniu | Dane w bieżącym skrypcie i ciasteczko zostają | Pełne wylogowanie z punktu 10 |
| Trzymanie hasła w sesji | Niepotrzebne ryzyko wycieku | Tylko ID, login, rola |
| Uprawnienia w ciasteczku (`rola=admin`) | Użytkownik może je zmienić | Uprawnienia w `$_SESSION` |
| Ukrywanie linku zamiast sprawdzania uprawnień | Adres można wpisać ręcznie | Sprawdzanie roli w każdym chronionym pliku |
| Brak `session_regenerate_id()` po zalogowaniu | Podatność na *session fixation* | `session_regenerate_id(true)` |
| `echo $_SESSION['login']` bez zabezpieczenia | XSS – login mógł zawierać kod HTML/JS | `htmlspecialchars($_SESSION['login'])` |

> ⚠️ **Najważniejsza pułapka:** sesja nie „wie”, że użytkownik zamknął kartę. Jeśli nie zadbasz o wylogowanie i limit czasu, sesja na wspólnym komputerze może zostać przejęta przez kolejną osobę.

---

## 14. Podsumowanie

- HTTP jest **bezstanowy** – sesje i ciasteczka pozwalają „pamiętać” użytkownika.
- **Ciasteczka** są u użytkownika (można je zmienić), **sesja** jest na serwerze (użytkownik ma tylko ID).
- `session_start()` – na samym początku pliku, przed jakimkolwiek HTML-em.
- `$_SESSION` to zwykła tablica: zapis `=`, sprawdzanie `isset()` / `??`, usuwanie `unset()`.
- Ochrona strony = plik „strażnik” sprawdzający `$_SESSION['user_id']` + `header()` + `exit`.
- Pełne wylogowanie: wyczyszczenie `$_SESSION`, usunięcie ciasteczka, `session_destroy()`.
- Bezpieczeństwo: `session_regenerate_id(true)` po zalogowaniu, `httponly`, `secure`, limit bezczynności, `htmlspecialchars()`.

---

## Pytania sprawdzające

1. Co oznacza, że protokół HTTP jest bezstanowy? Dlaczego to problem przy logowaniu?
2. Wymień trzy różnice między ciasteczkiem a sesją.
3. Co przechowuje ciasteczko `PHPSESSID`, a co plik sesji na serwerze?
4. Dlaczego `session_start()` musi zostać wywołane przed jakąkolwiek treścią HTML? Co się stanie w przeciwnym razie?
5. Jak sprawdzić, czy w sesji istnieje zmienna `koszyk`? Podaj dwa sposoby.
6. Dlaczego po `header("Location: ...")` należy wywołać `exit`?
7. Jakie dane powinny, a jakie nie powinny być przechowywane w sesji zalogowanego użytkownika?
8. Co dokładnie robi `session_destroy()`, a czego **nie** robi?
9. Na czym polega atak *session fixation* i jak się przed nim bronić?
10. Do czego służą flagi `httponly` i `secure` ciasteczka sesji?
11. Czym jest wzorzec PRG i jaki problem rozwiązuje?

---

## Zadania do wykonania

**Zadanie 1.** Napisz `licznik.php` z punktu 6. Sprawdź jego działanie w zwykłym oknie i w trybie incognito. Znajdź plik sesji w katalogu `C:\xampp\tmp` i obejrzyj jego zawartość.

**Zadanie 2.** Utwórz stronę `imie.php` z formularzem, w którym użytkownik podaje imię. Po wysłaniu zapisz imię w sesji. Na drugiej stronie `powitanie.php` wyświetl „Witaj, [imię]!” albo „Witaj, gościu!”, jeśli imienia nie ma w sesji.

**Zadanie 3.** Rozbuduj koszyk z punktu 7: pobierz nazwy i ceny produktów z bazy (PDO) i wyświetl łączną wartość koszyka. Dodawanie i usuwanie zrealizuj formularzami `POST`.

**Zadanie 4.** Zbuduj system logowania: `rejestracja.php`, `logowanie.php`, `auth.php`, `panel.php`, `wyloguj.php`. Hasła hashuj `password_hash()`, po zalogowaniu wywołaj `session_regenerate_id(true)`.

**Zadanie 5.** Dodaj do systemu z zadania 4 stronę `admin.php` dostępną tylko dla roli `admin` oraz automatyczne wylogowanie po 5 minutach bezczynności.

**Zadanie 6.** Na stronie dodawania produktu zastosuj wzorzec PRG i komunikat flash. Sprawdź, czy po naciśnięciu F5 produkt nie dodaje się ponownie.

---

## Materiały dodatkowe

- Dokumentacja – obsługa sesji (PL): https://www.php.net/manual/pl/book.session.php
- `session_start()`: https://www.php.net/manual/pl/function.session-start.php
- `session_destroy()` (z przykładem pełnego wylogowania): https://www.php.net/manual/pl/function.session-destroy.php
- `session_regenerate_id()`: https://www.php.net/manual/pl/function.session-regenerate-id.php
- Bezpieczeństwo sesji w dokumentacji PHP: https://www.php.net/manual/pl/session.security.php
- OWASP – Session Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html

---

*Materiał przygotowany dla kwalifikacji INF.03 – Tworzenie i administrowanie stronami i aplikacjami internetowymi oraz bazami danych*

*Opracował: Bartosz Bryniarski*
