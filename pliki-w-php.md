# Zapisywanie i odczytywanie plików w skryptach PHP
### Technik Programista | Kwalifikacja INF.03 | Programowanie aplikacji internetowych

---

## Plan materiału

1. Pliki czy baza danych – kiedy co?
2. Ścieżki do plików i uprawnienia
3. Szybki zapis – `file_put_contents()`
4. Szybki odczyt – `file_get_contents()` i `file()`
5. Praca „krok po kroku” – `fopen()`, `fwrite()`, `fgets()`, `fclose()`
6. Tryby otwarcia pliku
7. Blokowanie plików – `flock()`
8. Formaty danych: CSV i JSON
9. Przykład 1 – księga gości
10. Przykład 2 – licznik odwiedzin
11. Inne przydatne funkcje plikowe
12. Bezpieczeństwo pracy z plikami
13. Najczęstsze błędy

---

## 1. Pliki czy baza danych – kiedy co?

PHP może zapisywać dane nie tylko w bazie, ale też w zwykłych plikach na serwerze. Oba rozwiązania mają swoje miejsce:

| Zastosowanie | Plik | Baza danych |
|---|---|---|
| Logi (dziennik zdarzeń, błędów) | [OK] Idealne – tylko dopisujemy | [!] Możliwe, ale zbędne |
| Konfiguracja, ustawienia | [OK] JSON / INI / PHP | [!] Możliwe |
| Eksport danych (np. do Excela) | [OK] CSV | – |
| Prosty licznik, mała księga gości | [OK] Wystarczy | [OK] |
| Użytkownicy, zamówienia, produkty | [X] Trudne wyszukiwanie, problemy przy wielu użytkownikach | [OK] **Zalecane** |

> **Zasada:** gdy dane trzeba **wyszukiwać, filtrować, łączyć i równocześnie zmieniać** przez wielu użytkowników – wybierz bazę. Gdy dane głównie **dopisujesz** albo **wczytujesz w całości** – plik często wystarczy.

---

## 2. Ścieżki do plików i uprawnienia

### Ścieżka względna a bezwzględna

```php
file_get_contents("dane.txt");            // względna – względem bieżącego katalogu roboczego
file_get_contents(__DIR__ . "/dane.txt"); // bezwzględna – względem katalogu TEGO pliku .php
```

Stała `__DIR__` zawiera pełną ścieżkę do katalogu, w którym leży aktualny plik PHP (np. `C:\xampp\htdocs\projekt`).

> **Dlaczego `__DIR__`?** Ścieżka względna liczona jest od katalogu **uruchomionego** skryptu, a nie od pliku, w którym ją napisałeś. Gdy plik z kodem dołączasz przez `require` z innego katalogu, `"dane.txt"` może wskazywać zupełnie inne miejsce. `__DIR__` zawsze działa tak samo.

### Struktura projektu w przykładach

```
htdocs/projekt/
├── index.php
├── ksiega.php
└── dane/
    ├── wpisy.csv
    ├── licznik.txt
    └── log.txt
```

### Uprawnienia

Na Windowsie z XAMPP zapis zwykle „po prostu działa”. Na serwerze z Linuksem proces serwera WWW (np. użytkownik `www-data`) musi mieć **prawo zapisu** do katalogu `dane/`. Jeśli go nie ma, pojawi się błąd:

```
Warning: file_put_contents(...): Failed to open stream: Permission denied
```

Sprawdzanie przed operacją:

```php
if (!is_writable(__DIR__ . "/dane")) {
    die("Brak uprawnień do zapisu w katalogu dane/");
}
```

---

## 3. Szybki zapis – `file_put_contents()`

Najprostszy sposób zapisu – **jedna funkcja** otwiera plik, zapisuje dane i go zamyka.

```php
file_put_contents(ścieżka, dane, flagi);
```

### Nadpisanie pliku

```php
<?php
$plik = __DIR__ . "/dane/notatka.txt";

file_put_contents($plik, "Pierwsza linia\n");
file_put_contents($plik, "Druga linia\n");   // [!] nadpisuje – w pliku jest tylko "Druga linia"
```

Jeśli pliku nie ma – zostanie **utworzony**. Jeśli jest – jego zawartość zostanie **usunięta** i zastąpiona nową.

### Dopisywanie na końcu – `FILE_APPEND`

```php
file_put_contents($plik, "Pierwsza linia\n", FILE_APPEND);
file_put_contents($plik, "Druga linia\n",    FILE_APPEND);
// w pliku są obie linie
```

### Blokada przy zapisie – `LOCK_EX`

Gdy kilku użytkowników zapisuje do pliku **w tej samej chwili**, dane mogą się wymieszać. Flaga `LOCK_EX` zakłada blokadę na czas zapisu (więcej w punkcie 7). Flagi łączymy operatorem `|`:

```php
file_put_contents($plik, $linia, FILE_APPEND | LOCK_EX);
```

### Przykład – prosty dziennik zdarzeń (log)

```php
<?php
function zapiszLog(string $wiadomosc): void
{
    $linia = date("Y-m-d H:i:s") . " | " . $wiadomosc . PHP_EOL;
    file_put_contents(__DIR__ . "/dane/log.txt", $linia, FILE_APPEND | LOCK_EX);
}

zapiszLog("Uruchomiono stronę główną");
zapiszLog("Nieudane logowanie: login=" . ($_POST['login'] ?? '-'));
```

Zawartość `log.txt`:

```
2026-11-03 10:15:02 | Uruchomiono stronę główną
2026-11-03 10:15:40 | Nieudane logowanie: login=jan
```

> `PHP_EOL` to znak końca linii właściwy dla systemu (`\r\n` na Windowsie, `\n` na Linuksie). W plikach danych możesz też po prostu używać `"\n"` – ważne, by robić to konsekwentnie.

### Wartość zwracana

`file_put_contents()` zwraca **liczbę zapisanych bajtów** albo `false` przy błędzie:

```php
if (file_put_contents($plik, $dane) === false) {
    echo "Nie udało się zapisać pliku.";
}
```

> ⚠️ Porównujemy przez `=== false`, a nie `!`. Zapis pustego tekstu zwraca `0`, które w zwykłym porównaniu też jest „fałszem”.

---

## 4. Szybki odczyt – `file_get_contents()` i `file()`

### `file_get_contents()` – cały plik jako jeden tekst

```php
<?php
$plik = __DIR__ . "/dane/notatka.txt";

if (file_exists($plik)) {
    $tresc = file_get_contents($plik);
    echo nl2br(htmlspecialchars($tresc));
} else {
    echo "Plik nie istnieje.";
}
```

| Funkcja | Po co |
|---|---|
| `file_exists()` | Bez sprawdzenia – przy braku pliku PHP zgłosi ostrzeżenie, a funkcja zwróci `false` |
| `htmlspecialchars()` | Zawartość pliku mogła pochodzić od użytkownika – ochrona przed XSS |
| `nl2br()` | Zamienia znaki nowej linii na `<br>`, żeby linie były widoczne w przeglądarce |

> Kolejność ma znaczenie: najpierw `htmlspecialchars()`, potem `nl2br()`. W odwrotnej kolejności `<br>` zostałyby zamienione na `&lt;br&gt;` i wyświetlone jako tekst.

### `file()` – plik jako tablica linii

```php
<?php
$linie = file(__DIR__ . "/dane/log.txt", FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);

echo "Liczba wpisów: " . count($linie) . "<br>";

foreach ($linie as $nr => $linia) {
    echo ($nr + 1) . ". " . htmlspecialchars($linia) . "<br>";
}
```

| Flaga | Znaczenie |
|---|---|
| `FILE_IGNORE_NEW_LINES` | Usuwa `\n` z końca każdego elementu |
| `FILE_SKIP_EMPTY_LINES` | Pomija puste linie (działa razem z `FILE_IGNORE_NEW_LINES`) |

### Ostatnie 5 wpisów (najnowsze na górze)

```php
$linie = file(__DIR__ . "/dane/log.txt", FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);
$ostatnie = array_reverse(array_slice($linie, -5));
```

> ⚠️ `file_get_contents()` i `file()` wczytują **cały plik do pamięci**. Dla plików o rozmiarze kilku KB czy MB to nie problem. Plik logów mający kilka GB trzeba czytać linia po linii (punkt 5).

---

## 5. Praca „krok po kroku” – `fopen()`, `fwrite()`, `fgets()`, `fclose()`

Funkcje z punktów 3 i 4 to skróty. Pod spodem praca z plikiem zawsze wygląda tak:

```
1. Otwórz plik       fopen()   → dostajesz "uchwyt" (wskaźnik do pliku)
2. Czytaj / zapisuj  fgets(), fread(), fwrite() ...
3. Zamknij plik      fclose()
```

**Uchwyt** (*handle*, zasób) to coś w rodzaju zakładki w książce – PHP pamięta, który plik jest otwarty i w którym miejscu aktualnie jesteśmy.

### Zapis

```php
<?php
$f = fopen(__DIR__ . "/dane/lista.txt", "w");   // "w" = zapis, wyczyść plik

if ($f === false) {
    die("Nie można otworzyć pliku.");
}

fwrite($f, "Jabłka\n");
fwrite($f, "Gruszki\n");
fwrite($f, "Śliwki\n");

fclose($f);
```

> `fputs()` to inna nazwa (alias) tej samej funkcji co `fwrite()`.

### Odczyt linia po linii

```php
<?php
$f = fopen(__DIR__ . "/dane/lista.txt", "r");   // "r" = tylko odczyt

if ($f === false) {
    die("Nie można otworzyć pliku.");
}

while (($linia = fgets($f)) !== false) {
    echo htmlspecialchars(rtrim($linia)) . "<br>";
}

fclose($f);
```

| Element | Znaczenie |
|---|---|
| `fgets($f)` | Czyta jedną linię (razem z `\n` na końcu); na końcu pliku zwraca `false` |
| `!== false` | Pętla kończy się, gdy nie ma już linii do przeczytania |
| `rtrim()` | Usuwa znak końca linii (i spacje) z prawej strony |

> Ten sposób zużywa mało pamięci – w danej chwili trzymamy **tylko jedną linię**. Dlatego nadaje się do bardzo dużych plików.

### Pozostałe funkcje odczytu

| Funkcja | Co czyta |
|---|---|
| `fgets($f)` | Jedną linię |
| `fgetc($f)` | Jeden znak (bajt) |
| `fread($f, $n)` | `$n` bajtów |
| `stream_get_contents($f)` | Wszystko od bieżącej pozycji do końca |
| `feof($f)` | Zwraca `true`, gdy jesteśmy na końcu pliku |

### Uwaga na `while (!feof($f))`

Często spotkasz taką pętlę:

```php
while (!feof($f)) {          // [!] podatne na błędy
    $linia = fgets($f);
    echo $linia;
}
```

`feof()` zwraca `true` dopiero **po próbie** odczytu za końcem pliku. Ostatni obieg pętli może więc dostać z `fgets()` wartość `false`. Bezpieczniej używać wzorca `while (($linia = fgets($f)) !== false)`.

---

## 6. Tryby otwarcia pliku

Drugi argument `fopen()` określa, **co** chcemy z plikiem robić:

| Tryb | Operacja | Gdy plik nie istnieje | Gdy plik istnieje | Pozycja startowa |
|---|---|---|---|---|
| `r` | odczyt | **błąd** | otwiera | początek |
| `r+` | odczyt i zapis | **błąd** | otwiera | początek |
| `w` | zapis | tworzy | **czyści zawartość** | początek |
| `w+` | odczyt i zapis | tworzy | **czyści zawartość** | początek |
| `a` | dopisywanie | tworzy | otwiera | koniec |
| `a+` | odczyt i dopisywanie | tworzy | otwiera | zapis zawsze na końcu |
| `x` | zapis | tworzy | **błąd** | początek |
| `x+` | odczyt i zapis | tworzy | **błąd** | początek |
| `c` | zapis | tworzy | otwiera **bez czyszczenia** | początek |
| `c+` | odczyt i zapis | tworzy | otwiera **bez czyszczenia** | początek |

Jak zapamiętać:
- **r** – *read* (czytaj),
- **w** – *write* (pisz od nowa),
- **a** – *append* (dopisz),
- **x** – *exclusive* (tylko nowy plik),
- **+** – dodatkowo drugi kierunek (odczyt + zapis).

> ⚠️ **Najczęstsza pomyłka:** użycie `w` zamiast `a`. Tryb `w` **kasuje całą zawartość pliku w chwili otwarcia** – jeszcze zanim cokolwiek zapiszesz.

> Na Windowsie można dodać literę `b` (np. `rb`, `wb`) – plik jest wtedy traktowany binarnie, bez zamiany znaków końca linii. Przy obrazkach i innych plikach binarnych zawsze używaj `b`.

---

## 7. Blokowanie plików – `flock()`

Strona internetowa obsługuje **wielu użytkowników naraz**. Wyobraź sobie licznik odwiedzin:

```
Użytkownik A: odczytuje licznik → 100
Użytkownik B: odczytuje licznik → 100
Użytkownik A: zapisuje 101
Użytkownik B: zapisuje 101        ← jedna wizyta "zginęła"
```

To **wyścig** (*race condition*). Rozwiązaniem jest **blokada** – drugi skrypt czeka, aż pierwszy skończy.

```php
flock($f, LOCK_EX);   // blokada wyłączna – do zapisu (inni czekają)
flock($f, LOCK_SH);   // blokada współdzielona – do odczytu (wielu może czytać naraz)
flock($f, LOCK_UN);   // zwolnienie blokady
```

| Blokada | Kto może jednocześnie czytać? | Kto może jednocześnie pisać? |
|---|---|---|
| `LOCK_SH` (współdzielona) | Inni z `LOCK_SH` | Nikt |
| `LOCK_EX` (wyłączna) | Nikt | Nikt |

> **Ważne:** `flock()` to blokada „umowna” – działa tylko wtedy, gdy **wszystkie** skrypty korzystające z pliku też używają `flock()`. Nie zablokuje np. edycji pliku w Notatniku.

> `file_put_contents(..., LOCK_EX)` chroni sam zapis. Ale schemat „odczytaj → zmień → zapisz” (jak w liczniku) wymaga, by blokada trwała **przez cały czas** – wtedy potrzebne jest `fopen()` + `flock()` (punkt 10).

---

## 8. Formaty danych: CSV i JSON

Zapisywanie „gołego” tekstu jest proste, ale co, jeśli chcemy zapisać **rekord** – np. imię, e-mail i wiadomość? Gdybyśmy rozdzielali pola własnym znakiem (np. `|`), wystarczy, że użytkownik wpisze ten znak lub naciśnie Enter w wiadomości – i format pliku się psuje.

Dlatego korzystamy z gotowych formatów, które PHP obsługuje za nas.

### CSV – dane tabelaryczne

CSV (*Comma-Separated Values*) to tekstowa tabela: jeden wiersz = jeden rekord, pola rozdzielone separatorem. Polska wersja Excela domyślnie używa **średnika** `;`.

```
imie;email;wiadomosc
Anna;anna@example.com;"Świetna strona!"
Jan;jan@example.com;"Wiadomość ze średnikiem; i ""cudzysłowem"""
```

#### Zapis – `fputcsv()`

```php
<?php
$f = fopen(__DIR__ . "/dane/kontakty.csv", "a");

fputcsv($f, ["Anna", "anna@example.com", "Świetna strona!"], ";", '"', "");
fputcsv($f, ["Jan", "jan@example.com", "Tekst; z \"cudzysłowem\""], ";", '"', "");

fclose($f);
```

`fputcsv()` sama zadba o cudzysłowy, gdy pole zawiera separator, cudzysłów lub znak nowej linii.

#### Odczyt – `fgetcsv()`

```php
<?php
$f = fopen(__DIR__ . "/dane/kontakty.csv", "r");

echo "<table border='1'>";
while (($wiersz = fgetcsv($f, null, ";", '"', "")) !== false) {
    echo "<tr>";
    foreach ($wiersz as $pole) {
        echo "<td>" . htmlspecialchars($pole) . "</td>";
    }
    echo "</tr>";
}
echo "</table>";

fclose($f);
```

| Argument | Znaczenie |
|---|---|
| `null` (w `fgetcsv`) | Maksymalna długość linii – `null` = bez limitu |
| `";"` | Separator pól |
| `'"'` | Znak otaczający pole (cudzysłów) |
| `""` | Znak ucieczki – pusty, zgodnie ze standardem CSV |

> **Dlaczego podajemy aż tyle argumentów?** Od PHP 8.4 pominięcie argumentu `escape` w `fputcsv()` / `fgetcsv()` powoduje ostrzeżenie o przestarzałym użyciu (*deprecated*). Podanie wszystkich argumentów jawnie działa poprawnie w każdej wersji PHP 8.

> **Polskie znaki w Excelu:** jeśli Excel pokazuje „krzaczki” w pliku UTF-8, zapisz na samym początku nowego pliku znacznik BOM: `fwrite($f, "\xEF\xBB\xBF");`. Pamiętaj wtedy, że przy odczycie w PHP pierwsze pole pierwszego wiersza będzie zawierało te trzy bajty.

### JSON – dane o dowolnej strukturze

JSON świetnie nadaje się do konfiguracji i zagnieżdżonych danych (tablice w tablicach).

#### Zapis

```php
<?php
$ustawienia = [
    "tytul"     => "Mój sklep",
    "motyw"     => "ciemny",
    "naStronie" => 12,
    "kategorie" => ["AGD", "RTV", "Ogród"],
];

$json = json_encode($ustawienia, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);
file_put_contents(__DIR__ . "/dane/ustawienia.json", $json, LOCK_EX);
```

Zawartość pliku:

```json
{
    "tytul": "Mój sklep",
    "motyw": "ciemny",
    "naStronie": 12,
    "kategorie": [
        "AGD",
        "RTV",
        "Ogród"
    ]
}
```

| Flaga | Znaczenie |
|---|---|
| `JSON_PRETTY_PRINT` | Czytelne wcięcia i nowe linie |
| `JSON_UNESCAPED_UNICODE` | „Ogród” zamiast „Ogr\u00f3d” |

#### Odczyt

```php
<?php
$json = file_get_contents(__DIR__ . "/dane/ustawienia.json");
$ustawienia = json_decode($json, true);   // true = tablica asocjacyjna, a nie obiekt

if ($ustawienia === null) {
    die("Błędny format pliku JSON.");
}

echo $ustawienia['tytul'];          // Mój sklep
echo $ustawienia['kategorie'][2];   // Ogród
```

### Porównanie formatów

| | TXT (linie) | CSV | JSON |
|---|---|---|---|
| Struktura | Brak | Tabela (wiersze × kolumny) | Dowolna, zagnieżdżona |
| Dopisywanie rekordu | [OK] Łatwe | [OK] Łatwe | [!] Trzeba wczytać i zapisać całość |
| Otwarcie w Excelu | – | [OK] | [X] |
| Typowe zastosowanie | Logi | Eksport, listy rekordów | Konfiguracja, dane dla JavaScriptu |
| Funkcje PHP | `fwrite`, `fgets`, `file` | `fputcsv`, `fgetcsv` | `json_encode`, `json_decode` |

---

## 9. Przykład 1 – księga gości

Łączymy wszystko: formularz → walidacja → zapis do CSV z blokadą → odczyt i bezpieczne wyświetlenie. Dodatkowo wzorzec PRG (przekierowanie po `POST`), żeby F5 nie dodawało wpisu drugi raz.

```php
<?php
// ksiega.php
$plik = __DIR__ . "/dane/wpisy.csv";
$bledy = [];

// ---------- ZAPIS ----------
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $imie  = trim($_POST['imie'] ?? '');
    $tresc = trim($_POST['tresc'] ?? '');

    if ($imie === '' || mb_strlen($imie) > 50) {
        $bledy[] = "Imię jest wymagane (maks. 50 znaków).";
    }
    if ($tresc === '' || mb_strlen($tresc) > 500) {
        $bledy[] = "Wpis jest wymagany (maks. 500 znaków).";
    }

    if (!$bledy) {
        $f = fopen($plik, "a");
        if ($f && flock($f, LOCK_EX)) {
            fputcsv($f, [date("Y-m-d H:i"), $imie, $tresc], ";", '"', "");
            flock($f, LOCK_UN);
        }
        if ($f) {
            fclose($f);
        }

        header("Location: ksiega.php");   // PRG
        exit;
    }
}

// ---------- ODCZYT ----------
$wpisy = [];
if (file_exists($plik)) {
    $f = fopen($plik, "r");
    if ($f && flock($f, LOCK_SH)) {
        while (($w = fgetcsv($f, null, ";", '"', "")) !== false) {
            if (count($w) === 3) {          // pomijamy uszkodzone wiersze
                $wpisy[] = $w;
            }
        }
        flock($f, LOCK_UN);
    }
    if ($f) {
        fclose($f);
    }
}
$wpisy = array_reverse($wpisy);   // najnowsze na górze
?>
<!DOCTYPE html>
<html lang="pl">
<head><meta charset="UTF-8"><title>Księga gości</title></head>
<body>
    <h1>Księga gości</h1>

    <?php foreach ($bledy as $b): ?>
        <p style="color:red"><?= htmlspecialchars($b) ?></p>
    <?php endforeach; ?>

    <form method="post">
        <label>Imię: <input type="text" name="imie" maxlength="50" required></label><br>
        <label>Wpis:<br><textarea name="tresc" rows="4" cols="40" maxlength="500" required></textarea></label><br>
        <button type="submit">Dodaj wpis</button>
    </form>

    <hr>

    <?php if (!$wpisy): ?>
        <p>Brak wpisów – bądź pierwszy!</p>
    <?php endif; ?>

    <?php foreach ($wpisy as [$data, $imie, $tresc]): ?>
        <div>
            <strong><?= htmlspecialchars($imie) ?></strong>
            <small>(<?= htmlspecialchars($data) ?>)</small>
            <p><?= nl2br(htmlspecialchars($tresc)) ?></p>
        </div>
    <?php endforeach; ?>
</body>
</html>
```

Co tu jest ważne:
- **walidacja** długości – bez niej ktoś mógłby wysłać wiadomość o rozmiarze wielu megabajtów i zapchać dysk,
- `mb_strlen()` zamiast `strlen()` – liczy **znaki**, a nie bajty (polska litera w UTF-8 to 2 bajty),
- **CSV** – Enter albo średnik w treści wpisu nie psują formatu pliku,
- **blokady** `LOCK_EX` przy zapisie i `LOCK_SH` przy odczycie,
- `htmlspecialchars()` przy **każdym** wyświetleniu – wpis z `<script>` zostanie pokazany jako tekst,
- `foreach ($wpisy as [$data, $imie, $tresc])` – rozpakowanie tablicy od razu do trzech zmiennych.

---

## 10. Przykład 2 – licznik odwiedzin (wersja odporna na wyścig)

Tutaj cała operacja „odczytaj → zwiększ → zapisz” musi odbyć się **pod jedną blokadą**.

```php
<?php
// licznik.php
$plik = __DIR__ . "/dane/licznik.txt";

$f = fopen($plik, "c+");      // odczyt + zapis, utwórz jeśli brak, NIE czyść
$licznik = 0;

if ($f && flock($f, LOCK_EX)) {
    $licznik = (int)stream_get_contents($f);   // odczyt (pusty plik → 0)
    $licznik++;

    ftruncate($f, 0);       // wyczyść plik
    rewind($f);             // wróć na początek
    fwrite($f, (string)$licznik);
    fflush($f);             // wymuś zapis na dysk przed zwolnieniem blokady

    flock($f, LOCK_UN);
}
if ($f) {
    fclose($f);
}

echo "Liczba odwiedzin: " . $licznik;
```

### Dlaczego `c+`, a nie `w+` lub `r+`?

| Tryb | Problem |
|---|---|
| `r+` | Przy pierwszym uruchomieniu pliku nie ma → **błąd** |
| `w+` | Czyści plik **od razu przy otwarciu**, jeszcze przed założeniem blokady → inny skrypt może w tej chwili odczytać pusty plik |
| `c+` | Tworzy plik, gdy go brak, i **nie czyści** go – czyścimy sami (`ftruncate`) dopiero pod blokadą |

### Wersja „szkolna” (bez ochrony przed wyścigiem)

Na lekcji często spotkasz krótszy zapis:

```php
$licznik = (int)@file_get_contents($plik);
file_put_contents($plik, $licznik + 1);
```

Działa dla jednej osoby testującej stronę, ale przy wielu jednoczesnych odwiedzinach może gubić wizyty lub chwilowo „zerować” licznik. Warto rozumieć, **dlaczego** wersja z `flock()` jest poprawna.

---

## 11. Inne przydatne funkcje plikowe

| Funkcja | Co robi |
|---|---|
| `file_exists($p)` | Czy plik lub katalog istnieje |
| `is_file($p)` / `is_dir($p)` | Czy to plik / katalog |
| `is_readable($p)` / `is_writable($p)` | Czy można czytać / zapisywać |
| `filesize($p)` | Rozmiar w bajtach |
| `filemtime($p)` | Czas ostatniej modyfikacji (znacznik czasu – użyj z `date()`) |
| `copy($z, $do)` | Kopiuje plik |
| `rename($z, $do)` | Zmienia nazwę lub przenosi plik |
| `unlink($p)` | **Usuwa** plik |
| `mkdir($p, 0755, true)` | Tworzy katalog (`true` = także katalogi nadrzędne) |
| `rmdir($p)` | Usuwa **pusty** katalog |
| `scandir($p)` | Lista plików w katalogu (zawiera też `.` i `..`) |
| `glob("dane/*.txt")` | Lista plików pasujących do wzorca |
| `basename($p)` | Sama nazwa pliku ze ścieżki |
| `pathinfo($p)` | Tablica: katalog, nazwa, rozszerzenie |

### Przykład – lista plików tekstowych z datą modyfikacji

```php
<?php
foreach (glob(__DIR__ . "/dane/*.txt") as $sciezka) {
    $nazwa = basename($sciezka);
    $kb    = round(filesize($sciezka) / 1024, 1);
    $data  = date("Y-m-d H:i", filemtime($sciezka));

    echo htmlspecialchars($nazwa) . " – $kb KB – zmieniony: $data<br>";
}
```

---

## 12. Bezpieczeństwo pracy z plikami

### Zagrożenie 1: Path Traversal – „wędrowanie” po katalogach

Strona wyświetla plik, którego nazwę podaje użytkownik:

```php
// [X] NIEBEZPIECZNE
$plik = $_GET['plik'];
echo file_get_contents(__DIR__ . "/dane/" . $plik);
```

Adres `pokaz.php?plik=../../../xampp/apache/conf/httpd.conf` – każde `../` to „katalog wyżej”. Atakujący może odczytać **dowolny plik** na serwerze, do którego PHP ma dostęp, np. konfigurację z hasłem do bazy (`db.php`).

#### [OK] Obrona 1 – biała lista (najlepsza)

```php
$dozwolone = [
    'regulamin' => 'regulamin.txt',
    'kontakt'   => 'kontakt.txt',
];

$klucz = $_GET['plik'] ?? '';

if (!isset($dozwolone[$klucz])) {
    die("Nie ma takiego pliku.");
}

echo nl2br(htmlspecialchars(file_get_contents(__DIR__ . "/dane/" . $dozwolone[$klucz])));
```

Do ścieżki trafia **tylko** wartość z naszej tablicy – podobnie jak przy sortowaniu w materiale o PDO.

#### [OK] Obrona 2 – `basename()` + sprawdzenie rozszerzenia

Gdy listy nie da się przygotować z góry:

```php
$nazwa = basename($_GET['plik'] ?? '');   // "../../db.php" → "db.php"

if (pathinfo($nazwa, PATHINFO_EXTENSION) !== 'txt') {
    die("Niedozwolony typ pliku.");
}

$sciezka = __DIR__ . "/dane/" . $nazwa;
if (!is_file($sciezka)) {
    die("Nie ma takiego pliku.");
}

echo nl2br(htmlspecialchars(file_get_contents($sciezka)));
```

`basename()` usuwa wszystkie katalogi z podanej ścieżki – zostaje sama nazwa pliku, więc `../` przestaje działać.

### Zagrożenie 2: Pliki z danymi dostępne przez przeglądarkę

Jeśli `dane/wpisy.csv` leży w `htdocs`, każdy może go pobrać, wpisując `http://localhost/projekt/dane/wpisy.csv`. Rozwiązania:

1. **Najlepiej:** trzymać pliki z danymi **poza** katalogiem publicznym (np. `C:\xampp\dane_projektu\`) i odwoływać się do nich np. przez `__DIR__ . "/../dane_projektu/wpisy.csv"`.
2. Na Apache (XAMPP): umieścić w katalogu `dane/` plik `.htaccess` blokujący dostęp:

```apache
Require all denied
```

### Zagrożenie 3: Zapis plików wykonywalnych

```php
// [X] NIGDY tak nie rób
file_put_contents(__DIR__ . "/" . $_POST['nazwa'], $_POST['tresc']);
```

Atakujący wyśle `nazwa=hack.php` i `tresc=<?php ... ?>` – a potem otworzy `hack.php` w przeglądarce i **wykona dowolny kod na serwerze**. Nazwy plików do zapisu ustala **programista**, nie użytkownik. Tak samo nigdy nie używamy `include` / `require` z nazwą pliku pochodzącą od użytkownika.

### Zagrożenie 4: XSS przy wyświetlaniu

Zawartość pliku zapisana przez użytkownika jest tak samo niebezpieczna jak dane z bazy. **Każde** wyświetlenie → `htmlspecialchars()`.

### Zagrożenie 5: Zapchanie dysku

Bez limitu długości ktoś może wysłać skryptem tysiące ogromnych wpisów. Ograniczaj długość danych (walidacja w PHP, nie tylko `maxlength` w HTML – ten atrybut łatwo obejść).

> **Dwa kierunki, dwa zabezpieczenia (jak przy bazie):**
> - **Do pliku** (wejście) → walidacja danych, nazwy plików ustala programista lub biała lista.
> - **Z pliku na stronę** (wyjście) → `htmlspecialchars()`.

---

## 13. Najczęstsze błędy

| Błąd | Dlaczego źle | Poprawnie |
|---|---|---|
| `fopen($p, "w")` przy dopisywaniu | Tryb `w` kasuje całą zawartość pliku | `"a"` albo `FILE_APPEND` |
| Brak sprawdzenia, czy plik istnieje | Ostrzeżenie i `false` zamiast danych | `file_exists()` / `is_file()` przed odczytem |
| Brak `fclose()` | Plik zostaje otwarty, blokada może nie zostać zwolniona | Zawsze zamykaj uchwyt |
| `if (!file_put_contents(...))` | Zapis 0 bajtów też jest „fałszem” | `=== false` |
| Ścieżki względne bez `__DIR__` | Działają z jednego pliku, z innego już nie | `__DIR__ . "/dane/plik.txt"` |
| Własny separator `\|` w danych od użytkownika | Enter lub separator w treści psuje plik | `fputcsv()` / `json_encode()` |
| `while (!feof($f))` | Dodatkowy obieg z `false` | `while (($l = fgets($f)) !== false)` |
| Nazwa pliku z `$_GET` / `$_POST` bez kontroli | Path Traversal, nadpisanie plików, wykonanie kodu | Biała lista lub `basename()` + kontrola rozszerzenia |
| Wyświetlanie zawartości bez `htmlspecialchars()` | XSS | `nl2br(htmlspecialchars($tresc))` |
| „Odczytaj → zmień → zapisz” bez blokady | Wyścig – gubienie danych przy wielu użytkownikach | `fopen("c+")` + `flock(LOCK_EX)` |
| `strlen()` do sprawdzania długości tekstu po polsku | Liczy bajty, nie znaki | `mb_strlen()` |

> ⚠️ **Najważniejsza pułapka:** nazwa lub ścieżka pliku pochodząca od użytkownika. Nawet jeśli wszystko inne jest zrobione dobrze, jedno `file_get_contents($_GET['plik'])` może ujawnić hasło do bazy danych.

---

## 14. Podsumowanie

- Pliki nadają się do logów, konfiguracji, eksportu i prostych danych. Do danych wyszukiwanych i współdzielonych – baza.
- Ścieżki budujemy od `__DIR__`.
- **Zapis:** `file_put_contents()` (nadpisanie), z `FILE_APPEND` (dopisanie), z `LOCK_EX` (blokada).
- **Odczyt:** `file_get_contents()` (cały tekst), `file()` (tablica linii), `fgets()` w pętli (duże pliki).
- `fopen()` → operacje → `fclose()`. Tryb decyduje o wszystkim: `r` czyta, `w` **czyści**, `a` dopisuje, `c+` – odczyt i zapis bez czyszczenia.
- Wielu użytkowników naraz → `flock()`.
- Dane strukturalne zapisujemy w **CSV** (`fputcsv` / `fgetcsv`) lub **JSON** (`json_encode` / `json_decode`).
- Bezpieczeństwo: biała lista nazw plików, `basename()`, pliki danych poza katalogiem publicznym, `htmlspecialchars()` przy wyświetlaniu.

---

## Pytania sprawdzające

1. Podaj dwa przykłady danych, które lepiej zapisać w pliku, i dwa, które lepiej zapisać w bazie danych. Uzasadnij.
2. Co zawiera stała `__DIR__` i dlaczego warto jej używać w ścieżkach?
3. Czym różni się wywołanie `file_put_contents($p, $d)` od `file_put_contents($p, $d, FILE_APPEND)`?
4. Jaka jest różnica między `file_get_contents()` a `file()`? Kiedy lepiej czytać plik funkcją `fgets()` w pętli?
5. Omów tryby `r`, `w`, `a` i `x` funkcji `fopen()`. Który z nich usuwa zawartość pliku?
6. Czym jest wyścig (*race condition*) przy zapisie do pliku? Jak mu zapobiec?
7. Czym różni się blokada `LOCK_SH` od `LOCK_EX`?
8. Dlaczego do zapisu rekordów lepiej użyć `fputcsv()` niż samodzielnie sklejać pola separatorem?
9. Co robi drugi argument `true` w `json_decode($json, true)`?
10. Na czym polega atak Path Traversal? Podaj dwa sposoby obrony.
11. Dlaczego nie wolno pozwolić użytkownikowi decydować o nazwie zapisywanego pliku?

---

## Zadania do wykonania

**Zadanie 1.** Napisz funkcję `zapiszLog()` z punktu 3. Wywołuj ją na stronie głównej przy każdym wejściu (zapisz datę i adres IP: `$_SERVER['REMOTE_ADDR']`). Utwórz stronę `logi.php`, która wyświetla 10 ostatnich wpisów, najnowsze na górze.

**Zadanie 2.** Utwórz plik `ustawienia.json` z tytułem strony i kolorem tła. Napisz stronę, która odczytuje te ustawienia i z nich korzysta, oraz formularz, który pozwala je zmienić i zapisuje plik ponownie.

**Zadanie 3.** Uruchom księgę gości z punktu 9. Sprawdź, co się stanie, gdy w treści wpisu wpiszesz `<b>test</b>`, średnik i kilka linii tekstu. Otwórz plik `wpisy.csv` w Notatniku i w Excelu.

**Zadanie 4.** Dodaj do strony z produktami (materiał o PDO) przycisk „Eksportuj do CSV”, który zapisuje wszystkie produkty z bazy do pliku `eksport.csv` (z wierszem nagłówków: nazwa, kategoria, cena).

**Zadanie 5.** Napisz stronę `pokaz.php?plik=...` wyświetlającą jeden z plików tekstowych w katalogu `dane/`. Najpierw w wersji niezabezpieczonej – **na własnym XAMPP** sprawdź, czy da się odczytać plik `db.php` przez `../`. Następnie zabezpiecz ją białą listą.

**Zadanie 6.** Napisz licznik odwiedzin w wersji „szkolnej” i w wersji z `flock()` (punkt 10). Wyjaśnij w komentarzu w kodzie, w jakiej sytuacji wersja „szkolna” poda błędny wynik.

---

## Materiały dodatkowe

- Funkcje systemu plików (PL): https://www.php.net/manual/pl/ref.filesystem.php
- `fopen()` – opis wszystkich trybów: https://www.php.net/manual/pl/function.fopen.php
- `file_put_contents()`: https://www.php.net/manual/pl/function.file-put-contents.php
- `flock()`: https://www.php.net/manual/pl/function.flock.php
- `fputcsv()` / `fgetcsv()`: https://www.php.net/manual/pl/function.fputcsv.php
- `json_encode()`: https://www.php.net/manual/pl/function.json-encode.php
- OWASP – Path Traversal: https://owasp.org/www-community/attacks/Path_Traversal

---

*Materiał przygotowany dla kwalifikacji INF.03 – Tworzenie i administrowanie stronami i aplikacjami internetowymi oraz bazami danych*

*Opracował: Bartosz Bryniarski*
