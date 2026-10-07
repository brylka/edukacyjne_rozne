# PDO i zapytania parametryzowane – dlaczego nie `mysqli_*`?
### Technik Programista | Kwalifikacja INF.03 | Programowanie aplikacji internetowych

---

## Plan materiału

1. Jak PHP łączy się z bazą danych – krótka historia
2. Problem: sklejanie zapytań SQL i **SQL Injection**
3. Dlaczego `mysqli_real_escape_string()` to nie jest rozwiązanie
4. Zapytania przygotowane (prepared statements) – idea
5. Dlaczego PDO, a nie przygotowane zapytania w `mysqli`?
6. PDO od podstaw – połączenie z bazą
7. Parametryzowanie zapytań – `?` i `:nazwa`
8. CRUD w PDO
9. Czego **nie da się** sparametryzować
10. Przed i po – przepisujemy kod z `mysqli` na PDO
11. Najczęstsze błędy

---

## 1. Jak PHP łączy się z bazą danych?

W PHP istniały trzy sposoby pracy z bazą MySQL:

| Rozszerzenie | Status | Uwagi |
|---|---|---|
| `mysql_*` | [X] **Usunięte** w PHP 7.0 | Stary kod z internetu – nie działa w nowym PHP |
| `mysqli_*` | [!] Działa | Tylko MySQL/MariaDB, w szkołach często używane w wersji „sklejanej” |
| **PDO** | [OK] Zalecane | Jeden interfejs dla wielu baz, wygodne parametry, wyjątki |

> **Ważne:** Sam fakt użycia `mysqli` nie oznacza od razu, że kod jest dziurawy. Problemem jest **sposób**, w jaki najczęściej się go używa – czyli wklejanie danych od użytkownika prosto do treści zapytania SQL.

---

## 2. Problem: sklejanie zapytań i SQL Injection

### Przykładowa tabela

```sql
CREATE TABLE uzytkownicy (
    id INT AUTO_INCREMENT PRIMARY KEY,
    login VARCHAR(50) NOT NULL,
    haslo VARCHAR(255) NOT NULL
);
```

### Typowy „szkolny” kod logowania

```php
<?php
$conn = mysqli_connect("localhost", "root", "", "sklep");

$login = $_POST['login'];
$haslo = $_POST['haslo'];

$sql = "SELECT * FROM uzytkownicy WHERE login = '$login' AND haslo = '$haslo'";
$wynik = mysqli_query($conn, $sql);

if (mysqli_num_rows($wynik) > 0) {
    echo "Zalogowano!";
} else {
    echo "Błędny login lub hasło";
}
```

Wygląda niewinnie. Kod działa. Ale co się stanie, gdy ktoś wpisze w pole **login**:

```
' OR '1'='1' -- 
```

(na końcu jest spacja po `--`). Zapytanie, które trafi do bazy, wygląda tak:

```sql
SELECT * FROM uzytkownicy WHERE login = '' OR '1'='1' -- ' AND haslo = ''
```

Rozbierzmy to:

| Fragment | Co robi |
|---|---|
| `login = ''` | Fałsz – nie ma pustego loginu |
| `OR '1'='1'` | **Zawsze prawda** |
| `-- ...` | Komentarz w SQL – reszta zapytania (z hasłem!) jest ignorowana |

Wynik: baza zwraca **wszystkich użytkowników**, `mysqli_num_rows()` > 0 → **zalogowano bez hasła**.

### Czym jest SQL Injection?

**SQL Injection** (wstrzyknięcie SQL) to atak, w którym dane wpisane przez użytkownika **zmieniają strukturę zapytania SQL**. Baza nie odróżnia, co napisał programista, a co dopisał użytkownik – dostaje jeden ciąg znaków i go wykonuje.

Skutki mogą być poważne:
- logowanie bez hasła,
- odczyt całej bazy (np. danych osobowych, haseł),
- modyfikacja lub usunięcie danych.

> ⚠️ SQL Injection od lat znajduje się na liście najgroźniejszych podatności aplikacji webowych (OWASP Top 10 – kategoria *Injection*). To nie jest teoria – to realne wycieki danych firm.

> ⚠️ Testuj podatności **wyłącznie** na własnych aplikacjach w pracowni (XAMPP). Atakowanie cudzych systemów jest przestępstwem (art. 267 Kodeksu karnego).

---

## 3. Dlaczego `mysqli_real_escape_string()` to nie rozwiązanie?

Często spotkasz „poprawkę”:

```php
$login = mysqli_real_escape_string($conn, $_POST['login']);
```

Funkcja dodaje `\` przed znakami specjalnymi (np. `'` → `\'`). Pomaga… ale tylko częściowo:

**Problem 1 – łatwo zapomnieć.** Wystarczy jedna zmienna w jednym pliku bez escape'owania i aplikacja jest dziurawa.

**Problem 2 – nie chroni liczb bez cudzysłowów.**

```php
$id = mysqli_real_escape_string($conn, $_GET['id']);
$sql = "SELECT * FROM produkty WHERE id = $id";
```

Adres `produkt.php?id=1 OR 1=1` daje zapytanie:

```sql
SELECT * FROM produkty WHERE id = 1 OR 1=1
```

W `1 OR 1=1` nie ma żadnego apostrofu do zabezpieczenia – funkcja nic nie zmienia, a atak działa.

**Problem 3 – dane i kod nadal są wymieszane.** To wciąż sklejanie stringów, tylko „ostrożniejsze”.

> **Zasada:** Nie próbujemy „czyścić” danych, żeby bezpiecznie wkleić je do SQL. **W ogóle ich nie wklejamy.** Do tego służą zapytania przygotowane.

---

## 4. Zapytania przygotowane – idea

**Prepared statement** (zapytanie przygotowane) działa w dwóch krokach:

```
Krok 1:  PHP → baza:  "SELECT * FROM uzytkownicy WHERE login = ?"
         Baza analizuje strukturę zapytania. Wie, że ? to MIEJSCE NA WARTOŚĆ.

Krok 2:  PHP → baza:  wartość dla ? = "' OR '1'='1' -- "
         Baza traktuje to wyłącznie jako tekst do porównania – nie jako kod SQL.
```

Złośliwy tekst zostaje po prostu porównany z kolumną `login`. Nikt nie ma takiego loginu → brak wyników → brak logowania.

| Sklejanie stringów | Zapytanie przygotowane |
|---|---|
| Kod i dane w jednym ciągu | Kod osobno, dane osobno |
| Dane mogą zmienić strukturę zapytania | Struktura ustalona przed podaniem danych |
| Bezpieczeństwo zależy od pamięci programisty | Bezpieczeństwo wynika z mechanizmu |

**Bindowanie** (wiązanie) to właśnie podpięcie konkretnej wartości pod znacznik (`?` lub `:nazwa`).

---

## 5. Dlaczego PDO, a nie przygotowane zapytania w `mysqli`?

Uczciwie: `mysqli` **też** obsługuje zapytania przygotowane. Zobacz jednak, jak to wygląda:

```php
// mysqli – zapytanie przygotowane
$stmt = mysqli_prepare($conn, "SELECT * FROM produkty WHERE kategoria = ? AND cena < ?");
mysqli_stmt_bind_param($stmt, "sd", $kategoria, $cena);   // "sd" = string, double
mysqli_stmt_execute($stmt);
$wynik = mysqli_stmt_get_result($stmt);
while ($wiersz = mysqli_fetch_assoc($wynik)) {
    echo $wiersz['nazwa'];
}
```

```php
// PDO – to samo
$stmt = $pdo->prepare("SELECT * FROM produkty WHERE kategoria = :kategoria AND cena < :cena");
$stmt->execute(['kategoria' => $kategoria, 'cena' => $cena]);
foreach ($stmt as $wiersz) {
    echo $wiersz['nazwa'];
}
```

### Porównanie

| Cecha | `mysqli` | PDO |
|---|---|---|
| Obsługiwane bazy | Tylko MySQL / MariaDB | MySQL, PostgreSQL, SQLite, SQL Server i inne |
| Parametry nazwane (`:login`) | [X] Tylko `?` | [OK] `?` oraz `:nazwa` |
| Przekazanie wartości | `bind_param("ssi", ...)` – trzeba pilnować typów i kolejności | `execute([...])` – zwykła tablica |
| Liczba funkcji do zapamiętania | Dużo (`mysqli_stmt_*`) | Mało, spójny styl obiektowy |
| Obsługa błędów | Wyjątki od PHP 8.1 | Wyjątki (domyślnie od PHP 8.0) |
| Zmiana bazy w przyszłości | Przepisanie całego kodu | Zwykle zmiana tylko połączenia (DSN) |

### Najważniejsze argumenty za PDO

1. **Parametry nazwane** – `:login` jest czytelniejsze niż piąty znak zapytania w kolejce.
2. **Jedno API do wielu baz** – nauczysz się raz, użyjesz z MySQL w szkole i z PostgreSQL czy SQLite w pracy.
3. **Mniej kodu = mniej błędów** – brak ciągów typu `"ssdi"`, brak przekazywania przez referencję.
4. **Standard w branży** – frameworki PHP (Laravel, Symfony) pod spodem korzystają z PDO.

---

## 6. PDO od podstaw – połączenie z bazą

Utwórz plik `db.php`, który będziesz dołączać w innych plikach:

```php
<?php
// db.php – połączenie z bazą przez PDO

$host  = "localhost";
$baza  = "sklep";
$user  = "root";
$haslo = "";            // w XAMPP domyślnie puste

$dsn = "mysql:host=$host;dbname=$baza;charset=utf8mb4";

$opcje = [
    PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION, // błędy jako wyjątki
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,       // wyniki jako tablice asocjacyjne
    PDO::ATTR_EMULATE_PREPARES   => false,                  // prawdziwe zapytania przygotowane
];

try {
    $pdo = new PDO($dsn, $user, $haslo, $opcje);
} catch (PDOException $e) {
    // Na produkcji NIE pokazujemy szczegółów błędu użytkownikowi
    die("Błąd połączenia z bazą danych.");
}
```

### Co tu się dzieje?

| Element | Znaczenie |
|---|---|
| **DSN** (*Data Source Name*) | Opis połączenia: typ bazy, host, nazwa bazy, kodowanie |
| `charset=utf8mb4` | Poprawne polskie znaki (i emoji) |
| `ERRMODE_EXCEPTION` | Błąd SQL rzuca wyjątek – nie przejdzie „po cichu” |
| `FETCH_ASSOC` | `$wiersz['nazwa']` zamiast mieszanki indeksów liczbowych i nazw |
| `EMULATE_PREPARES => false` | Zapytanie i dane trafiają do bazy osobno (patrz punkt 4) |

W innych plikach:

```php
<?php
require "db.php";
// tu już masz gotową zmienną $pdo
```

---

## 7. Parametryzowanie zapytań

### Sposób 1 – znaki zapytania `?`

```php
$stmt = $pdo->prepare("SELECT * FROM produkty WHERE kategoria = ? AND cena < ?");
$stmt->execute([$kategoria, $cena]);   // kolejność ma znaczenie!
```

### Sposób 2 – parametry nazwane `:nazwa` (zalecane)

```php
$stmt = $pdo->prepare("SELECT * FROM produkty WHERE kategoria = :kat AND cena < :cena");
$stmt->execute([
    'kat'  => $kategoria,
    'cena' => $cena,
]);
```

Kolejność w tablicy nie ma znaczenia – liczy się nazwa.

> ⚠️ W jednym zapytaniu **nie mieszamy** `?` i `:nazwa`.

### Sposób 3 – `bindValue()` z typem

Przydaje się, gdy chcesz jawnie określić typ, np. dla `LIMIT`:

```php
$stmt = $pdo->prepare("SELECT * FROM produkty ORDER BY cena LIMIT :ile");
$stmt->bindValue(':ile', (int)$ile, PDO::PARAM_INT);
$stmt->execute();
```

| Stała | Typ |
|---|---|
| `PDO::PARAM_STR` | tekst (domyślny) |
| `PDO::PARAM_INT` | liczba całkowita |
| `PDO::PARAM_BOOL` | wartość logiczna |
| `PDO::PARAM_NULL` | `NULL` |

> Istnieje też `bindParam()` – wiąże **zmienną** (przez referencję), a jej wartość jest odczytywana dopiero przy `execute()`. Na start wystarczy `execute([...])` albo `bindValue()`.

### Pobieranie wyników

| Metoda | Zwraca |
|---|---|
| `$stmt->fetch()` | Jeden wiersz (lub `false`, gdy brak) |
| `$stmt->fetchAll()` | Wszystkie wiersze jako tablicę |
| `$stmt->fetchColumn()` | Wartość jednej kolumny (np. wynik `COUNT(*)`) |
| `$stmt->rowCount()` | Liczba zmienionych wierszy (dla `INSERT/UPDATE/DELETE`) |
| `$pdo->lastInsertId()` | ID ostatnio dodanego rekordu |

---

## 8. CRUD w PDO

Przykładowa tabela:

```sql
CREATE TABLE produkty (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nazwa VARCHAR(100) NOT NULL,
    kategoria VARCHAR(50) NOT NULL,
    cena DECIMAL(10,2) NOT NULL
);
```

### CREATE – dodawanie

```php
$stmt = $pdo->prepare(
    "INSERT INTO produkty (nazwa, kategoria, cena) VALUES (:nazwa, :kategoria, :cena)"
);
$stmt->execute([
    'nazwa'     => $_POST['nazwa'],
    'kategoria' => $_POST['kategoria'],
    'cena'      => $_POST['cena'],
]);

echo "Dodano produkt o ID: " . $pdo->lastInsertId();
```

### READ – odczyt wielu rekordów

```php
$stmt = $pdo->prepare("SELECT * FROM produkty WHERE kategoria = :kat ORDER BY nazwa");
$stmt->execute(['kat' => $_GET['kategoria']]);
$produkty = $stmt->fetchAll();

foreach ($produkty as $p) {
    echo htmlspecialchars($p['nazwa']) . " – " . $p['cena'] . " zł<br>";
}
```

### READ – odczyt jednego rekordu

```php
$stmt = $pdo->prepare("SELECT * FROM produkty WHERE id = :id");
$stmt->execute(['id' => $_GET['id']]);
$produkt = $stmt->fetch();

if (!$produkt) {
    die("Nie ma takiego produktu.");
}
echo htmlspecialchars($produkt['nazwa']);
```

> Tu atak `?id=1 OR 1=1` z punktu 3 **nie działa** – cały tekst `1 OR 1=1` jest traktowany jako wartość, a nie jako kod SQL.

### UPDATE – modyfikacja

```php
$stmt = $pdo->prepare("UPDATE produkty SET cena = :cena WHERE id = :id");
$stmt->execute([
    'cena' => $_POST['cena'],
    'id'   => $_POST['id'],
]);

echo "Zmieniono rekordów: " . $stmt->rowCount();
```

### DELETE – usuwanie

```php
$stmt = $pdo->prepare("DELETE FROM produkty WHERE id = :id");
$stmt->execute(['id' => $_POST['id']]);

echo "Usunięto rekordów: " . $stmt->rowCount();
```

### A zapytanie bez żadnych danych od użytkownika?

Można użyć `query()` – nie ma czego parametryzować:

```php
$produkty = $pdo->query("SELECT * FROM produkty ORDER BY nazwa")->fetchAll();
```

> ⚠️ Jeśli w zapytaniu pojawia się **jakakolwiek** zmienna – używaj `prepare()` + `execute()`. Bez wyjątków.

---

## 9. Czego nie da się sparametryzować?

Parametry zastępują **wartości**, nie fragmenty składni SQL. Nie zadziała:

```php
// [X] NIE DZIAŁA – nazwy tabel i kolumn nie mogą być parametrami
$stmt = $pdo->prepare("SELECT * FROM :tabela ORDER BY :kolumna");
```

Gdy użytkownik wybiera np. sortowanie, stosujemy **białą listę** (whitelist) – dopuszczamy tylko znane wartości:

```php
$dozwolone = ['nazwa', 'cena', 'kategoria'];
$sortuj = $_GET['sort'] ?? 'nazwa';

if (!in_array($sortuj, $dozwolone, true)) {
    $sortuj = 'nazwa';   // wartość domyślna
}

$kierunek = (($_GET['kier'] ?? '') === 'desc') ? 'DESC' : 'ASC';

$stmt = $pdo->query("SELECT * FROM produkty ORDER BY $sortuj $kierunek");
```

Tutaj sklejanie jest bezpieczne, bo do zapytania może trafić **tylko** wartość z naszej listy.

---

## 10. Przed i po – przepisujemy logowanie

### [X] Przed (mysqli, sklejanie, hasło jawnym tekstem)

```php
<?php
$conn = mysqli_connect("localhost", "root", "", "sklep");

$login = $_POST['login'];
$haslo = $_POST['haslo'];

$sql = "SELECT * FROM uzytkownicy WHERE login = '$login' AND haslo = '$haslo'";
$wynik = mysqli_query($conn, $sql);

if (mysqli_num_rows($wynik) > 0) {
    echo "Zalogowano!";
}
```

### [OK] Po (PDO, parametry, hash hasła)

```php
<?php
require "db.php";

$stmt = $pdo->prepare("SELECT id, login, haslo FROM uzytkownicy WHERE login = :login");
$stmt->execute(['login' => $_POST['login']]);
$user = $stmt->fetch();

if ($user && password_verify($_POST['haslo'], $user['haslo'])) {
    echo "Zalogowano!";
} else {
    echo "Błędny login lub hasło";
}
```

Co się zmieniło:
- dane od użytkownika **nie trafiają do treści SQL** – SQL Injection nie zadziała,
- hasło porównujemy przez `password_verify()`, bo w bazie przechowujemy **hash**, a nie hasło.

Przy rejestracji hasło zapisujemy tak:

```php
$hash = password_hash($_POST['haslo'], PASSWORD_DEFAULT);

$stmt = $pdo->prepare("INSERT INTO uzytkownicy (login, haslo) VALUES (:login, :haslo)");
$stmt->execute(['login' => $_POST['login'], 'haslo' => $hash]);
```

> **Dwa różne zagrożenia, dwa różne zabezpieczenia:**
> - **Do bazy** (wejście) → zapytania parametryzowane chronią przed **SQL Injection**.
> - **Na stronę** (wyjście) → `htmlspecialchars()` chroni przed **XSS** (wstrzyknięciem JavaScriptu).
>
> PDO **nie** chroni przed XSS – dane z bazy przed wyświetleniem nadal trzeba zabezpieczyć.

---

## 11. Najczęstsze błędy

| Błąd | Dlaczego źle | Poprawnie |
|---|---|---|
| `$pdo->prepare("... WHERE id = $id")` | `prepare()` niczego nie chroni, jeśli zmienna jest wklejona w tekst! | `"... WHERE id = :id"` + `execute(['id' => $id])` |
| `'... WHERE login = ':login''` | Parametru nie bierzemy w cudzysłowy | `WHERE login = :login` |
| Wyświetlanie `$e->getMessage()` użytkownikowi | Ujawnia strukturę bazy, ścieżki, czasem dane | Komunikat ogólny, szczegóły do logu |
| Brak `charset=utf8mb4` w DSN | „Krzaczki” zamiast polskich znaków | Zawsze podawaj kodowanie w DSN |
| `query()` z danymi od użytkownika | To znowu sklejanie stringów | `prepare()` + `execute()` |
| Wyświetlanie danych z bazy bez `htmlspecialchars()` | Podatność na XSS | `echo htmlspecialchars($w['nazwa']);` |

> ⚠️ **Najważniejsza pułapka:** samo użycie `prepare()` nie czyni kodu bezpiecznym. Bezpieczny jest dopiero kod, w którym **żadna zmienna nie jest wklejona w treść zapytania**.

---

## 12. Podsumowanie

- Rozszerzenie `mysql_*` zostało usunięte z PHP – nie używamy go w ogóle.
- Sklejanie zapytań z danymi od użytkownika prowadzi do **SQL Injection** – niezależnie od tego, czy używasz `mysqli`, czy PDO.
- `mysqli_real_escape_string()` to półśrodek – łatwo zapomnieć, nie chroni liczb.
- **Zapytania przygotowane** oddzielają kod SQL od danych – to właściwa ochrona.
- **PDO** daje parametry nazwane, prosty `execute([...])`, wyjątki i obsługę wielu baz.
- Parametrami można zastąpić tylko **wartości** – nazwy kolumn i tabel sprawdzamy białą listą.
- Hasła: `password_hash()` / `password_verify()`. Wyświetlanie: `htmlspecialchars()`.

---

## Pytania sprawdzające

1. Czym jest SQL Injection? Podaj przykład danych, które mogą zmienić sens zapytania.
2. Dlaczego zapytanie `"SELECT * FROM produkty WHERE id = $id"` jest niebezpieczne nawet po użyciu `mysqli_real_escape_string()`?
3. Wyjaśnij, na czym polega zapytanie przygotowane (prepared statement). Co oznacza „bindowanie” parametru?
4. Wymień co najmniej trzy zalety PDO w porównaniu z `mysqli`.
5. Co oznacza skrót DSN i jakie informacje zawiera?
6. Jaka jest różnica między parametrem `?` a `:nazwa`?
7. Do czego służą metody `fetch()`, `fetchAll()`, `rowCount()` i `lastInsertId()`?
8. Dlaczego nie można przekazać nazwy kolumny jako parametru? Jak bezpiecznie obsłużyć sortowanie wybierane przez użytkownika?
9. Czy kod `$pdo->prepare("SELECT * FROM uzytkownicy WHERE login = '$login'")` jest bezpieczny? Uzasadnij.
10. Przed jakim atakiem chroni PDO, a przed jakim `htmlspecialchars()`?

---

## Zadania do wykonania

**Zadanie 1.** Utwórz bazę `sklep` z tabelą `produkty` (punkt 8) i plik `db.php` z połączeniem PDO. Sprawdź, czy połączenie działa.

**Zadanie 2.** Napisz stronę `produkty.php`, która wyświetla listę produktów z kategorii podanej w adresie (`produkty.php?kategoria=AGD`). Użyj parametru nazwanego.

**Zadanie 3.** Dodaj formularz dodawania produktu. Po dodaniu wyświetl komunikat z ID nowego rekordu (`lastInsertId()`).

**Zadanie 4.** Na swoim komputerze (XAMPP) uruchom kod logowania z punktu 2 i sprawdź, czy da się zalogować bez hasła. Następnie przepisz go na PDO (punkt 10) i sprawdź ponownie.

**Zadanie 5.** Dodaj do listy produktów sortowanie po nazwie lub cenie (`?sort=cena&kier=desc`) z wykorzystaniem białej listy.

---

## Materiały dodatkowe

- Dokumentacja PDO (PL): https://www.php.net/manual/pl/book.pdo.php
- Zapytania przygotowane w PDO: https://www.php.net/manual/pl/pdo.prepared-statements.php
- `password_hash()`: https://www.php.net/manual/pl/function.password-hash.php
- OWASP – SQL Injection: https://owasp.org/www-community/attacks/SQL_Injection
- OWASP – SQL Injection Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html

---

*Materiał przygotowany dla kwalifikacji INF.03 – Tworzenie i administrowanie stronami i aplikacjami internetowymi oraz bazami danych*

*Opracował: Bartosz Bryniarski*