# Debian – serwer WWW: Apache i nginx (tryb tekstowy)

**Dla kogo:** uczniowie technikum (kwalifikacja INF.02)
**Założenie:** Debian (najnowsza wersja) w VirtualBox, bez środowiska graficznego. Znasz już podstawy z materiału *„Debian – pierwsze kroki w trybie tekstowym”* (sudo, nano, uprawnienia, systemctl, sieć).
**Cel:** postawić serwer WWW, skonfigurować hosty wirtualne, katalogi domowe użytkowników, ochronę hasłem, HTTPS i PHP – najpierw w **Apache** (najczęściej na egzaminie), potem **to samo w nginx**.

> 💡 **Jak korzystać z materiału:**
> - **Ścieżka A – Apache** – podstawowa, przerób ją w całości.
> - **Ścieżka B – nginx** – robi **dokładnie te same rzeczy**, ale innym serwerem. Nginx **zastępuje** Apache (nie działają razem na porcie 80). Najlepiej przerabiaj ją na osobnej maszynie wirtualnej albo najpierw wyłącz Apache (opisane w Części 13).
> - Części 1–2 i 22–23 są wspólne dla obu serwerów.

---

## Spis treści

**Wspólne**
1. [CZĘŚĆ 1: Jak działa serwer WWW](#część-1-jak-działa-serwer-www)
2. [CZĘŚĆ 2: Przygotowanie – sieć, klient, narzędzia](#część-2-przygotowanie--sieć-klient-narzędzia)

**Ścieżka A – Apache**

3. [CZĘŚĆ 3: Instalacja Apache](#część-3-instalacja-apache)
4. [CZĘŚĆ 4: Pliki konfiguracyjne Apache i polecenia a2*](#część-4-pliki-konfiguracyjne-apache-i-polecenia-a2)
5. [CZĘŚĆ 5: Własna strona i uprawnienia](#część-5-własna-strona-i-uprawnienia)
6. [CZĘŚĆ 6: Hosty wirtualne (VirtualHost)](#część-6-hosty-wirtualne-virtualhost)
7. [CZĘŚĆ 7: Katalogi domowe użytkowników (userdir)](#część-7-katalogi-domowe-użytkowników-userdir)
8. [CZĘŚĆ 8: Ochrona hasłem i ograniczenie dostępu](#część-8-ochrona-hasłem-i-ograniczenie-dostępu)
9. [CZĘŚĆ 9: Listowanie katalogów, strony błędów, przekierowania](#część-9-listowanie-katalogów-strony-błędów-przekierowania)
10. [CZĘŚĆ 10: HTTPS w Apache](#część-10-https-w-apache)
11. [CZĘŚĆ 11: PHP w Apache](#część-11-php-w-apache)
12. [CZĘŚĆ 12: Logi i diagnostyka Apache](#część-12-logi-i-diagnostyka-apache)

**Ścieżka B – nginx**

13. [CZĘŚĆ 13: Zamiana Apache na nginx](#część-13-zamiana-apache-na-nginx)
14. [CZĘŚĆ 14: Pliki konfiguracyjne nginx](#część-14-pliki-konfiguracyjne-nginx)
15. [CZĘŚĆ 15: Hosty wirtualne w nginx (server blocks)](#część-15-hosty-wirtualne-w-nginx-server-blocks)
16. [CZĘŚĆ 16: Katalogi domowe użytkowników w nginx](#część-16-katalogi-domowe-użytkowników-w-nginx)
17. [CZĘŚĆ 17: Ochrona hasłem i ograniczenie dostępu w nginx](#część-17-ochrona-hasłem-i-ograniczenie-dostępu-w-nginx)
18. [CZĘŚĆ 18: Listowanie, strony błędów, przekierowania w nginx](#część-18-listowanie-strony-błędów-przekierowania-w-nginx)
19. [CZĘŚĆ 19: HTTPS w nginx](#część-19-https-w-nginx)
20. [CZĘŚĆ 20: PHP w nginx (PHP-FPM)](#część-20-php-w-nginx-php-fpm)
21. [CZĘŚĆ 21: Logi i diagnostyka nginx](#część-21-logi-i-diagnostyka-nginx)

**Wspólne**

22. [CZĘŚĆ 22: Inne serwery WWW](#część-22-inne-serwery-www)
23. [CZĘŚĆ 23: Typowe błędy i jak je naprawić](#część-23-typowe-błędy-i-jak-je-naprawić)
24. [Słownik: Apache ↔ nginx](#słownik-apache--nginx)
25. [Ściąga – najważniejsze polecenia](#ściąga--najważniejsze-polecenia)
26. [Zadania do samodzielnego wykonania](#zadania-do-samodzielnego-wykonania)
27. [Rozwiązania](#rozwiązania)

---

## CZĘŚĆ 1: Jak działa serwer WWW

Serwer WWW to program, który **czeka na żądania** przeglądarki i **odsyła pliki** (strony HTML, obrazki, wyniki skryptów PHP).

```
Przeglądarka (klient)                     Serwer WWW (Debian)
  http://www.firma.local/index.html  ──►  Apache / nginx
                                          czyta /var/www/firma/index.html
                                     ◄──  odsyła stronę (kod 200 OK)
```

### Porty

| Protokół | Port | Uwagi |
|----------|------|-------|
| HTTP | **80** | bez szyfrowania |
| HTTPS | **443** | szyfrowane (SSL/TLS, certyfikat) |
| HTTP alternatywny | 8080, 8000, 8081… | często używane do drugiej strony / testów |

> ⚠️ **Uwaga:** Na jednym porcie może nasłuchiwać **tylko jeden program**. Dlatego Apache i nginx nie mogą jednocześnie działać na porcie 80.

### Kody odpowiedzi HTTP (warto znać!)

| Kod | Znaczenie | Typowa przyczyna |
|-----|-----------|------------------|
| **200** | OK | wszystko działa |
| **301 / 302** | przekierowanie | np. z HTTP na HTTPS |
| **401** | wymagane logowanie | strona chroniona hasłem |
| **403** | Forbidden – brak dostępu | złe uprawnienia plików, brak `index.html`, blokada IP |
| **404** | Not Found | zła ścieżka lub nazwa pliku (wielkość liter!) |
| **500** | błąd serwera | błąd w `.htaccess` lub skrypcie |
| **502** | Bad Gateway | nginx nie może połączyć się z PHP-FPM |

### Kto czyta pliki strony?

Serwer WWW **nie działa jako root** ani jako Ty. W Debianie działa jako użytkownik **`www-data`**. To on musi mieć prawo:
- **odczytu** plików (`r`),
- **wejścia** do każdego katalogu na ścieżce (`x`).

Większość błędów **403 Forbidden** bierze się właśnie stąd.

---

## CZĘŚĆ 2: Przygotowanie – sieć, klient, narzędzia

### Sieć w VirtualBox

Żeby otworzyć stronę z innego komputera (np. z Windowsa-hosta albo z drugiej maszyny wirtualnej), karta sieciowa VM powinna być ustawiona na:

| Tryb karty | Kiedy |
|------------|-------|
| **Mostkowana karta sieciowa** (Bridged) | serwer dostaje adres z sieci szkolnej – widać go z innych komputerów |
| **Karta sieci izolowanej** (Host-only) | serwer widoczny tylko z Twojego komputera-hosta |
| **Sieć wewnętrzna** | kilka VM (np. serwer Debian + klient Windows) w jednej sieci |
| NAT | serwer ma internet, ale **nie da się** do niego wejść z zewnątrz (bez przekierowania portów) |

Serwer powinien mieć **stały adres IP** (patrz poprzedni materiał, Część 12). W przykładach używamy:

```
serwer: 192.168.1.100/24
```

### Narzędzia do testowania (na serwerze, w trybie tekstowym)

Nie masz przeglądarki graficznej – ale masz `curl` i przeglądarki tekstowe.

```bash
sudo apt install -y curl w3m
```

```bash
curl http://localhost                       # pobierz stronę (wyświetli kod HTML)
curl -I http://localhost                    # tylko nagłówki – widać kod odpowiedzi (200, 404...)
curl http://192.168.1.100:8080              # inny port
curl -u jan:haslo http://localhost/admin/   # z logowaniem (Basic Auth)
curl -k https://localhost                   # HTTPS z certyfikatem self-signed (-k = nie sprawdzaj)
curl -L http://localhost                    # podążaj za przekierowaniami
w3m http://localhost                        # przeglądarka tekstowa (q = wyjście)
```

> 💡 **Wskazówka:** `curl -I` to Twój najlepszy przyjaciel. Pierwsza linia odpowiedzi (`HTTP/1.1 200 OK`, `HTTP/1.1 403 Forbidden`…) od razu mówi, co jest nie tak.

### Nazwy domenowe bez serwera DNS – plik hosts

Hosty wirtualne działają po **nazwie** (np. `www.firma.local`). Jeśli w sieci nie ma serwera DNS, wpisujesz nazwy ręcznie do pliku **hosts** na kliencie.

**Na serwerze Debian** (żeby testować `curl`-em lokalnie):

```bash
sudo nano /etc/hosts
```
```
127.0.0.1       localhost
127.0.1.1       debian
192.168.1.100   www.firma.local firma.local
192.168.1.100   www.szkola.local
```

**Na kliencie Windows:** uruchom Notatnik **jako administrator** i otwórz:

```
C:\Windows\System32\drivers\etc\hosts
```
Dopisz na końcu:
```
192.168.1.100   www.firma.local firma.local
192.168.1.100   www.szkola.local
```

> 💡 **Wskazówka:** Na egzaminie DNS bywa osobnym zadaniem (bind9 albo serwer DNS w Windows Server). Jeśli DNS jest skonfigurowany – plik hosts nie jest potrzebny.

> ⚠️ **Uwaga:** Domena `.local` w niektórych sieciach koliduje z mDNS. Jeśli coś nie działa, możesz użyć np. `.lan` albo `.test`. Na egzaminie używaj **dokładnie tych nazw, które są w treści zadania**.

---

# ŚCIEŻKA A – APACHE

---

## CZĘŚĆ 3: Instalacja Apache

```bash
sudo apt update
sudo apt install -y apache2
```

Apache uruchamia się **od razu po instalacji** i jest włączony przy starcie systemu.

```bash
sudo systemctl status apache2           # powinno być: active (running)
systemctl is-enabled apache2            # enabled
sudo ss -tulpn | grep :80               # nasłuchuje na porcie 80
apache2 -v                              # wersja Apache
curl -I http://localhost                # HTTP/1.1 200 OK
```

Teraz z klienta wejdź na `http://192.168.1.100` – zobaczysz stronę **„Apache2 Debian Default Page – It works!”**.

Ta strona to plik:

```
/var/www/html/index.html
```

### Zarządzanie usługą

```bash
sudo systemctl start apache2
sudo systemctl stop apache2
sudo systemctl restart apache2          # pełny restart
sudo systemctl reload apache2           # przeładuj konfigurację bez rozłączania klientów
sudo systemctl enable apache2           # start przy uruchomieniu systemu
```

> 💡 **Wskazówka:** Po zmianach w `sites-*` wystarczy `reload`. Po włączeniu/wyłączeniu **modułu** (`a2enmod`) – zrób `restart`.

---

## CZĘŚĆ 4: Pliki konfiguracyjne Apache i polecenia a2*

Cała konfiguracja jest w `/etc/apache2/`:

```
/etc/apache2/
├── apache2.conf          # główny plik konfiguracyjny (rzadko edytujesz)
├── ports.conf            # na jakich portach nasłuchuje (Listen 80, Listen 443)
├── envvars               # zmienne (użytkownik www-data, katalog logów)
├── sites-available/      # WSZYSTKIE zdefiniowane strony (vhosty)
│   ├── 000-default.conf  # domyślna strona HTTP
│   └── default-ssl.conf  # domyślna strona HTTPS
├── sites-enabled/        # strony WŁĄCZONE (dowiązania do sites-available)
├── mods-available/       # wszystkie dostępne moduły
├── mods-enabled/         # moduły włączone
├── conf-available/       # dodatkowe fragmenty konfiguracji
└── conf-enabled/         # włączone fragmenty
```

### Zasada „available / enabled”

Plik w `*-available` **nic nie robi**, dopóki nie zostanie „włączony”, czyli dowiązany do `*-enabled`. Nie robisz tego ręcznie – służą do tego polecenia:

| Polecenie | Co robi |
|-----------|---------|
| `a2ensite firma` | włącz stronę `sites-available/firma.conf` |
| `a2dissite 000-default` | wyłącz stronę |
| `a2enmod userdir` | włącz moduł |
| `a2dismod userdir` | wyłącz moduł |
| `a2enconf nazwa` | włącz fragment konfiguracji |
| `a2disconf nazwa` | wyłącz fragment konfiguracji |
| `apache2ctl configtest` | **sprawdź składnię konfiguracji** (`Syntax OK`) |
| `apache2ctl -S` | pokaż wszystkie vhosty i porty – świetne do diagnozy |
| `apache2ctl -M` | lista załadowanych modułów |

```bash
ls -l /etc/apache2/sites-enabled/       # widać strzałki -> ../sites-available/...
```

> ⚠️ **Uwaga:** Pliki w `sites-available` **muszą mieć rozszerzenie `.conf`**, inaczej `a2ensite` ich nie znajdzie. W poleceniu `a2ensite` rozszerzenie możesz pominąć.

> 💡 **Wskazówka:** Złota zasada – **po każdej zmianie**:
> ```bash
> sudo apache2ctl configtest && sudo systemctl reload apache2
> ```
> `&&` oznacza: wykonaj `reload` tylko wtedy, gdy test przeszedł.

### Domyślny vhost – `000-default.conf`

```bash
cat /etc/apache2/sites-available/000-default.conf | grep -v "#"
```
```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

| Dyrektywa | Znaczenie |
|-----------|-----------|
| `<VirtualHost *:80>` | obsługuj żądania na **każdym adresie IP** (`*`), port **80** |
| `ServerAdmin` | e-mail administratora (pokazywany na stronach błędów) |
| `DocumentRoot` | **katalog z plikami strony** |
| `ErrorLog` | gdzie zapisywać błędy |
| `CustomLog` | gdzie zapisywać odwiedziny (access log) |

### Ostrzeżenie AH00558

Przy restarcie możesz zobaczyć:

```
AH00558: apache2: Could not reliably determine the server's fully qualified domain name...
```

To **tylko ostrzeżenie** – serwer działa. Żeby się go pozbyć:

```bash
echo "ServerName localhost" | sudo tee /etc/apache2/conf-available/servername.conf
sudo a2enconf servername
sudo systemctl reload apache2
```

---

## CZĘŚĆ 5: Własna strona i uprawnienia

### Podmiana strony domyślnej

```bash
sudo cp /var/www/html/index.html /var/www/html/index.html.bak   # kopia!
echo "<h1>Serwer Jana Kowalskiego</h1>" | sudo tee /var/www/html/index.html
curl http://localhost
```

> 💡 **Wskazówka:** `sudo echo "..." > plik` **nie zadziała** (przekierowanie robi się bez uprawnień roota). Używaj `| sudo tee plik` albo `sudo nano plik`.

### Prosta strona HTML w nano

```bash
sudo nano /var/www/html/index.html
```
```html
<!DOCTYPE html>
<html lang="pl">
<head>
    <meta charset="UTF-8">
    <title>Moja strona</title>
</head>
<body>
    <h1>Witaj na serwerze Debian!</h1>
    <p>Serwer: Apache</p>
</body>
</html>
```

> ⚠️ **Uwaga:** Bez `<meta charset="UTF-8">` polskie znaki mogą wyświetlać się jako „krzaczki”.

### Plik startowy – DirectoryIndex

Gdy wpiszesz `http://serwer/` (bez nazwy pliku), Apache szuka po kolei:

```
index.html  index.cgi  index.pl  index.php  index.xhtml  index.htm
```

Kolejność możesz zmienić (np. w vhoście):

```apache
DirectoryIndex start.html index.html
```

Jeśli żadnego z tych plików nie ma – dostaniesz **listę plików** (jeśli włączone `Indexes`) albo **403 Forbidden**.

### Uprawnienia plików strony

Typowe, bezpieczne ustawienie:

| Co | Uprawnienia | Właściciel |
|----|-------------|-----------|
| katalogi | `755` (`rwxr-xr-x`) | root lub www-data lub webmaster |
| pliki | `644` (`rw-r--r--`) | j.w. |

```bash
sudo chown -R www-data:www-data /var/www/firma      # gdy strona ma móc zapisywać (np. upload)
sudo find /var/www/firma -type d -exec chmod 755 {} \;
sudo find /var/www/firma -type f -exec chmod 644 {} \;
```

> 💡 **Wskazówka:** Na egzaminie zwykle wystarcza, że pliki stworzone przez `sudo` mają właściciela `root` i prawa `644`/`755` – `www-data` może je czytać jako „pozostali”.

---

## CZĘŚĆ 6: Hosty wirtualne (VirtualHost)

**Host wirtualny** = kilka stron WWW na **jednym** serwerze. Apache rozpoznaje, którą stronę pokazać, na trzy sposoby:

| Rodzaj | Rozróżnia po | Przykład |
|--------|--------------|----------|
| **name-based** (najczęstszy!) | nazwie w adresie | `www.firma.local` i `www.szkola.local` → ten sam IP |
| **port-based** | porcie | `http://192.168.1.100:80` i `http://192.168.1.100:8080` |
| **IP-based** | adresie IP | serwer ma dwa IP: `.100` i `.101` |

### Name-based – krok po kroku

**1. Katalogi i strony**

```bash
sudo mkdir -p /var/www/firma /var/www/szkola
echo "<h1>Strona FIRMY</h1>"  | sudo tee /var/www/firma/index.html
echo "<h1>Strona SZKOLY</h1>" | sudo tee /var/www/szkola/index.html
```

**2. Plik konfiguracyjny vhosta**

```bash
sudo nano /etc/apache2/sites-available/firma.conf
```
```apache
<VirtualHost *:80>
    ServerName www.firma.local
    ServerAlias firma.local
    ServerAdmin admin@firma.local
    DocumentRoot /var/www/firma

    ErrorLog ${APACHE_LOG_DIR}/firma_error.log
    CustomLog ${APACHE_LOG_DIR}/firma_access.log combined
</VirtualHost>
```

| Dyrektywa | Znaczenie |
|-----------|-----------|
| `ServerName` | **główna** nazwa strony |
| `ServerAlias` | dodatkowe nazwy, pod którymi strona też działa (można kilka, po spacji) |

Drugi vhost najszybciej zrobić kopią:

```bash
sudo cp /etc/apache2/sites-available/firma.conf /etc/apache2/sites-available/szkola.conf
sudo nano /etc/apache2/sites-available/szkola.conf     # zamień firma -> szkola
```

> 💡 **Wskazówka:** Zamiast ręcznie zamieniać – `sed`:
> ```bash
> sudo sed -i 's/firma/szkola/g' /etc/apache2/sites-available/szkola.conf
> ```

**3. Włączenie, test, przeładowanie**

```bash
sudo a2ensite firma szkola
sudo apache2ctl configtest
sudo systemctl reload apache2
sudo apache2ctl -S                      # sprawdź listę vhostów
```

**4. Test** (wpisy w `/etc/hosts` – patrz Część 2)

```bash
curl http://www.firma.local             # Strona FIRMY
curl http://www.szkola.local            # Strona SZKOLY
curl http://192.168.1.100               # strona domyślna (000-default)
```

Bez wpisu w hosts można „oszukać” nagłówek `Host`:

```bash
curl -H "Host: www.firma.local" http://192.168.1.100
```

> ⚠️ **Uwaga – który vhost jest domyślny?** Gdy nazwa w adresie nie pasuje do żadnego `ServerName`/`ServerAlias` (np. wejście po samym IP), Apache pokazuje **pierwszy** vhost w kolejności alfabetycznej plików w `sites-enabled`. Dlatego domyślny plik nazywa się `000-default.conf` – „zero zero zero” zawsze jest pierwsze.
> Jeśli zadanie mówi „po wpisaniu adresu IP ma się wyświetlić strona firmy” – albo wyłącz `000-default` (`a2dissite 000-default`), albo nazwij plik np. `000-firma.conf`.

### Port-based – strona na porcie 8080

**1. Apache musi nasłuchiwać na nowym porcie**

```bash
sudo nano /etc/apache2/ports.conf
```
```apache
Listen 80
Listen 8080
```

**2. Vhost na porcie 8080**

```bash
sudo mkdir -p /var/www/intranet
echo "<h1>INTRANET</h1>" | sudo tee /var/www/intranet/index.html
sudo nano /etc/apache2/sites-available/intranet.conf
```
```apache
<VirtualHost *:8080>
    DocumentRoot /var/www/intranet
    ErrorLog ${APACHE_LOG_DIR}/intranet_error.log
    CustomLog ${APACHE_LOG_DIR}/intranet_access.log combined
</VirtualHost>
```

```bash
sudo a2ensite intranet
sudo apache2ctl configtest
sudo systemctl restart apache2          # nowy port → restart
sudo ss -tulpn | grep apache            # powinno być :80 i :8080
curl http://localhost:8080
```

> ⚠️ **Uwaga:** Sam `<VirtualHost *:8080>` **nie wystarczy** – bez `Listen 8080` w `ports.conf` Apache w ogóle nie otworzy tego portu.

### IP-based – dwa adresy IP

Serwer musi mieć dwa adresy (np. dwie karty sieciowe albo drugi adres na jednej karcie). Konfiguracja różni się tylko nagłówkiem:

```apache
<VirtualHost 192.168.1.100:80>
    DocumentRoot /var/www/firma
</VirtualHost>

<VirtualHost 192.168.1.101:80>
    DocumentRoot /var/www/szkola
</VirtualHost>
```

### DocumentRoot poza /var/www

Apache domyślnie pozwala serwować pliki **tylko** z `/var/www` i `/usr/share` (ustawione w `apache2.conf`). Jeśli strona ma być np. w `/srv/www/firma` albo `/strony/firma`, dodaj w vhoście:

```apache
<VirtualHost *:80>
    ServerName www.firma.local
    DocumentRoot /srv/www/firma

    <Directory /srv/www/firma>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
```

Bez bloku `<Directory>` z `Require all granted` dostaniesz **403 Forbidden**.

### Blok `<Directory>` – najważniejsze opcje

| Dyrektywa | Znaczenie |
|-----------|-----------|
| `Require all granted` | wszyscy mają dostęp |
| `Require all denied` | nikt nie ma dostępu |
| `Options Indexes` | pokazuj listę plików, gdy brak `index.html` |
| `Options -Indexes` | **nie** pokazuj listy plików |
| `Options FollowSymLinks` | podążaj za dowiązaniami symbolicznymi |
| `AllowOverride None` | ignoruj pliki `.htaccess` |
| `AllowOverride All` | pozwól `.htaccess` zmieniać wszystko |
| `AllowOverride AuthConfig` | pozwól `.htaccess` tylko na ustawienia logowania |

---

## CZĘŚĆ 7: Katalogi domowe użytkowników (userdir)

**Pewniak na INF.02!** Każdy użytkownik systemu ma własną stronę pod adresem:

```
http://192.168.1.100/~jan/
```

Pliki strony leżą w **jego** katalogu domowym:

```
/home/jan/public_html/
```

### Krok 1 – włącz moduł

```bash
sudo a2enmod userdir
sudo systemctl restart apache2
```

Konfiguracja modułu (zwykle nic nie zmieniasz):

```bash
cat /etc/apache2/mods-available/userdir.conf
```
```apache
<IfModule mod_userdir.c>
    UserDir public_html
    UserDir disabled root
    <Directory /home/*/public_html>
        AllowOverride FileInfo AuthConfig Limit Indexes
        Options MultiViews Indexes SymLinksIfOwnerMatch IncludesNoExec
        Require method GET POST OPTIONS
    </Directory>
</IfModule>
```

- `UserDir public_html` – nazwa katalogu ze stroną w katalogu domowym
- `UserDir disabled root` – root nie ma strony (bezpieczeństwo)

> 💡 **Wskazówka:** Jeśli zadanie mówi, że katalog ma się nazywać np. `www` zamiast `public_html` – zmień tu `UserDir public_html` na `UserDir www` **i** ścieżkę w `<Directory /home/*/www>`.

### Krok 2 – katalog public_html użytkownika

Najlepiej jako ten użytkownik (wtedy jest właścicielem plików):

```bash
su - jan
mkdir ~/public_html
echo "<h1>Strona Jana</h1>" > ~/public_html/index.html
exit
```

Albo jako admin:

```bash
sudo mkdir /home/jan/public_html
echo "<h1>Strona Jana</h1>" | sudo tee /home/jan/public_html/index.html
sudo chown -R jan:jan /home/jan/public_html
```

### Krok 3 – UPRAWNIENIA (tu wszyscy się wykładają!)

`www-data` musi móc **wejść** do `/home/jan` i **czytać** `public_html`.

> ⚠️ **Uwaga – pułapka Debiana:** Od Debiana 12 katalogi domowe tworzone przez `adduser` mają prawa **`700`** (`drwx------`) – nikt poza właścicielem nie wejdzie. Wynik: **403 Forbidden** przy wejściu na `/~jan/`.

```bash
ls -ld /home/jan                        # drwx------  ← problem!
sudo chmod 711 /home/jan                # pozostali mogą WEJŚĆ (x), ale nie listować
sudo chmod 755 /home/jan/public_html
sudo chmod 644 /home/jan/public_html/index.html
```

| Ścieżka | Minimalne prawa | Dlaczego |
|---------|-----------------|----------|
| `/home/jan` | `711` (`rwx--x--x`) | www-data musi przez niego **przejść** |
| `/home/jan/public_html` | `755` (`rwxr-xr-x`) | www-data musi wejść i odczytać listę |
| pliki strony | `644` (`rw-r--r--`) | www-data musi je czytać |

### Krok 4 – test

```bash
curl http://localhost/~jan/
```

> ⚠️ **Uwaga:** Pamiętaj o **tyldzie** `~` i ukośniku na końcu: `/~jan/`.

### Automatycznie dla nowych użytkowników – /etc/skel

Wszystko, co jest w `/etc/skel`, kopiuje się do katalogu domowego **każdego nowego** użytkownika.

```bash
sudo mkdir /etc/skel/public_html
echo "<h1>Moja strona domowa</h1>" | sudo tee /etc/skel/public_html/index.html
```

Żeby nowi użytkownicy mieli od razu katalog domowy z prawami `711` (zamiast `700`):

```bash
sudo nano /etc/adduser.conf
```
```
DIR_MODE=0711
```

Teraz `sudo adduser ola` → Ola od razu ma działającą stronę `http://serwer/~ola/`.

> 💡 **Wskazówka:** `/etc/skel` działa tylko dla użytkowników tworzonych **po** jego zmianie (i przez `adduser` albo `useradd -m`). Starym użytkownikom katalog tworzysz ręcznie.

---

## CZĘŚĆ 8: Ochrona hasłem i ograniczenie dostępu

### Uwierzytelnianie Basic – plik z hasłami

Hasła do stron WWW są **osobne** od haseł systemowych. Trzyma się je w pliku tworzonym poleceniem `htpasswd`:

```bash
sudo apt install -y apache2-utils               # zawiera htpasswd
sudo htpasswd -c /etc/apache2/.htpasswd szef    # -c = UTWÓRZ nowy plik
sudo htpasswd /etc/apache2/.htpasswd kierownik  # kolejny użytkownik – BEZ -c!
sudo htpasswd -D /etc/apache2/.htpasswd kierownik  # usuń użytkownika
cat /etc/apache2/.htpasswd                      # szef:$apr1$... (hasło zaszyfrowane)
```

> ⚠️ **Uwaga:** `-c` **nadpisuje** plik! Użyj go tylko przy **pierwszym** użytkowniku. Dodasz `-c` przy drugim – pierwszy zniknie.

> ⚠️ **Uwaga:** Plik z hasłami trzymaj **poza** `DocumentRoot` (np. w `/etc/apache2/`), żeby nikt nie mógł go pobrać przez przeglądarkę.

### Sposób 1 – w konfiguracji vhosta (zalecany)

Chronimy katalog `/var/www/firma/admin`:

```bash
sudo mkdir /var/www/firma/admin
echo "<h1>Panel administracyjny</h1>" | sudo tee /var/www/firma/admin/index.html
sudo nano /etc/apache2/sites-available/firma.conf
```
```apache
<VirtualHost *:80>
    ServerName www.firma.local
    DocumentRoot /var/www/firma

    <Directory /var/www/firma/admin>
        AuthType Basic
        AuthName "Strefa chroniona"
        AuthUserFile /etc/apache2/.htpasswd
        Require valid-user
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/firma_error.log
    CustomLog ${APACHE_LOG_DIR}/firma_access.log combined
</VirtualHost>
```

| Dyrektywa | Znaczenie |
|-----------|-----------|
| `AuthType Basic` | logowanie podstawowe (okienko loginu w przeglądarce) |
| `AuthName` | napis w okienku logowania |
| `AuthUserFile` | ścieżka do pliku z hasłami |
| `Require valid-user` | wpuść **każdego** użytkownika z pliku |
| `Require user szef` | wpuść **tylko** użytkownika `szef` |

```bash
sudo apache2ctl configtest && sudo systemctl reload apache2
curl -I http://www.firma.local/admin/                   # 401 Unauthorized
curl -u szef:haslo http://www.firma.local/admin/        # strona się wyświetla
```

### Sposób 2 – plik .htaccess

`.htaccess` to plik w katalogu strony, który zmienia ustawienia **tylko dla tego katalogu**. Przydaje się, gdy użytkownik (np. `jan`) chce sam zabezpieczyć swoją stronę.

**1. Pozwól na `.htaccess`** (w vhoście lub w `<Directory>`):

```apache
<Directory /var/www/firma>
    AllowOverride AuthConfig
</Directory>
```

**2. Utwórz plik:**

```bash
sudo nano /var/www/firma/admin/.htaccess
```
```apache
AuthType Basic
AuthName "Strefa chroniona"
AuthUserFile /etc/apache2/.htpasswd
Require valid-user
```

> 💡 **Wskazówka:** W katalogach domowych (`userdir`) `.htaccess` z `AuthConfig` działa **od razu** – moduł userdir ma już `AllowOverride FileInfo AuthConfig Limit Indexes`.

> ⚠️ **Uwaga:** Jeśli `AllowOverride None` – Apache **ignoruje** `.htaccess` i strona nie jest chroniona. Jeśli w `.htaccess` jest literówka – dostaniesz **500 Internal Server Error**.

### Ograniczenie dostępu po adresie IP

```apache
<Directory /var/www/intranet>
    Require ip 192.168.1.0/24
    Require ip 127.0.0.1
</Directory>
```

Kilka linii `Require` = wystarczy, że **jeden** warunek jest spełniony (LUB).

Pozostałe przykłady:

```apache
Require all granted                     # wszyscy
Require all denied                      # nikt
Require ip 192.168.1.50                 # tylko jeden komputer
Require host szkola.local               # tylko komputery z domeny
```

Zablokowanie jednego adresu, reszta wpuszczona:

```apache
<Directory /var/www/firma>
    <RequireAll>
        Require all granted
        Require not ip 192.168.1.66
    </RequireAll>
</Directory>
```

Hasło **i** sieć lokalna jednocześnie (oba warunki muszą być spełnione):

```apache
<Directory /var/www/firma/admin>
    AuthType Basic
    AuthName "Tylko z sieci szkolnej"
    AuthUserFile /etc/apache2/.htpasswd
    <RequireAll>
        Require valid-user
        Require ip 192.168.1.0/24
    </RequireAll>
</Directory>
```

---

## CZĘŚĆ 9: Listowanie katalogów, strony błędów, przekierowania

### Listowanie plików (Indexes)

Udostępnienie katalogu z plikami do pobrania (np. `/var/www/firma/pliki`):

```apache
<Directory /var/www/firma/pliki>
    Options +Indexes
</Directory>
```

Wyłączenie listowania w całej stronie (zalecane ze względów bezpieczeństwa):

```apache
<Directory /var/www/firma>
    Options -Indexes
</Directory>
```

### Własne strony błędów

```bash
echo "<h1>Ups! Nie ma takiej strony.</h1>" | sudo tee /var/www/firma/404.html
```

W vhoście:

```apache
ErrorDocument 404 /404.html
ErrorDocument 403 "Brak dostepu!"
```

### Przekierowania

```apache
Redirect permanent /stara /nowa                     # 301 – na stałe
Redirect /google https://www.google.com             # 302 – tymczasowo
```

### Dodatkowy katalog spoza DocumentRoot – Alias

Katalog `/srv/dokumenty` widoczny pod `http://serwer/dokumenty`:

```apache
Alias /dokumenty /srv/dokumenty
<Directory /srv/dokumenty>
    Options +Indexes
    Require all granted
</Directory>
```

---

## CZĘŚĆ 10: HTTPS w Apache

HTTPS wymaga **certyfikatu**. W szkole/na egzaminie tworzymy certyfikat **samopodpisany** (self-signed) – przeglądarka pokaże ostrzeżenie, ale połączenie będzie szyfrowane.

### Szybka wersja – domyślna strona SSL

Debian ma gotowy certyfikat testowy („snakeoil”) i gotowy vhost:

```bash
sudo a2enmod ssl
sudo a2ensite default-ssl
sudo systemctl restart apache2
curl -k -I https://localhost
```

### Pełna wersja – własny certyfikat dla vhosta

**1. Wygeneruj klucz i certyfikat**

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/firma.key \
    -out /etc/ssl/certs/firma.crt
```

| Opcja | Znaczenie |
|-------|-----------|
| `req -x509` | utwórz certyfikat (samopodpisany) |
| `-nodes` | klucz bez hasła (inaczej Apache pyta o hasło przy każdym starcie) |
| `-days 365` | ważny rok |
| `-newkey rsa:2048` | nowy klucz RSA 2048 bitów |
| `-keyout` | plik klucza prywatnego |
| `-out` | plik certyfikatu |

Program zada pytania (kraj, miasto, firma…). **Najważniejsze:**
```
Common Name (e.g. server FQDN or YOUR name) []: www.firma.local
```

> 💡 **Wskazówka:** Bez pytań – dopisz na końcu polecenia:
> `-subj "/C=PL/ST=Malopolskie/L=Krakow/O=Firma/CN=www.firma.local"`

**2. Vhost na porcie 443**

```bash
sudo a2enmod ssl
sudo nano /etc/apache2/sites-available/firma-ssl.conf
```
```apache
<VirtualHost *:443>
    ServerName www.firma.local
    DocumentRoot /var/www/firma

    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/firma.crt
    SSLCertificateKeyFile /etc/ssl/private/firma.key

    ErrorLog ${APACHE_LOG_DIR}/firma_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/firma_ssl_access.log combined
</VirtualHost>
```

```bash
sudo a2ensite firma-ssl
sudo apache2ctl configtest
sudo systemctl restart apache2
sudo ss -tulpn | grep :443
curl -k https://www.firma.local
```

> 💡 **Wskazówka:** `Listen 443` jest już w `ports.conf` – aktywuje się sam po włączeniu modułu `ssl`.

**3. Przekierowanie HTTP → HTTPS**

W vhoście na porcie 80 (`firma.conf`) zamiast `DocumentRoot`:

```apache
<VirtualHost *:80>
    ServerName www.firma.local
    ServerAlias firma.local
    Redirect permanent / https://www.firma.local/
</VirtualHost>
```

```bash
curl -I http://www.firma.local          # 301 Moved Permanently, Location: https://...
```

---

## CZĘŚĆ 11: PHP w Apache

```bash
sudo apt install -y php libapache2-mod-php
php -v                                  # wersja PHP
sudo systemctl restart apache2
```

Test:

```bash
echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info.php
curl -s http://localhost/info.php | head -20
```

Z klienta wejdź na `http://192.168.1.100/info.php` – zobaczysz tabelę z informacjami o PHP.

Prosty skrypt:

```bash
sudo nano /var/www/html/data.php
```
```php
<?php
echo "<h1>Dzisiaj jest: " . date("Y-m-d H:i") . "</h1>";
echo "<p>Twój adres IP: " . $_SERVER['REMOTE_ADDR'] . "</p>";
?>
```

> ⚠️ **Uwaga:** Po testach **usuń `info.php`** – zdradza dużo informacji o serwerze.

> ⚠️ **Uwaga – PHP w userdir:** Debian domyślnie **wyłącza PHP w `public_html`** użytkowników (bezpieczeństwo). Plik `.php` w `/~jan/` zostanie pobrany jako tekst zamiast się wykonać. Jeśli zadanie tego wymaga:
> ```bash
> ls /etc/apache2/mods-available/php*.conf        # sprawdź wersję, np. php8.4.conf
> sudo nano /etc/apache2/mods-available/php8.4.conf
> ```
> Zakomentuj (dopisz `#` na początku) blok:
> ```apache
> #<IfModule mod_userdir.c>
> #    <Directory /home/*/public_html>
> #        php_admin_flag engine Off
> #    </Directory>
> #</IfModule>
> ```
> i zrób `sudo systemctl restart apache2`.

### PHP + baza danych (gdy zadanie tego wymaga)

```bash
sudo apt install -y mariadb-server php-mysql
sudo systemctl restart apache2
```

---

## CZĘŚĆ 12: Logi i diagnostyka Apache

```bash
ls /var/log/apache2/                                # access.log, error.log, + logi vhostów
sudo tail -f /var/log/apache2/access.log            # odwiedziny na żywo (Ctrl+C = koniec)
sudo tail -n 20 /var/log/apache2/error.log          # ostatnie błędy
sudo tail -n 20 /var/log/apache2/firma_error.log    # błędy konkretnego vhosta
sudo journalctl -u apache2 -n 30                    # logi usługi (gdy Apache nie startuje)
```

Przykładowa linia access.log:

```
192.168.1.20 - szef [18/Nov/2026:10:15:32 +0100] "GET /admin/ HTTP/1.1" 200 312 "-" "Mozilla/5.0..."
```

| Fragment | Znaczenie |
|----------|-----------|
| `192.168.1.20` | IP klienta |
| `szef` | zalogowany użytkownik (Basic Auth) |
| `GET /admin/` | metoda i adres |
| `200` | kod odpowiedzi |
| `312` | rozmiar odpowiedzi w bajtach |

### Kolejność diagnozy, gdy „nie działa”

```bash
sudo apache2ctl configtest              # 1. czy składnia jest OK?
sudo systemctl status apache2           # 2. czy usługa działa?
sudo apache2ctl -S                      # 3. czy vhost jest włączony i na właściwym porcie?
sudo ss -tulpn | grep apache            # 4. na jakich portach nasłuchuje?
curl -I http://www.firma.local          # 5. jaki kod odpowiedzi?
sudo tail /var/log/apache2/error.log    # 6. co mówi log błędów?
```

---

# ŚCIEŻKA B – NGINX

Nginx (czyt. „endżin-iks”) to drugi najpopularniejszy serwer WWW. Jest lżejszy i szybszy przy dużym ruchu, ma **prostszą składnię**, ale **nie obsługuje `.htaccess`** i nie ma poleceń `a2*`.

W tej ścieżce robimy **to samo co w Apache**, ale nginx-em.

---

## CZĘŚĆ 13: Zamiana Apache na nginx

Apache i nginx **nie mogą** jednocześnie nasłuchiwać na porcie 80. Masz trzy opcje:

**Opcja 1 – nowa maszyna wirtualna** (najczystsza). Sklonuj VM z samym Debianem i tam instaluj nginx.

**Opcja 2 – wyłącz Apache, zostaw zainstalowany**

```bash
sudo systemctl disable --now apache2    # zatrzymaj i wyłącz z autostartu
sudo ss -tulpn | grep :80               # port 80 musi być WOLNY (brak wyniku)
```

**Opcja 3 – usuń Apache całkowicie**

```bash
sudo apt purge -y apache2 apache2-utils libapache2-mod-php*
sudo apt autoremove -y
```

### Instalacja nginx

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl status nginx             # active (running)
systemctl is-enabled nginx              # enabled
sudo ss -tulpn | grep nginx             # :80
nginx -v                                # wersja
curl -I http://localhost                # Server: nginx
```

> ⚠️ **Uwaga – „dlaczego nginx pokazuje stronę Apache?”** Jeśli wcześniej był zainstalowany Apache, w `/var/www/html/` został jego `index.html` („Apache2 Debian Default Page”). Nginx po prostu go wyświetla! Sprawdź nagłówek `Server:` w `curl -I` – tam zobaczysz, który serwer naprawdę odpowiada.

> ⚠️ **Uwaga:** Jeśli przy instalacji nginx pojawi się błąd `bind() to 0.0.0.0:80 failed (98: Address already in use)` – Apache nadal działa. Wyłącz go (Opcja 2) i wykonaj `sudo systemctl start nginx`.

### Zarządzanie usługą

```bash
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx             # przeładuj konfigurację
sudo systemctl enable nginx
```

---

## CZĘŚĆ 14: Pliki konfiguracyjne nginx

```
/etc/nginx/
├── nginx.conf            # główny plik (user www-data, include ...)
├── sites-available/      # WSZYSTKIE strony
│   └── default           # domyślna strona
├── sites-enabled/        # strony WŁĄCZONE (dowiązania)
│   └── default -> /etc/nginx/sites-available/default
├── conf.d/               # dodatkowa konfiguracja (*.conf)
└── snippets/             # gotowe fragmenty (np. fastcgi-php.conf, snakeoil.conf)
```

Logi: `/var/log/nginx/access.log` i `/var/log/nginx/error.log`.

### Włączanie stron – BEZ poleceń a2*

Nginx nie ma `a2ensite`. Stronę włączasz, tworząc **dowiązanie symboliczne** (`ln -s`, znasz z pierwszego materiału):

| Apache | nginx |
|--------|-------|
| `a2ensite firma` | `ln -s /etc/nginx/sites-available/firma /etc/nginx/sites-enabled/` |
| `a2dissite firma` | `rm /etc/nginx/sites-enabled/firma` |
| `apache2ctl configtest` | `nginx -t` |
| `apache2ctl -S` | `nginx -T` (pokazuje całą konfigurację) |

> ⚠️ **Uwaga:** Przy `ln -s` **zawsze podawaj pełną ścieżkę** źródła (`/etc/nginx/sites-available/firma`). Ścieżka względna da „martwy” link.

> 💡 **Wskazówka:** W nginx pliki w `sites-available` **nie muszą** mieć rozszerzenia `.conf`.

Złota zasada – po każdej zmianie:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### Składnia nginx – różnice względem Apache

```nginx
server {
    listen 80;
    root /var/www/html;
}
```

- bloki w **nawiasach klamrowych** `{ }` (a nie `<VirtualHost>...</VirtualHost>`)
- **każda dyrektywa kończy się średnikiem `;`** – brak średnika to najczęstszy błąd!
- komentarze od `#`

### Domyślna strona – `sites-available/default`

```bash
grep -v "#" /etc/nginx/sites-available/default | grep -v "^\s*$"
```
```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /var/www/html;
    index index.html index.htm index.nginx-debian.html;

    server_name _;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

| Dyrektywa | Odpowiednik w Apache | Znaczenie |
|-----------|---------------------|-----------|
| `listen 80` | `<VirtualHost *:80>` + `Listen 80` | port (nginx sam go otwiera – nie ma `ports.conf`!) |
| `default_server` | pierwszy vhost (`000-default`) | strona pokazywana, gdy nazwa nie pasuje |
| `listen [::]:80` | – | to samo dla IPv6 |
| `root` | `DocumentRoot` | katalog ze stroną |
| `index` | `DirectoryIndex` | plik startowy |
| `server_name` | `ServerName` + `ServerAlias` | nazwy strony (`_` = dowolna) |
| `location / { }` | `<Directory>` / `<Location>` | ustawienia dla danej ścieżki URL |
| `try_files $uri $uri/ =404` | – | szukaj pliku, potem katalogu, inaczej 404 |

### Własna strona

```bash
echo "<h1>Serwer nginx Jana Kowalskiego</h1>" | sudo tee /var/www/html/index.html
curl http://localhost
```

Uprawnienia – **identycznie jak w Apache** (Część 5): nginx też działa jako `www-data`.

---

## CZĘŚĆ 15: Hosty wirtualne w nginx (server blocks)

W nginx host wirtualny to **blok `server { }`**.

### Name-based

```bash
sudo mkdir -p /var/www/firma /var/www/szkola
echo "<h1>Strona FIRMY (nginx)</h1>"  | sudo tee /var/www/firma/index.html
echo "<h1>Strona SZKOLY (nginx)</h1>" | sudo tee /var/www/szkola/index.html
sudo nano /etc/nginx/sites-available/firma
```
```nginx
server {
    listen 80;
    server_name www.firma.local firma.local;

    root /var/www/firma;
    index index.html index.htm index.php;

    access_log /var/log/nginx/firma_access.log;
    error_log  /var/log/nginx/firma_error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

> 💡 **Wskazówka:** `server_name` przyjmuje **wiele nazw po spacji** – to odpowiednik `ServerName` + `ServerAlias` naraz.

Drugi vhost:

```bash
sudo cp /etc/nginx/sites-available/firma /etc/nginx/sites-available/szkola
sudo sed -i 's/firma/szkola/g' /etc/nginx/sites-available/szkola
```

Włączenie:

```bash
sudo ln -s /etc/nginx/sites-available/firma  /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/szkola /etc/nginx/sites-enabled/
ls -l /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

Test:

```bash
curl http://www.firma.local
curl http://www.szkola.local
curl -H "Host: www.firma.local" http://192.168.1.100
```

### Który server jest domyślny?

Ten z **`default_server`** w linii `listen`. Jeśli zadanie mówi, że po wpisaniu IP ma się pokazać strona firmy:

```bash
sudo rm /etc/nginx/sites-enabled/default        # wyłącz domyślną stronę
sudo nano /etc/nginx/sites-available/firma
```
```nginx
    listen 80 default_server;
```

> ⚠️ **Uwaga:** `default_server` może być tylko **jeden** na danym porcie. Dwa = błąd `duplicate default server` w `nginx -t`.

### Port-based – port 8080

W nginx **nie ma** `ports.conf` – wystarczy `listen` w bloku `server`:

```bash
sudo mkdir -p /var/www/intranet
echo "<h1>INTRANET (nginx)</h1>" | sudo tee /var/www/intranet/index.html
sudo nano /etc/nginx/sites-available/intranet
```
```nginx
server {
    listen 8080;
    root /var/www/intranet;
    index index.html;

    access_log /var/log/nginx/intranet_access.log;
    error_log  /var/log/nginx/intranet_error.log;
}
```
```bash
sudo ln -s /etc/nginx/sites-available/intranet /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo ss -tulpn | grep nginx             # :80 i :8080
curl http://localhost:8080
```

### IP-based

```nginx
server {
    listen 192.168.1.100:80;
    root /var/www/firma;
}
server {
    listen 192.168.1.101:80;
    root /var/www/szkola;
}
```

### Katalog poza /var/www

W nginx **nie trzeba** nic dodatkowo zezwalać (jak `Require all granted` w Apache) – wystarczy `root /srv/www/firma;` i odpowiednie uprawnienia plików dla `www-data`.

---

## CZĘŚĆ 16: Katalogi domowe użytkowników w nginx

Nginx **nie ma modułu userdir** – ten sam efekt (`http://serwer/~jan/` → `/home/jan/public_html/`) uzyskujemy jednym blokiem `location` z wyrażeniem regularnym.

### Krok 1 – konfiguracja

Dopisz do bloku `server` strony domyślnej (lub tej, pod którą mają działać strony użytkowników):

```bash
sudo nano /etc/nginx/sites-available/default
```
```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    root /var/www/html;
    index index.html index.htm index.nginx-debian.html;
    server_name _;

    location / {
        try_files $uri $uri/ =404;
    }

    # strony użytkowników: /~jan/ -> /home/jan/public_html/
    location ~ ^/~([^/]+)(/.*)?$ {
        alias /home/$1/public_html$2;
        index index.html index.htm;
        autoindex on;
    }
}
```

Co tu się dzieje?

| Fragment | Znaczenie |
|----------|-----------|
| `location ~` | `~` = dopasowanie wyrażeniem regularnym |
| `^/~` | adres zaczyna się od `/~` |
| `([^/]+)` | **$1** – nazwa użytkownika (wszystko do następnego `/`) |
| `(/.*)?` | **$2** – reszta ścieżki (może jej nie być) |
| `alias /home/$1/public_html$2` | podstaw nazwę użytkownika do ścieżki |
| `autoindex on` | pokazuj listę plików, gdy brak `index.html` (jak `Indexes` w userdir Apache) |

Przykład: `/~jan/galeria/foto.jpg` → `$1 = jan`, `$2 = /galeria/foto.jpg` → `/home/jan/public_html/galeria/foto.jpg`.

> 💡 **Wskazówka:** Nie musisz rozumieć wyrażenia regularnego na pamięć – **przepisz je dokładnie**. Warto mieć tę linię w notatkach.

### Krok 2 – katalogi i uprawnienia

**Dokładnie tak samo jak w Apache** (Część 7):

```bash
sudo mkdir /home/jan/public_html
echo "<h1>Strona Jana (nginx)</h1>" | sudo tee /home/jan/public_html/index.html
sudo chown -R jan:jan /home/jan/public_html
sudo chmod 711 /home/jan
sudo chmod 755 /home/jan/public_html
```

Pułapka z prawami `700` w Debianie i trik z `/etc/skel` – identycznie jak w Części 7.

### Krok 3 – test

```bash
sudo nginx -t && sudo systemctl reload nginx
curl http://localhost/~jan/
```

### Wyłączenie strony roota

Żeby nikt nie podglądał `/root` (który i tak nie jest w `/home`, ale na wszelki wypadek), dopisz **przed** blokiem userdir:

```nginx
    location ~ ^/~root { return 403; }
```

---

## CZĘŚĆ 17: Ochrona hasłem i ograniczenie dostępu w nginx

### Plik z hasłami

Nginx używa **tego samego formatu** pliku co Apache. Program `htpasswd` jest w pakiecie `apache2-utils` – to **tylko narzędzia**, nie instaluje serwera Apache.

```bash
sudo apt install -y apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd szef      # -c tylko za pierwszym razem!
sudo htpasswd /etc/nginx/.htpasswd kierownik
```

> 💡 **Wskazówka – bez apache2-utils:** hasło można wygenerować przez `openssl`:
> ```bash
> echo "szef:$(openssl passwd -apr1 TajneHaslo)" | sudo tee -a /etc/nginx/.htpasswd
> ```

### Zabezpieczenie katalogu

```nginx
server {
    listen 80;
    server_name www.firma.local firma.local;
    root /var/www/firma;
    index index.html;

    location /admin/ {
        auth_basic "Strefa chroniona";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
```

| nginx | Apache |
|-------|--------|
| `auth_basic "Napis";` | `AuthType Basic` + `AuthName "Napis"` |
| `auth_basic_user_file plik;` | `AuthUserFile plik` + `Require valid-user` |

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -I http://www.firma.local/admin/                   # 401
curl -u szef:TajneHaslo http://www.firma.local/admin/   # OK
```

> ⚠️ **Uwaga:** Nginx **nie czyta `.htaccess`**! Jeśli zostawisz plik `.htaccess` z przeniesionej strony Apache – nic nie robi, a co gorsza, nginx może go **wyświetlić** każdemu. Zablokuj pliki ukryte:
> ```nginx
>     location ~ /\.ht { deny all; }
> ```

> ⚠️ **Uwaga – konsekwencja dla userdir:** W Apache użytkownik `jan` mógł sam zabezpieczyć swoją stronę plikiem `.htaccess`. W nginx **tylko administrator** może to zrobić – w konfiguracji serwera:
> ```nginx
>     location ^~ /~jan/prywatne/ {
>         alias /home/jan/public_html/prywatne/;
>         auth_basic "Prywatne pliki Jana";
>         auth_basic_user_file /etc/nginx/.htpasswd_jan;
>     }
> ```
> (`^~` sprawia, że ten blok ma pierwszeństwo przed wyrażeniem regularnym userdir.)

### Ograniczenie po adresie IP

```nginx
    location / {
        allow 192.168.1.0/24;
        allow 127.0.0.1;
        deny all;
    }
```

> ⚠️ **Uwaga:** W nginx **kolejność ma znaczenie** – reguły sprawdzane są od góry, pierwsza pasująca wygrywa. `deny all` zawsze **na końcu**.

Zablokowanie jednego IP:

```nginx
    deny 192.168.1.66;
    allow all;
```

Hasło **i** sieć lokalna (oba warunki):

```nginx
    location /admin/ {
        satisfy all;                        # all = oba warunki; any = wystarczy jeden
        allow 192.168.1.0/24;
        deny all;
        auth_basic "Tylko z sieci szkolnej";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
```

---

## CZĘŚĆ 18: Listowanie, strony błędów, przekierowania w nginx

### Listowanie plików

```nginx
    location /pliki/ {
        autoindex on;
        autoindex_exact_size off;           # rozmiary w KB/MB zamiast bajtów
        autoindex_localtime on;             # czas lokalny
    }
```

Domyślnie `autoindex` jest **wyłączony** (odwrotnie niż w Apache, gdzie `Indexes` jest domyślnie włączone dla `/var/www`).

### Własne strony błędów

```nginx
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;
```

### Przekierowania

```nginx
    location = /stara { return 301 /nowa; }
    location /google  { return 302 https://www.google.com; }
```

### Alias – katalog spoza root

```nginx
    location /dokumenty/ {
        alias /srv/dokumenty/;
        autoindex on;
    }
```

> ⚠️ **Uwaga:** Przy `alias` ukośniki na końcu muszą się zgadzać: `location /dokumenty/` ↔ `alias /srv/dokumenty/;`.

### Rodzaje `location` (dla ciekawych)

| Zapis | Znaczenie | Priorytet |
|-------|-----------|-----------|
| `location = /x` | dokładnie `/x` | najwyższy |
| `location ^~ /x/` | zaczyna się od `/x/`, bez sprawdzania regexów | wysoki |
| `location ~ wzorzec` | wyrażenie regularne (wielkość liter ma znaczenie) | średni |
| `location ~* wzorzec` | wyrażenie regularne (bez rozróżniania liter) | średni |
| `location /x/` | zaczyna się od `/x/` | najniższy |

---

## CZĘŚĆ 19: HTTPS w nginx

### Szybka wersja – certyfikat testowy Debiana

```bash
sudo apt install -y ssl-cert            # zwykle już jest
```
W bloku `server` strony domyślnej odkomentuj / dopisz:
```nginx
    listen 443 ssl default_server;
    include snippets/snakeoil.conf;
```

### Pełna wersja – własny certyfikat

**1. Certyfikat** – to samo polecenie co w Apache:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/firma.key \
    -out /etc/ssl/certs/firma.crt \
    -subj "/C=PL/O=Firma/CN=www.firma.local"
```

**2. Konfiguracja** – w jednym pliku obie wersje (HTTP z przekierowaniem i HTTPS):

```bash
sudo nano /etc/nginx/sites-available/firma
```
```nginx
# HTTP -> przekierowanie na HTTPS
server {
    listen 80;
    server_name www.firma.local firma.local;
    return 301 https://www.firma.local$request_uri;
}

# HTTPS
server {
    listen 443 ssl;
    server_name www.firma.local firma.local;

    ssl_certificate     /etc/ssl/certs/firma.crt;
    ssl_certificate_key /etc/ssl/private/firma.key;

    root /var/www/firma;
    index index.html index.php;

    access_log /var/log/nginx/firma_ssl_access.log;
    error_log  /var/log/nginx/firma_ssl_error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

| nginx | Apache |
|-------|--------|
| `listen 443 ssl;` | `<VirtualHost *:443>` + `SSLEngine on` + `a2enmod ssl` |
| `ssl_certificate` | `SSLCertificateFile` |
| `ssl_certificate_key` | `SSLCertificateKeyFile` |
| `return 301 https://...$request_uri;` | `Redirect permanent / https://.../` |

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -I http://www.firma.local              # 301 -> https
curl -k https://www.firma.local             # strona
```

> 💡 **Wskazówka:** W nginx **nie trzeba** włączać żadnego modułu SSL – jest wbudowany.

---

## CZĘŚĆ 20: PHP w nginx (PHP-FPM)

To **największa różnica** między serwerami:

- **Apache** ma PHP „w środku” (moduł `libapache2-mod-php`) – instalujesz i działa.
- **nginx** nie umie wykonywać PHP. Przekazuje pliki `.php` do **osobnej usługi PHP-FPM** i odbiera wynik.

```
przeglądarka → nginx → (gniazdo /run/php/php8.4-fpm.sock) → PHP-FPM → wynik → nginx → przeglądarka
```

### Instalacja

```bash
sudo apt install -y php-fpm
sudo systemctl status php*-fpm          # usługa musi działać!
ls /run/php/                            # nazwa gniazda, np. php8.4-fpm.sock
```

> ⚠️ **Uwaga:** Wersja PHP w nazwie (`8.4`) zależy od wersji Debiana. **Zawsze sprawdź** `ls /run/php/` i wpisz dokładnie tę nazwę.

> ⚠️ **Uwaga:** Instaluj `php-fpm`, a **nie** samo `php` – pakiet `php` może pociągnąć za sobą Apache (`libapache2-mod-php`)!

### Konfiguracja

W bloku `server`:

```nginx
    index index.php index.html;

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
```

```bash
sudo nginx -t && sudo systemctl reload nginx
echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info.php
curl -s http://localhost/info.php | grep "PHP Version"
```

### PHP w katalogach użytkowników (nginx)

Blok userdir obsługuje tylko pliki statyczne. Żeby `.php` w `/~jan/` się wykonywało, **przed** zwykłym blokiem userdir dodaj:

```nginx
    location ~ ^/~([^/]+)(/.*\.php)$ {
        alias /home/$1/public_html$2;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME /home/$1/public_html$2;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }
```

### Objawy błędów PHP w nginx

| Objaw | Przyczyna |
|-------|-----------|
| przeglądarka **pobiera** plik `.php` zamiast go wyświetlić | brak bloku `location ~ \.php$` |
| **502 Bad Gateway** | PHP-FPM nie działa albo zła nazwa gniazda w `fastcgi_pass` |
| pusta strona / „File not found.” | zła ścieżka `root` lub brak uprawnień |

---

## CZĘŚĆ 21: Logi i diagnostyka nginx

```bash
ls /var/log/nginx/
sudo tail -f /var/log/nginx/access.log
sudo tail -n 20 /var/log/nginx/error.log
sudo journalctl -u nginx -n 30
sudo journalctl -u php8.4-fpm -n 30     # logi PHP-FPM
```

Kolejność diagnozy:

```bash
sudo nginx -t                           # 1. składnia (pokazuje plik i NUMER LINII błędu!)
sudo systemctl status nginx             # 2. czy działa?
ls -l /etc/nginx/sites-enabled/         # 3. czy strona jest włączona (link nie jest „czerwony”)?
sudo ss -tulpn | grep nginx             # 4. porty
curl -I http://www.firma.local          # 5. kod odpowiedzi
sudo tail /var/log/nginx/error.log      # 6. log błędów
```

Przykładowy błąd z `nginx -t`:

```
nginx: [emerg] unexpected "}" in /etc/nginx/sites-enabled/firma:9
```
→ w pliku `firma`, w okolicy **linii 9**, brakuje średnika `;` na końcu poprzedniej linii.

---

## CZĘŚĆ 22: Inne serwery WWW

Na egzaminie liczy się głównie Apache, ale dobrze wiedzieć, że jest ich więcej.

| Serwer | Kiedy się go używa |
|--------|--------------------|
| **Apache** | uniwersalny, `.htaccess`, najwięcej poradników, standard na hostingach |
| **nginx** | duży ruch, serwer pośredniczący (reverse proxy), nowoczesne serwery |
| **lighttpd** | słabe komputery (np. Raspberry Pi, routery) |
| **Caddy** | automatyczny HTTPS, bardzo prosta konfiguracja |
| **IIS** | Windows Server (osobny temat na INF.02!) |
| `python3 -m http.server` | szybkie udostępnienie plików na chwilę |

> ⚠️ **Uwaga:** Każdy z tych serwerów chce port 80. Przed testem wyłącz poprzedni: `sudo systemctl disable --now apache2 nginx`.

### lighttpd

```bash
sudo apt install -y lighttpd
sudo nano /etc/lighttpd/lighttpd.conf
```
Najważniejsze linie:
```
server.document-root = "/var/www/html"
server.port          = 80
```
Moduły włącza się podobnie jak w Apache:
```bash
sudo lighty-enable-mod userdir          # strony /~uzytkownik/ (katalog public_html)
sudo lighty-enable-mod dir-listing      # listowanie katalogów
sudo lighty-enable-mod fastcgi-php      # PHP (wymaga php-cgi)
sudo systemctl restart lighttpd
```

Host wirtualny:
```
$HTTP["host"] == "www.firma.local" {
    server.document-root = "/var/www/firma"
}
```

### Caddy

```bash
sudo apt install -y caddy
sudo nano /etc/caddy/Caddyfile
```
```
:80 {
    root * /var/www/html
    file_server
}

www.firma.local:80 {
    root * /var/www/firma
    file_server browse                  # browse = listowanie plików
}
```
```bash
sudo systemctl reload caddy
```

### Python – serwer „na chwilę”

```bash
cd ~/szkola
python3 -m http.server 8000             # udostępnia bieżący katalog na porcie 8000
```
Z klienta: `http://192.168.1.100:8000`. Zatrzymanie: **Ctrl+C**. To **nie jest** serwer produkcyjny – tylko do szybkiego przesłania plików.

---

## CZĘŚĆ 23: Typowe błędy i jak je naprawić

| Objaw | Prawdopodobna przyczyna | Co zrobić |
|-------|------------------------|-----------|
| **403 Forbidden** na `/~jan/` | katalog domowy ma prawa `700` | `sudo chmod 711 /home/jan` |
| **403 Forbidden** na stronie | brak `index.html` i wyłączone listowanie / złe prawa / brak `Require all granted` (Apache, katalog poza `/var/www`) | sprawdź `ls -l`, nazwę pliku, blok `<Directory>` |
| **404 Not Found** na `/~jan/` | nie włączony `userdir` (Apache) / brak bloku `location` (nginx) / brak katalogu `public_html` | `a2enmod userdir` / sprawdź konfigurację |
| zawsze pokazuje się strona domyślna zamiast vhosta | vhost niewłączony / brak wpisu w hosts / literówka w `ServerName` | `apache2ctl -S`, `ls sites-enabled`, sprawdź `/etc/hosts` |
| usługa nie startuje, `Address already in use` | port zajęty przez inny serwer | `sudo ss -tulpn \| grep :80` i wyłącz drugi serwer |
| `Syntax error` / `[emerg]` | literówka w konfiguracji | `apache2ctl configtest` / `nginx -t` pokaże linię |
| port 8080 nie działa (Apache) | brak `Listen 8080` w `ports.conf` | dopisz i zrób `restart` |
| **500 Internal Server Error** (Apache) | błąd w `.htaccess` | `tail /var/log/apache2/error.log` |
| `.php` się pobiera zamiast wykonać | brak modułu PHP (Apache) / brak `location ~ \.php$` (nginx) / PHP wyłączone w userdir | Części 11 i 20 |
| **502 Bad Gateway** (nginx) | PHP-FPM nie działa / zła nazwa gniazda | `systemctl status php*-fpm`, `ls /run/php/` |
| hasło nie działa, choć „na pewno dobre” | użyto `-c` przy drugim użytkowniku – plik nadpisany | `cat .htpasswd`, dodaj użytkowników ponownie |
| `.htaccess` nic nie zmienia (Apache) | `AllowOverride None` | ustaw `AllowOverride AuthConfig` lub `All` |
| `.htaccess` nic nie zmienia (nginx) | nginx **nie obsługuje** `.htaccess` | przenieś ustawienia do konfiguracji serwera |
| polskie znaki to „krzaczki” | brak `<meta charset="UTF-8">` | dopisz w `<head>` |
| z klienta nie ma połączenia, `curl` na serwerze działa | karta VM w trybie NAT / zły adres IP / brak wpisu w hosts klienta | Część 2 |

---

## Słownik: Apache ↔ nginx

| Zadanie | Apache | nginx |
|---------|--------|-------|
| Pakiet | `apache2` | `nginx` |
| Usługa | `apache2` | `nginx` |
| Katalog konfiguracji | `/etc/apache2/` | `/etc/nginx/` |
| Logi | `/var/log/apache2/` | `/var/log/nginx/` |
| Test składni | `apache2ctl configtest` | `nginx -t` |
| Włącz stronę | `a2ensite firma` | `ln -s /etc/nginx/sites-available/firma /etc/nginx/sites-enabled/` |
| Wyłącz stronę | `a2dissite firma` | `rm /etc/nginx/sites-enabled/firma` |
| Host wirtualny | `<VirtualHost *:80> ... </VirtualHost>` | `server { listen 80; ... }` |
| Nazwa strony | `ServerName` + `ServerAlias` | `server_name a b c;` |
| Katalog strony | `DocumentRoot` | `root` |
| Plik startowy | `DirectoryIndex` | `index` |
| Nowy port | `Listen 8080` w `ports.conf` + `<VirtualHost *:8080>` | `listen 8080;` |
| Strona domyślna | pierwszy plik (`000-default.conf`) | `listen 80 default_server;` |
| Userdir | `a2enmod userdir` | `location ~ ^/~([^/]+)(/.*)?$ { alias /home/$1/public_html$2; }` |
| Listowanie | `Options +Indexes` | `autoindex on;` |
| Hasło | `AuthType Basic`, `AuthName`, `AuthUserFile`, `Require valid-user` | `auth_basic "...";` `auth_basic_user_file ...;` |
| Plik haseł | `htpasswd` (apache2-utils) | `htpasswd` (apache2-utils) lub `openssl passwd -apr1` |
| `.htaccess` | ✓ (gdy `AllowOverride`) | ✗ – nie obsługuje |
| Dostęp po IP | `Require ip 192.168.1.0/24` | `allow 192.168.1.0/24; deny all;` |
| Strona błędu | `ErrorDocument 404 /404.html` | `error_page 404 /404.html;` |
| Przekierowanie | `Redirect permanent / https://...` | `return 301 https://...$request_uri;` |
| Alias | `Alias /x /srv/x` | `location /x/ { alias /srv/x/; }` |
| HTTPS | `a2enmod ssl`, `SSLEngine on`, `SSLCertificateFile`, `SSLCertificateKeyFile` | `listen 443 ssl;` `ssl_certificate`, `ssl_certificate_key` |
| PHP | `libapache2-mod-php` – działa od razu | `php-fpm` + `location ~ \.php$ { fastcgi_pass ...; }` |
| Koniec linii | bez znaku | **średnik `;`** |

---

## Ściąga – najważniejsze polecenia

| Kategoria | Apache | nginx |
|-----------|--------|-------|
| Instalacja | `apt install apache2` | `apt install nginx` |
| Stan | `systemctl status apache2` | `systemctl status nginx` |
| Test + przeładowanie | `apache2ctl configtest && systemctl reload apache2` | `nginx -t && systemctl reload nginx` |
| Strony | `a2ensite`, `a2dissite` | `ln -s`, `rm` w `sites-enabled` |
| Moduły | `a2enmod userdir ssl` | – (wbudowane) |
| Lista vhostów | `apache2ctl -S` | `nginx -T` |
| Hasła | `htpasswd -c plik user` (pierwszy), `htpasswd plik user` (kolejni) | to samo |
| Certyfikat | `openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ... -out ...` | to samo |
| PHP | `apt install php libapache2-mod-php` | `apt install php-fpm` |
| Wspólne | `curl -I`, `curl -u`, `curl -k`, `ss -tulpn`, `/etc/hosts`, `chmod 711 /home/user` | |

---

## Zadania do samodzielnego wykonania

Wykonaj zadania 1–8 w **Apache**. Następnie wykonaj zadanie 9 – czyli to samo w **nginx**.

**Zadanie 1 – Instalacja**
Zainstaluj serwer Apache. Sprawdź, czy działa, czy uruchamia się automatycznie i na jakim porcie nasłuchuje. Podmień stronę domyślną na własną z napisem „Serwer – Twoje imię i nazwisko” (z polskimi znakami, które mają się poprawnie wyświetlać). Sprawdź kod odpowiedzi serwera.

**Zadanie 2 – Hosty wirtualne po nazwie**
Utwórz dwie strony:
- `www.sklep.local` (alias `sklep.local`) – pliki w `/var/www/sklep`, napis „Sklep”,
- `www.blog.local` – pliki w `/var/www/blog`, napis „Blog”.

Każda strona ma mieć własne pliki logów. Dodaj wpisy do `/etc/hosts` i sprawdź obie strony poleceniem `curl`.

**Zadanie 3 – Host wirtualny na porcie**
Utwórz stronę „Intranet” w katalogu `/srv/intranet`, dostępną na porcie **8080**. Zwróć uwagę, że katalog jest poza `/var/www`.

**Zadanie 4 – Strony użytkowników**
- Utwórz użytkowników `adam` i `ewa`.
- Skonfiguruj serwer tak, żeby każdy z nich miał stronę pod adresem `http://IP_serwera/~login/`.
- Zadbaj, żeby **każdy nowy** użytkownik automatycznie dostawał katalog `public_html` z plikiem `index.html` o treści „Strona w budowie”. Sprawdź to, tworząc użytkownika `ola`.

**Zadanie 5 – Ochrona hasłem**
Na stronie `www.sklep.local` utwórz katalog `admin`. Dostęp do niego mają mieć tylko użytkownicy `szef` i `kierownik` (hasła dowolne). Sprawdź dostęp bez hasła i z hasłem.

**Zadanie 6 – Ograniczenie po IP**
Strona „Intranet” (port 8080) ma być dostępna tylko z sieci `192.168.1.0/24` oraz z samego serwera.

**Zadanie 7 – HTTPS**
Wygeneruj certyfikat samopodpisany dla `www.sklep.local` (ważny 365 dni). Uruchom stronę sklepu po HTTPS i ustaw przekierowanie z HTTP na HTTPS.

**Zadanie 8 – PHP**
Zainstaluj obsługę PHP. Na stronie `www.blog.local` utwórz plik `index.php`, który wyświetla aktualną datę i adres IP odwiedzającego. Plik `index.php` ma mieć pierwszeństwo przed `index.html`.

**Zadanie 9 – Zamiana na nginx**
Wyłącz Apache (tak, by nie uruchamiał się przy starcie systemu). Zainstaluj nginx i wykonaj **ponownie zadania 2–8**, tym razem w nginx. Pliki stron (`/var/www/...`, `/srv/intranet`, `public_html`) i certyfikat możesz wykorzystać ponownie.

**Zadanie 10 – Szybkie udostępnienie plików (dodatkowe)**
Bez instalowania żadnego pakietu udostępnij zawartość katalogu `/var/log` na porcie 8000. Sprawdź z klienta, potem zatrzymaj serwer. Zastanów się: dlaczego nie widać niektórych plików / nie da się ich otworzyć?

---

## Rozwiązania

<details>
<summary><b>Zadanie 1</b></summary>

```bash
sudo apt update
sudo apt install -y apache2 curl
sudo systemctl status apache2           # active (running)
systemctl is-enabled apache2            # enabled
sudo ss -tulpn | grep apache            # :80
sudo cp /var/www/html/index.html /var/www/html/index.html.bak
sudo nano /var/www/html/index.html
```
```html
<!DOCTYPE html>
<html lang="pl">
<head><meta charset="UTF-8"><title>Serwer</title></head>
<body><h1>Serwer – Jan Kowalski</h1></body>
</html>
```
```bash
curl -I http://localhost                # HTTP/1.1 200 OK
curl http://localhost
```
</details>

<details>
<summary><b>Zadanie 2</b></summary>

```bash
sudo mkdir -p /var/www/sklep /var/www/blog
echo "<h1>Sklep</h1>" | sudo tee /var/www/sklep/index.html
echo "<h1>Blog</h1>"  | sudo tee /var/www/blog/index.html
sudo nano /etc/apache2/sites-available/sklep.conf
```
```apache
<VirtualHost *:80>
    ServerName www.sklep.local
    ServerAlias sklep.local
    DocumentRoot /var/www/sklep
    ErrorLog ${APACHE_LOG_DIR}/sklep_error.log
    CustomLog ${APACHE_LOG_DIR}/sklep_access.log combined
</VirtualHost>
```
```bash
sudo nano /etc/apache2/sites-available/blog.conf
```
```apache
<VirtualHost *:80>
    ServerName www.blog.local
    DocumentRoot /var/www/blog
    ErrorLog ${APACHE_LOG_DIR}/blog_error.log
    CustomLog ${APACHE_LOG_DIR}/blog_access.log combined
</VirtualHost>
```
```bash
sudo a2ensite sklep blog
sudo apache2ctl configtest && sudo systemctl reload apache2
sudo apache2ctl -S
sudo nano /etc/hosts
```
```
192.168.1.100   www.sklep.local sklep.local www.blog.local
```
```bash
curl http://www.sklep.local             # Sklep
curl http://sklep.local                 # Sklep
curl http://www.blog.local              # Blog
ls /var/log/apache2/                    # sklep_*.log, blog_*.log
```
</details>

<details>
<summary><b>Zadanie 3</b></summary>

```bash
sudo mkdir -p /srv/intranet
echo "<h1>Intranet</h1>" | sudo tee /srv/intranet/index.html
sudo nano /etc/apache2/ports.conf       # dopisz: Listen 8080
sudo nano /etc/apache2/sites-available/intranet.conf
```
```apache
<VirtualHost *:8080>
    DocumentRoot /srv/intranet
    <Directory /srv/intranet>
        Require all granted
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/intranet_error.log
    CustomLog ${APACHE_LOG_DIR}/intranet_access.log combined
</VirtualHost>
```
```bash
sudo a2ensite intranet
sudo apache2ctl configtest && sudo systemctl restart apache2
sudo ss -tulpn | grep apache            # :80 i :8080
curl http://localhost:8080              # Intranet
```
Bez bloku `<Directory /srv/intranet>` → **403 Forbidden** (katalog poza `/var/www`).
</details>

<details>
<summary><b>Zadanie 4</b></summary>

```bash
# najpierw szablon dla NOWYCH użytkowników
sudo mkdir /etc/skel/public_html
echo "<h1>Strona w budowie</h1>" | sudo tee /etc/skel/public_html/index.html
sudo nano /etc/adduser.conf             # DIR_MODE=0711

# użytkownicy (dostaną public_html z /etc/skel)
sudo adduser adam
sudo adduser ewa

# moduł
sudo a2enmod userdir
sudo systemctl restart apache2

# sprawdzenie uprawnień
ls -ld /home/adam /home/ewa             # drwx--x--x
curl http://localhost/~adam/            # Strona w budowie
curl http://localhost/~ewa/

# test dla nowego użytkownika
sudo adduser ola
curl http://localhost/~ola/             # Strona w budowie
```
Jeśli `adam` i `ewa` istniały wcześniej (prawa `700`, bez `public_html`):
```bash
sudo chmod 711 /home/adam /home/ewa
sudo cp -r /etc/skel/public_html /home/adam/
sudo chown -R adam:adam /home/adam/public_html
# to samo dla ewy
```
</details>

<details>
<summary><b>Zadanie 5</b></summary>

```bash
sudo mkdir /var/www/sklep/admin
echo "<h1>Panel admina</h1>" | sudo tee /var/www/sklep/admin/index.html
sudo apt install -y apache2-utils
sudo htpasswd -c /etc/apache2/.htpasswd szef
sudo htpasswd /etc/apache2/.htpasswd kierownik       # BEZ -c
sudo nano /etc/apache2/sites-available/sklep.conf
```
Wewnątrz `<VirtualHost>` dopisz:
```apache
    <Directory /var/www/sklep/admin>
        AuthType Basic
        AuthName "Panel sklepu"
        AuthUserFile /etc/apache2/.htpasswd
        Require user szef kierownik
    </Directory>
```
```bash
sudo apache2ctl configtest && sudo systemctl reload apache2
curl -I http://www.sklep.local/admin/                   # 401 Unauthorized
curl -u szef:haslo http://www.sklep.local/admin/        # Panel admina
```
</details>

<details>
<summary><b>Zadanie 6</b></summary>

```bash
sudo nano /etc/apache2/sites-available/intranet.conf
```
```apache
    <Directory /srv/intranet>
        Require ip 192.168.1.0/24
        Require ip 127.0.0.1
    </Directory>
```
```bash
sudo apache2ctl configtest && sudo systemctl reload apache2
curl -I http://localhost:8080           # 200 OK
```
Test z komputera spoza sieci `192.168.1.0/24` → **403 Forbidden**.
</details>

<details>
<summary><b>Zadanie 7</b></summary>

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/sklep.key \
    -out /etc/ssl/certs/sklep.crt \
    -subj "/C=PL/O=Sklep/CN=www.sklep.local"
sudo a2enmod ssl
sudo nano /etc/apache2/sites-available/sklep-ssl.conf
```
```apache
<VirtualHost *:443>
    ServerName www.sklep.local
    ServerAlias sklep.local
    DocumentRoot /var/www/sklep

    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/sklep.crt
    SSLCertificateKeyFile /etc/ssl/private/sklep.key

    <Directory /var/www/sklep/admin>
        AuthType Basic
        AuthName "Panel sklepu"
        AuthUserFile /etc/apache2/.htpasswd
        Require user szef kierownik
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/sklep_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/sklep_ssl_access.log combined
</VirtualHost>
```
W `sklep.conf` (port 80) zamień zawartość na przekierowanie:
```apache
<VirtualHost *:80>
    ServerName www.sklep.local
    ServerAlias sklep.local
    Redirect permanent / https://www.sklep.local/
</VirtualHost>
```
```bash
sudo a2ensite sklep-ssl
sudo apache2ctl configtest && sudo systemctl restart apache2
curl -I http://www.sklep.local          # 301, Location: https://www.sklep.local/
curl -k https://www.sklep.local         # Sklep
```
</details>

<details>
<summary><b>Zadanie 8</b></summary>

```bash
sudo apt install -y php libapache2-mod-php
sudo nano /var/www/blog/index.php
```
```php
<?php
echo "<h1>Blog</h1>";
echo "<p>Data: " . date("Y-m-d H:i:s") . "</p>";
echo "<p>Twoje IP: " . $_SERVER['REMOTE_ADDR'] . "</p>";
?>
```
W `blog.conf` dopisz:
```apache
    DirectoryIndex index.php index.html
```
```bash
sudo apache2ctl configtest && sudo systemctl restart apache2
curl http://www.blog.local              # wynik PHP, nie "<h1>Blog</h1>" z index.html
```
</details>

<details>
<summary><b>Zadanie 9 – nginx (zadania 2–8)</b></summary>

**Wyłączenie Apache i instalacja nginx**
```bash
sudo systemctl disable --now apache2
sudo ss -tulpn | grep :80               # pusto
sudo apt install -y nginx php-fpm apache2-utils
sudo rm /etc/nginx/sites-enabled/default    # wyłączymy i zrobimy własną domyślną (userdir)
ls /run/php/                            # np. php8.4-fpm.sock
```

**Strona domyślna z userdir (zad. 4)** – `/etc/nginx/sites-available/default`:
```nginx
server {
    listen 80 default_server;
    root /var/www/html;
    index index.html;
    server_name _;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ ^/~([^/]+)(/.*)?$ {
        alias /home/$1/public_html$2;
        index index.html;
        autoindex on;
    }
}
```

**Sklep z HTTPS i hasłem (zad. 2, 5, 7)** – `/etc/nginx/sites-available/sklep`:
```nginx
server {
    listen 80;
    server_name www.sklep.local sklep.local;
    return 301 https://www.sklep.local$request_uri;
}

server {
    listen 443 ssl;
    server_name www.sklep.local sklep.local;
    ssl_certificate     /etc/ssl/certs/sklep.crt;
    ssl_certificate_key /etc/ssl/private/sklep.key;

    root /var/www/sklep;
    index index.html;
    access_log /var/log/nginx/sklep_access.log;
    error_log  /var/log/nginx/sklep_error.log;

    location /admin/ {
        auth_basic "Panel sklepu";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
```
```bash
sudo htpasswd -c /etc/nginx/.htpasswd szef
sudo htpasswd /etc/nginx/.htpasswd kierownik
```

**Blog z PHP (zad. 2, 8)** – `/etc/nginx/sites-available/blog`:
```nginx
server {
    listen 80;
    server_name www.blog.local;
    root /var/www/blog;
    index index.php index.html;
    access_log /var/log/nginx/blog_access.log;
    error_log  /var/log/nginx/blog_error.log;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }
}
```

**Intranet na 8080 z ograniczeniem IP (zad. 3, 6)** – `/etc/nginx/sites-available/intranet`:
```nginx
server {
    listen 8080;
    root /srv/intranet;
    index index.html;
    access_log /var/log/nginx/intranet_access.log;
    error_log  /var/log/nginx/intranet_error.log;

    location / {
        allow 192.168.1.0/24;
        allow 127.0.0.1;
        deny all;
    }
}
```

**Włączenie i test**
```bash
cd /etc/nginx/sites-enabled
sudo ln -s /etc/nginx/sites-available/default .
sudo ln -s /etc/nginx/sites-available/sklep .
sudo ln -s /etc/nginx/sites-available/blog .
sudo ln -s /etc/nginx/sites-available/intranet .
ls -l
sudo nginx -t && sudo systemctl reload nginx

curl -I http://www.sklep.local                          # 301
curl -k -I https://www.sklep.local/admin/               # 401
curl -k -u szef:haslo https://www.sklep.local/admin/    # Panel admina
curl http://www.blog.local                              # wynik PHP
curl http://localhost:8080                              # Intranet
curl http://localhost/~adam/                            # Strona w budowie
curl -I http://localhost | grep Server                  # Server: nginx
```
> W `cd /etc/nginx/sites-enabled` + `ln -s /pełna/ścieżka .` kropka oznacza „utwórz link tutaj, pod tą samą nazwą”.
</details>

<details>
<summary><b>Zadanie 10</b></summary>

```bash
cd /var/log
python3 -m http.server 8000
```
Z klienta: `http://192.168.1.100:8000`. Zatrzymanie: **Ctrl+C**.

Niektórych plików nie da się otworzyć, bo serwer działa z uprawnieniami **Twojego użytkownika**, a wiele logów może czytać tylko root (lub grupa `adm`). Uruchamianie takiego serwera jako root (`sudo python3 -m http.server`) to **zły pomysł** – udostępnisz każdemu w sieci poufne logi.
</details>

---

*Opracował: Bartosz Bryniarski*
