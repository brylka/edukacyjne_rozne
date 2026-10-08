# Debian – pierwsze kroki w trybie tekstowym

**Dla kogo:** uczniowie technikum (kwalifikacja INF.02)
**Założenie:** Debian (najnowsza wersja) jest już zainstalowany w VirtualBox, bez środowiska graficznego.
**Cel:** zalogować się, nie zgubić w systemie i opanować polecenia, które przydają się na egzaminie.

---

## Spis treści

1. [CZĘŚĆ 1: Pierwsze logowanie](#część-1-pierwsze-logowanie)
2. [CZĘŚĆ 2: Zwykły użytkownik, root, su i sudo](#część-2-zwykły-użytkownik-root-su-i-sudo)
3. [CZĘŚĆ 3: Klawisze, które oszczędzają czas](#część-3-klawisze-które-oszczędzają-czas)
4. [CZĘŚĆ 4: Pomoc – man i --help](#część-4-pomoc--man-i---help)
5. [CZĘŚĆ 5: Struktura katalogów i nawigacja](#część-5-struktura-katalogów-i-nawigacja)
6. [CZĘŚĆ 6: Pliki i katalogi – tworzenie, kopiowanie, usuwanie](#część-6-pliki-i-katalogi--tworzenie-kopiowanie-usuwanie)
7. [CZĘŚĆ 7: Przeglądanie i edycja plików (nano)](#część-7-przeglądanie-i-edycja-plików-nano)
8. [CZĘŚĆ 8: Wyszukiwanie, potoki i przekierowania](#część-8-wyszukiwanie-potoki-i-przekierowania)
9. [CZĘŚĆ 9: Użytkownicy i grupy](#część-9-użytkownicy-i-grupy)
10. [CZĘŚĆ 10: Uprawnienia – chmod i chown](#część-10-uprawnienia--chmod-i-chown)
11. [CZĘŚĆ 11: Instalowanie programów – apt](#część-11-instalowanie-programów--apt)
12. [CZĘŚĆ 12: Sieć – sprawdzanie i konfiguracja](#część-12-sieć--sprawdzanie-i-konfiguracja)
13. [CZĘŚĆ 13: Usługi, procesy, logi](#część-13-usługi-procesy-logi)
14. [CZĘŚĆ 14: Dyski, pamięć, archiwa](#część-14-dyski-pamięć-archiwa)
15. [CZĘŚĆ 15: Wyłączanie i restart](#część-15-wyłączanie-i-restart)
16. [Ściąga – najważniejsze polecenia](#ściąga--najważniejsze-polecenia)
17. [Zadania do samodzielnego wykonania](#zadania-do-samodzielnego-wykonania)
18. [Rozwiązania](#rozwiązania)

---

## CZĘŚĆ 1: Pierwsze logowanie

Po uruchomieniu maszyny wirtualnej zobaczysz czarny ekran z napisem podobnym do:

```
Debian GNU/Linux 13 debian tty1

debian login: _
```

- `debian` – to nazwa komputera (hostname) nadana podczas instalacji
- `tty1` – to numer konsoli (terminala), na której jesteś

### Krok po kroku

1. Wpisz **login** użytkownika (ten utworzony podczas instalacji, np. `uczen`) i naciśnij **Enter**.
2. Pojawi się `Password:` – wpisz hasło i naciśnij **Enter**.

> ⚠️ **Uwaga:** Podczas wpisywania hasła **NIC się nie wyświetla** – nie ma kropek ani gwiazdek. To normalne! Po prostu wpisz hasło i naciśnij Enter.

Po poprawnym zalogowaniu zobaczysz **znak zachęty** (prompt):

```
uczen@debian:~$
```

| Fragment | Znaczenie |
|----------|-----------|
| `uczen` | kto jest zalogowany |
| `@debian` | na jakim komputerze |
| `~` | w jakim katalogu jesteś (`~` = Twój katalog domowy) |
| `$` | zwykły użytkownik |
| `#` | **root** (administrator) – uważaj, co robisz! |

### Sprawdź, w co się zalogowałeś

```bash
whoami                  # kim jestem?
hostname                # jak nazywa się komputer?
cat /etc/os-release     # jaka wersja Debiana?
uname -r                # wersja jądra (kernela)
date                    # data i godzina
```

### Wylogowanie

```bash
exit                    # lub logout, lub Ctrl+D
```

### Kilka konsol naraz

Linux ma kilka konsol tekstowych (tty1–tty6). Na każdej możesz zalogować się osobno, np. na jednej jako zwykły użytkownik, na drugiej jako root.

- Na prawdziwym komputerze: **Ctrl+Alt+F1 … F6**
- W VirtualBox: **Prawy Ctrl + F1 … F6** (prawy Ctrl to tzw. klawisz *Host*)

> 💡 **Wskazówka:** Jeśli VirtualBox „złapał” Ci mysz lub klawiaturę – naciśnij **prawy Ctrl**, żeby ją uwolnić.

---

## CZĘŚĆ 2: Zwykły użytkownik, root, su i sudo

W Linuxie jest **root** – administrator, który może wszystko (także zepsuć system). Na co dzień pracujesz jako zwykły użytkownik, a uprawnień administratora używasz tylko wtedy, gdy trzeba.

### su – przełącz się na roota

```bash
su -                    # przełącz na roota (podajesz hasło ROOTA)
exit                    # wróć do swojego użytkownika
```

> ⚠️ **Uwaga – pułapka Debiana:** Zawsze pisz `su -` (z myślnikiem), a nie samo `su`.
> Samo `su` nie ustawia poprawnie zmiennej PATH i polecenia administracyjne (np. `adduser`, `usermod`, `reboot`) zwracają błąd **command not found**, mimo że jesteś rootem.

### sudo – wykonaj jedno polecenie jako root

```bash
sudo apt update         # wykonaj polecenie jako root (podajesz SWOJE hasło)
```

Jeśli przy instalacji podałeś hasło roota, to Debian **nie dodaje** Twojego użytkownika do grupy `sudo` i zobaczysz:

```
uczen is not in the sudoers file.
```

### Jak to naprawić (robimy raz)

```bash
su -                            # zostań rootem
apt install sudo                # zainstaluj sudo (jeśli go nie ma)
usermod -aG sudo uczen          # dodaj użytkownika uczen do grupy sudo
exit                            # wróć do użytkownika
```

Następnie **wyloguj się i zaloguj ponownie** (`exit`, potem logowanie) – nowa grupa działa dopiero po ponownym zalogowaniu.

```bash
groups                          # sprawdź – powinno być widać "sudo"
sudo whoami                     # powinno wypisać: root
```

> 💡 **Wskazówka:** W dalszej części materiału polecenia wymagające uprawnień administratora poprzedzone są `sudo`. Jeśli pracujesz jako root (`#`), po prostu pomijasz `sudo`.

---

## CZĘŚĆ 3: Klawisze, które oszczędzają czas

| Klawisz | Co robi |
|---------|---------|
| **TAB** | autouzupełnianie nazw poleceń, plików, katalogów |
| **TAB TAB** | pokaż wszystkie możliwości |
| **↑ / ↓** | poprzednie / następne polecenie z historii |
| **Ctrl+C** | przerwij działające polecenie |
| **Ctrl+L** | wyczyść ekran (to samo co `clear`) |
| **Ctrl+D** | wyloguj / zakończ wpisywanie |
| **Ctrl+R** | szukaj w historii poleceń (wpisz fragment) |
| **Ctrl+A / Ctrl+E** | skok na początek / koniec linii |

```bash
history                 # lista wpisanych poleceń
!25                     # wykonaj polecenie nr 25 z historii
!!                      # powtórz ostatnie polecenie
sudo !!                 # powtórz ostatnie polecenie z sudo (gdy zapomnisz)
clear                   # wyczyść ekran
```

> 💡 **Wskazówka:** **Używaj TAB-a ciągle!** Mniej pisania = mniej literówek. Jeśli TAB nic nie podpowiada – prawdopodobnie masz błąd w ścieżce.

> ⚠️ **Uwaga:** W Linuxie **wielkość liter MA ZNACZENIE!** `Plik.txt` i `plik.txt` to **DWA RÓŻNE** pliki.

---

## CZĘŚĆ 4: Pomoc – man i --help

Nie musisz pamiętać wszystkich opcji. Musisz wiedzieć, **gdzie je znaleźć**.

```bash
man ls                  # podręcznik polecenia ls
ls --help               # krótka pomoc
whatis ls               # jednozdaniowy opis polecenia
```

Poruszanie się w `man`:

| Klawisz | Działanie |
|---------|-----------|
| strzałki, PgUp, PgDn | przewijanie |
| `/słowo` | szukaj słowa (`n` – następne wystąpienie) |
| `q` | wyjście |

---

## CZĘŚĆ 5: Struktura katalogów i nawigacja

Linux **nie ma dysków C:, D:**. Jest jeden katalog główny `/` (root), a wszystko inne jest „pod nim”.

| Katalog | Co tam jest |
|---------|-------------|
| `/` | katalog główny |
| `/home` | katalogi domowe użytkowników (np. `/home/uczen`) |
| `/root` | katalog domowy roota |
| `/etc` | **pliki konfiguracyjne** (najważniejszy katalog dla admina!) |
| `/var` | dane zmienne: logi (`/var/log`), strony www (`/var/www`) |
| `/tmp` | pliki tymczasowe |
| `/usr` | programy i biblioteki |
| `/bin`, `/sbin` | podstawowe polecenia / polecenia administracyjne |
| `/dev` | urządzenia (dyski, np. `/dev/sda`) |
| `/media`, `/mnt` | punkty montowania (pendrive, płyty, dyski) |

### Ścieżki

- **bezwzględna** – zaczyna się od `/`, np. `/var/log/syslog`
- **względna** – liczona od miejsca, w którym jesteś, np. `dokumenty/plik.txt`

| Symbol | Znaczenie |
|--------|-----------|
| `.` | bieżący katalog |
| `..` | katalog wyżej (rodzic) |
| `~` | katalog domowy |

### pwd – gdzie jestem?

```bash
pwd                     # Print Working Directory, np. /home/uczen
```

### ls – co tu jest?

```bash
ls                      # lista plików
ls -l                   # szczegółowa lista (uprawnienia, właściciel, rozmiar, data)
ls -a                   # pokaż ukryte pliki (zaczynające się od kropki)
ls -la                  # połączenie obu
ls -lh                  # rozmiary w czytelnej postaci (K, M, G)
ls /etc                 # zawartość katalogu /etc
```

### cd – zmień katalog

```bash
cd /etc                 # przejdź do /etc
cd ..                   # katalog wyżej
cd ../..                # dwa katalogi wyżej
cd ~                    # do katalogu domowego
cd                      # to samo – bez argumentu wraca do domu
cd -                    # wróć do poprzedniego katalogu
```

### tree – drzewo katalogów (warto doinstalować)

```bash
sudo apt install tree
tree                    # drzewo katalogów od bieżącego miejsca
```

### Ćwiczenie: nawigacja

1. Sprawdź, gdzie jesteś: `pwd`
2. Przejdź do `/var`: `cd /var`
3. Zobacz zawartość: `ls -l`
4. Wejdź do katalogu `log`: `cd log`
5. Sprawdź: `pwd` (powinno być `/var/log`)
6. Wróć do domu: `cd`
7. Potwierdź: `pwd`

---

## CZĘŚĆ 6: Pliki i katalogi – tworzenie, kopiowanie, usuwanie

### Tworzenie

```bash
mkdir projekty                  # utwórz katalog
mkdir -p szkola/inf02/linux     # utwórz całą ścieżkę naraz
touch plik.txt                  # utwórz pusty plik
touch a.txt b.txt c.txt         # kilka plików naraz
```

### Kopiowanie i przenoszenie

```bash
cp plik.txt kopia.txt           # kopiuj plik
cp plik.txt /tmp/               # kopiuj do katalogu /tmp
cp -r projekty/ projekty_kopia/ # kopiuj katalog z zawartością (-r = rekurencyjnie)
mv plik.txt nowa_nazwa.txt      # zmień nazwę
mv nowa_nazwa.txt /tmp/         # przenieś
```

> 💡 **Wskazówka:** W Linuxie **nie ma osobnego polecenia do zmiany nazwy** – robi się to przez `mv`.

### Usuwanie

```bash
rm plik.txt                     # usuń plik
rm -i plik.txt                  # usuń, ale zapytaj o potwierdzenie
rmdir pusty_katalog             # usuń PUSTY katalog
rm -r katalog                   # usuń katalog z zawartością
rm -rf katalog                  # usuń bez pytania (force)
```

> ⚠️ **Uwaga:** W Linuxie **nie ma kosza!** Usunięty plik znika na zawsze.
> `rm -rf` jest niebezpieczne. **Nigdy** nie wykonuj `rm -rf /` ani `rm -rf *` bez sprawdzenia, gdzie jesteś (`pwd`).

### Dowiązania (linki)

```bash
ln -s /var/log/syslog log_systemowy    # dowiązanie symboliczne (skrót)
ls -l log_systemowy                    # widać strzałkę -> do oryginału
```

---

## CZĘŚĆ 7: Przeglądanie i edycja plików (nano)

### Przeglądanie

```bash
cat plik.txt                    # wyświetl cały plik
less /etc/passwd                # przeglądaj stronami (q = wyjście, / = szukaj)
head /etc/passwd                # pierwsze 10 linii
head -n 3 /etc/passwd           # pierwsze 3 linie
tail /etc/passwd                # ostatnie 10 linii
tail -n 5 /etc/passwd           # ostatnie 5 linii
sudo tail -f /var/log/auth.log  # śledź plik na żywo (Ctrl+C = koniec)
wc -l /etc/passwd               # policz linie w pliku
```

> 💡 **Wskazówka:** W konsoli Debiana nie da się przewinąć ekranu w górę (Shift+PgUp nie działa). Jeśli wynik jest długi – dopisz `| less`, np. `ls -l /etc | less`.

### Edytor nano

```bash
nano notatki.txt                     # otwórz/utwórz plik
sudo nano /etc/hosts                 # pliki systemowe edytujemy z sudo!
```

| Skrót | Działanie |
|-------|-----------|
| **Ctrl+O**, potem **Enter** | zapisz |
| **Ctrl+X** | wyjdź (zapyta o zapis: `Y` / `N`) |
| **Ctrl+W** | szukaj |
| **Ctrl+K** | wytnij linię |
| **Ctrl+U** | wklej |
| **Alt+U** | cofnij |

> ⚠️ **Uwaga:** Zanim zmienisz ważny plik konfiguracyjny – **zrób kopię**:
> ```bash
> sudo cp /etc/network/interfaces /etc/network/interfaces.bak
> ```
> Jak coś zepsujesz, przywracasz kopię i system działa jak wcześniej.

---

## CZĘŚĆ 8: Wyszukiwanie, potoki i przekierowania

### find – szukanie plików

```bash
find /etc -name "hosts"                 # szukaj pliku o nazwie hosts w /etc
find /home -name "*.txt"                # wszystkie pliki .txt w /home
find / -name "sshd_config" 2>/dev/null  # szukaj w całym systemie, ukryj błędy
find /var/log -size +1M                 # pliki większe niż 1 MB
find . -type d                          # tylko katalogi
```

### grep – szukanie tekstu w plikach

```bash
grep uczen /etc/passwd                  # linie zawierające "uczen"
grep -i error /var/log/syslog           # bez rozróżniania wielkości liter
grep -r "Listen" /etc/apache2/          # szukaj we wszystkich plikach katalogu
grep -v "^#" plik.conf                  # pokaż linie, które NIE są komentarzem
grep -n root /etc/passwd                # z numerami linii
```

### Potok | – wynik jednego polecenia idzie do drugiego

```bash
ls -l /etc | less                       # przeglądaj długą listę
cat /etc/passwd | grep bash             # tylko użytkownicy z powłoką bash
ps aux | grep ssh                       # czy działa proces ssh?
cat /etc/passwd | wc -l                 # ilu jest użytkowników (kont)?
```

### Przekierowania

```bash
echo "Ala ma kota" > plik.txt           # zapisz do pliku (NADPISUJE!)
echo "Druga linia" >> plik.txt          # dopisz na końcu pliku
ls -l /etc > lista.txt                  # zapisz wynik polecenia do pliku
ls /nieistnieje 2> bledy.txt            # zapisz komunikaty błędów do pliku
polecenie > /dev/null 2>&1              # wyrzuć wszystko (wynik i błędy)
```

> ⚠️ **Uwaga:** `>` **kasuje** poprzednią zawartość pliku. Jeśli chcesz dopisać – użyj `>>`.

---

## CZĘŚĆ 9: Użytkownicy i grupy

**Bardzo częsty temat na INF.02!**

### Ważne pliki

| Plik | Co zawiera |
|------|-----------|
| `/etc/passwd` | lista kont użytkowników |
| `/etc/shadow` | zaszyfrowane hasła (czyta tylko root) |
| `/etc/group` | lista grup i ich członków |

Linia z `/etc/passwd`:

```
uczen:x:1000:1000:Jan Kowalski,,,:/home/uczen:/bin/bash
```

| Pole | Znaczenie |
|------|-----------|
| `uczen` | login |
| `x` | hasło jest w `/etc/shadow` |
| `1000` | UID – numer użytkownika |
| `1000` | GID – numer grupy podstawowej |
| `Jan Kowalski,,,` | opis (imię, nazwisko itp.) |
| `/home/uczen` | katalog domowy |
| `/bin/bash` | powłoka (shell) |

### Informacje o użytkownikach

```bash
whoami                          # kim jestem
id                              # mój UID, GID i grupy
id jan                          # informacje o użytkowniku jan
groups                          # moje grupy
who                             # kto jest teraz zalogowany
last                            # historia logowań
```

### Zarządzanie użytkownikami

```bash
sudo adduser jan                        # utwórz użytkownika (interaktywnie, z katalogiem domowym)
sudo passwd jan                         # ustaw/zmień hasło użytkownika jan
passwd                                  # zmień SWOJE hasło
sudo deluser jan                        # usuń użytkownika
sudo deluser --remove-home jan          # usuń użytkownika razem z katalogiem domowym
```

Wersja „niskopoziomowa” (działa na każdym Linuxie – warto znać):

```bash
sudo useradd -m -s /bin/bash anna       # -m = utwórz katalog domowy, -s = powłoka
sudo passwd anna                        # useradd NIE ustawia hasła – trzeba osobno!
sudo userdel -r anna                    # usuń z katalogiem domowym
```

> 💡 **Wskazówka:** Na Debianie wygodniejsze jest `adduser` – sam pyta o hasło i tworzy katalog domowy. `useradd` bez `-m` **nie utworzy** katalogu domowego.

### Modyfikacja konta

```bash
sudo usermod -aG sudo jan               # dodaj jana do grupy sudo (-a = DOPISZ, nie zastępuj!)
sudo usermod -s /bin/bash jan           # zmień powłokę
sudo usermod -L jan                     # zablokuj konto
sudo usermod -U jan                     # odblokuj konto
sudo chage -l jan                       # informacje o ważności hasła
sudo chage -M 30 jan                    # hasło ważne max 30 dni
sudo chage -d 0 jan                     # wymuś zmianę hasła przy następnym logowaniu
```

> ⚠️ **Uwaga:** `usermod -G grupa` **bez** `-a` usuwa użytkownika ze wszystkich innych grup dodatkowych! Zawsze pisz `-aG`.

### Grupy

```bash
sudo addgroup informatycy               # utwórz grupę (lub: groupadd)
sudo adduser jan informatycy            # dodaj jana do grupy
sudo deluser jan informatycy            # usuń jana z grupy
sudo delgroup informatycy               # usuń grupę (lub: groupdel)
getent group informatycy                # kto jest w grupie
```

---

## CZĘŚĆ 10: Uprawnienia – chmod i chown

### Jak czytać `ls -l`

```
-rwxr-xr-- 1 jan informatycy 1234 paź  8 09:30 skrypt.sh
```

| Fragment | Znaczenie |
|----------|-----------|
| `-` | typ: `-` plik, `d` katalog, `l` dowiązanie |
| `rwx` | uprawnienia **właściciela** (user) |
| `r-x` | uprawnienia **grupy** (group) |
| `r--` | uprawnienia **pozostałych** (others) |
| `jan` | właściciel |
| `informatycy` | grupa |

| Litera | Plik | Katalog | Wartość |
|--------|------|---------|---------|
| `r` | odczyt | wyświetlanie zawartości | **4** |
| `w` | zapis | tworzenie/usuwanie plików w środku | **2** |
| `x` | wykonanie | wejście do katalogu (`cd`) | **1** |

### chmod – zapis liczbowy (na egzaminie najczęstszy!)

Dodajesz wartości: `r=4`, `w=2`, `x=1`.

| Liczba | Uprawnienia |
|--------|-------------|
| 7 | rwx (4+2+1) |
| 6 | rw- (4+2) |
| 5 | r-x (4+1) |
| 4 | r-- |
| 0 | --- |

```bash
chmod 755 skrypt.sh             # rwxr-xr-x – właściciel wszystko, reszta czyta i wykonuje
chmod 644 plik.txt              # rw-r--r-- – typowe dla zwykłych plików
chmod 700 prywatny/             # rwx------ – tylko właściciel
chmod 750 katalog/              # rwxr-x--- – właściciel wszystko, grupa czyta i wchodzi
chmod -R 770 projekt/           # rekurencyjnie dla całego katalogu
```

### chmod – zapis symboliczny

```bash
chmod u+x skrypt.sh             # dodaj właścicielowi prawo wykonania
chmod g-w plik.txt              # zabierz grupie prawo zapisu
chmod o-rwx plik.txt            # zabierz pozostałym wszystko
chmod a+r plik.txt              # daj wszystkim (all) prawo odczytu
chmod ug=rw,o= plik.txt         # właściciel i grupa rw, pozostali nic
```

### chown i chgrp – zmiana właściciela i grupy

```bash
sudo chown jan plik.txt                 # zmień właściciela na jan
sudo chown jan:informatycy plik.txt     # zmień właściciela i grupę
sudo chown -R jan:jan /home/jan/www     # rekurencyjnie
sudo chgrp informatycy plik.txt         # zmień tylko grupę
```

### Typowe zadanie egzaminacyjne

> Utwórz katalog `/dane/projekt`, właścicielem ma być `jan`, grupą `informatycy`. Właściciel i grupa mają pełne prawa, pozostali – żadnych.

```bash
sudo mkdir -p /dane/projekt
sudo chown jan:informatycy /dane/projekt
sudo chmod 770 /dane/projekt
ls -ld /dane/projekt            # -d = pokaż sam katalog, nie jego zawartość
```

---

## CZĘŚĆ 11: Instalowanie programów – apt

Debian instaluje programy z **repozytoriów** (serwerów z pakietami) za pomocą `apt`.

```bash
sudo apt update                 # odśwież listę pakietów (ZAWSZE na początku!)
sudo apt upgrade                # zaktualizuj zainstalowane pakiety
sudo apt install mc htop        # zainstaluj pakiety (można kilka naraz)
sudo apt install -y tree        # -y = nie pytaj o potwierdzenie
sudo apt remove mc              # odinstaluj (konfiguracja zostaje)
sudo apt purge mc               # odinstaluj razem z konfiguracją
sudo apt autoremove             # usuń niepotrzebne zależności
apt search apache               # szukaj pakietu
apt show apache2                # informacje o pakiecie
apt list --installed            # lista zainstalowanych pakietów
dpkg -l | grep ssh              # czy jakiś pakiet ssh jest zainstalowany?
```

> 💡 **Wskazówka:** `apt update` **nie aktualizuje** programów – tylko pobiera aktualną listę. Aktualizuje `apt upgrade`.

> 💡 **Wskazówka:** Polecam doinstalować na start: `sudo apt install mc htop tree curl` – `mc` (Midnight Commander) to menedżer plików w trybie tekstowym w stylu Total Commandera.

---

## CZĘŚĆ 12: Sieć – sprawdzanie i konfiguracja

**Konfiguracja sieci to pewniak na INF.02!**

### Sprawdzanie

```bash
ip a                            # adresy IP wszystkich interfejsów (skrót od ip addr)
ip r                            # tablica routingu – brama domyślna (default via ...)
cat /etc/resolv.conf            # serwery DNS
ping -c 4 8.8.8.8               # test łączności (4 pakiety)
ping -c 4 debian.org            # test łączności + DNS
ss -tulpn                       # otwarte porty i nasłuchujące usługi (sudo pokaże procesy)
hostname -I                     # same adresy IP
```

Przykładowy wynik `ip a`:

```
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    link/ether 08:00:27:ab:cd:ef brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.50/24 brd 192.168.1.255 scope global dynamic enp0s3
```

- `enp0s3` – **nazwa interfejsu** (w VirtualBox zwykle `enp0s3`, druga karta `enp0s8`)
- `08:00:27:ab:cd:ef` – adres MAC
- `192.168.1.50/24` – adres IP z maską (`/24` = 255.255.255.0)
- `dynamic` – adres przydzielony przez DHCP

> ⚠️ **Uwaga:** Nie przepisuj na ślepo `enp0s3` z materiałów – **zawsze sprawdź swoją nazwę interfejsu** poleceniem `ip a`.

### Stały (statyczny) adres IP – plik /etc/network/interfaces

Debian bez środowiska graficznego konfiguruje sieć w pliku `/etc/network/interfaces`.

```bash
sudo cp /etc/network/interfaces /etc/network/interfaces.bak   # kopia zapasowa!
sudo nano /etc/network/interfaces
```

Domyślnie (DHCP) wygląda mniej więcej tak:

```
auto lo
iface lo inet loopback

allow-hotplug enp0s3
iface enp0s3 inet dhcp
```

Zmieniamy na adres statyczny:

```
auto lo
iface lo inet loopback

auto enp0s3
iface enp0s3 inet static
    address 192.168.1.100/24
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 1.1.1.1
```

Zastosowanie zmian:

```bash
sudo systemctl restart networking       # przeładuj konfigurację sieci
ip a                                    # sprawdź, czy adres się zmienił
ip r                                    # sprawdź bramę
```

> 💡 **Wskazówka:** Wpis `dns-nameservers` działa tylko z pakietem `resolvconf`. Jeśli go nie masz – DNS wpisujesz bezpośrednio do `/etc/resolv.conf`:
> ```
> nameserver 8.8.8.8
> nameserver 1.1.1.1
> ```

> 💡 **Wskazówka:** Jeśli po restarcie sieci adres się nie zmienił – zrestartuj maszynę (`sudo reboot`). Jeśli coś zepsułeś – przywróć kopię: `sudo cp /etc/network/interfaces.bak /etc/network/interfaces`.

### Nazwa komputera

```bash
hostnamectl                             # informacje o systemie i nazwie
sudo hostnamectl set-hostname serwer01  # zmień nazwę komputera
sudo nano /etc/hosts                    # zmień też starą nazwę w linii 127.0.1.1
```

Plik `/etc/hosts` po zmianie:

```
127.0.0.1       localhost
127.0.1.1       serwer01
```

Nowa nazwa w znaku zachęty pojawi się po ponownym zalogowaniu.

---

## CZĘŚĆ 13: Usługi, procesy, logi

### systemctl – zarządzanie usługami

Na egzaminie będziesz instalować usługi (ssh, apache2, samba, vsftpd, isc-dhcp-server, bind9…). Każdą zarządzasz tak samo:

```bash
sudo systemctl status ssh               # stan usługi (q = wyjście)
sudo systemctl start ssh                # uruchom
sudo systemctl stop ssh                 # zatrzymaj
sudo systemctl restart ssh              # uruchom ponownie (po zmianie konfiguracji!)
sudo systemctl reload ssh               # przeładuj konfigurację bez zatrzymywania
sudo systemctl enable ssh               # uruchamiaj automatycznie przy starcie systemu
sudo systemctl disable ssh              # nie uruchamiaj automatycznie
systemctl is-active ssh                 # krótko: active / inactive
systemctl list-units --type=service     # lista usług
```

> 💡 **Wskazówka:** Schemat pracy z każdą usługą: **instalujesz → kopia konfiguracji → edytujesz plik w /etc → restart → status → test.** Jeśli `status` pokazuje na czerwono `failed` – masz błąd w pliku konfiguracyjnym.

### Procesy

```bash
ps aux                          # wszystkie procesy
ps aux | grep apache            # szukaj konkretnego procesu
top                             # procesy na żywo (q = wyjście)
htop                            # ładniejsza wersja top (trzeba doinstalować)
kill 1234                       # zakończ proces o numerze PID 1234
kill -9 1234                    # zabij proces na siłę
pkill nano                      # zakończ procesy po nazwie
```

### Logi

```bash
sudo journalctl -xe                     # ostatnie wpisy z wyjaśnieniami (świetne przy błędach)
sudo journalctl -u ssh                  # logi konkretnej usługi
sudo journalctl -u ssh -f               # śledź logi usługi na żywo
sudo journalctl -b                      # logi od ostatniego uruchomienia
ls /var/log                             # katalog z plikami logów
sudo tail -n 20 /var/log/apache2/error.log    # np. błędy Apache (gdy jest zainstalowany)
```

> ⚠️ **Uwaga:** W nowych Debianach może **nie być** pliku `/var/log/syslog` – logi systemowe zbiera `journald`. Używaj `journalctl`.

---

## CZĘŚĆ 14: Dyski, pamięć, archiwa

### Dyski i pamięć

```bash
lsblk                           # dyski i partycje w postaci drzewa
df -h                           # wolne miejsce na partycjach
du -sh /var/log                 # ile zajmuje katalog
du -sh *                        # ile zajmuje każdy element w bieżącym katalogu
free -h                         # pamięć RAM i swap
sudo fdisk -l                   # szczegóły dysków i partycji
uptime                          # jak długo system działa i obciążenie
```

### Archiwa – tar

```bash
tar -cvf archiwum.tar katalog/              # spakuj (bez kompresji)
tar -czvf archiwum.tar.gz katalog/          # spakuj z kompresją gzip
tar -tzvf archiwum.tar.gz                   # pokaż zawartość archiwum
tar -xzvf archiwum.tar.gz                   # rozpakuj w bieżącym katalogu
tar -xzvf archiwum.tar.gz -C /tmp/          # rozpakuj do /tmp
```

Jak zapamiętać opcje:

| Opcja | Znaczenie |
|-------|-----------|
| `c` | create – utwórz |
| `x` | extract – rozpakuj |
| `t` | list – pokaż zawartość |
| `z` | gzip – kompresja |
| `v` | verbose – pokazuj, co robisz |
| `f` | file – nazwa pliku archiwum (**zawsze na końcu opcji**) |

### Przykład – kopia zapasowa konfiguracji

```bash
sudo tar -czvf /root/backup_etc_$(date +%F).tar.gz /etc
ls -lh /root/
```

`$(date +%F)` wstawi do nazwy dzisiejszą datę, np. `backup_etc_2026-10-08.tar.gz`.

---

## CZĘŚĆ 15: Wyłączanie i restart

```bash
sudo poweroff                   # wyłącz komputer
sudo reboot                     # uruchom ponownie
sudo shutdown -h now            # wyłącz teraz
sudo shutdown -r +5             # restart za 5 minut
sudo shutdown -c                # anuluj zaplanowane wyłączenie
```

> ⚠️ **Uwaga:** Nie zamykaj maszyny wirtualnej „krzyżykiem” ani opcją *Wyłącz maszynę* w VirtualBox, jeśli nie musisz. To jak wyciągnięcie wtyczki z prądu – możesz uszkodzić pliki. Zawsze `sudo poweroff`.

---

## Ściąga – najważniejsze polecenia

| Kategoria | Polecenia |
|-----------|-----------|
| Kim/gdzie jestem | `whoami`, `hostname`, `pwd`, `id`, `cat /etc/os-release` |
| Nawigacja | `ls -la`, `cd`, `cd ..`, `cd ~`, `tree` |
| Pliki | `mkdir -p`, `touch`, `cp -r`, `mv`, `rm -r`, `ln -s` |
| Przeglądanie | `cat`, `less`, `head`, `tail -f`, `nano` |
| Szukanie | `find`, `grep`, `\|`, `>`, `>>` |
| Administrator | `su -`, `sudo` |
| Użytkownicy | `adduser`, `passwd`, `deluser`, `usermod -aG`, `chage`, `addgroup` |
| Uprawnienia | `chmod 755`, `chmod u+x`, `chown user:grupa`, `ls -l` |
| Pakiety | `apt update`, `apt install`, `apt remove`, `apt purge`, `apt search` |
| Sieć | `ip a`, `ip r`, `ping`, `ss -tulpn`, `/etc/network/interfaces`, `hostnamectl` |
| Usługi | `systemctl status/start/stop/restart/enable` |
| Logi | `journalctl -xe`, `journalctl -u usługa` |
| System | `df -h`, `du -sh`, `free -h`, `lsblk`, `ps aux`, `top`, `kill` |
| Archiwa | `tar -czvf`, `tar -xzvf` |
| Koniec pracy | `exit`, `sudo reboot`, `sudo poweroff` |

---

## Zadania do samodzielnego wykonania

**Zadanie 1 – Rozpoznanie systemu**
Zaloguj się. Sprawdź: nazwę zalogowanego użytkownika, nazwę komputera, wersję Debiana, wersję jądra oraz adres IP maszyny. Wyniki zapisz do pliku `~/system.txt`.

**Zadanie 2 – sudo**
Sprawdź, czy Twój użytkownik może używać `sudo`. Jeśli nie – dodaj go do odpowiedniej grupy i potwierdź, że działa.

**Zadanie 3 – Struktura katalogów**
W katalogu domowym utwórz strukturę:
```
szkola/
├── inf02/
│   ├── linux/
│   └── windows/
└── notatki/
```
W katalogu `linux` utwórz plik `komendy.txt` i wpisz do niego (bez edytora!) trzy polecenia, które dziś poznałeś – każde w osobnej linii.

**Zadanie 4 – Kopiowanie i usuwanie**
Skopiuj plik `/etc/hosts` do katalogu `~/szkola/notatki/` pod nazwą `hosts_kopia`. Zmień nazwę katalogu `windows` na `win`. Następnie usuń katalog `win`.

**Zadanie 5 – Szukanie**
- Znajdź w `/etc` wszystkie pliki z rozszerzeniem `.conf` i policz, ile ich jest.
- Wyświetl z `/etc/passwd` tylko linię dotyczącą Twojego użytkownika.

**Zadanie 6 – Użytkownicy i grupy**
- Utwórz grupę `technikum`.
- Utwórz użytkowników `adam` i `ewa` z katalogami domowymi i hasłami.
- Dodaj oboje do grupy `technikum`.
- Wymuś na użytkowniku `ewa` zmianę hasła przy pierwszym logowaniu.
- Przełącz się na konsolę tty2 i zaloguj jako `adam`.

**Zadanie 7 – Uprawnienia**
Utwórz katalog `/projekty/technikum`. Właścicielem ma być `adam`, grupą `technikum`. Właściciel – pełne prawa, grupa – odczyt i wejście do katalogu, pozostali – brak praw. Sprawdź wynik.

**Zadanie 8 – Pakiety i usługa**
Zaktualizuj listę pakietów, zainstaluj `htop` i serwer `openssh-server`. Sprawdź, czy usługa ssh działa i czy uruchamia się automatycznie przy starcie systemu. Sprawdź, na jakim porcie nasłuchuje.

**Zadanie 9 – Sieć**
Ustaw maszynie stały adres IP (dobierz go do swojej sieci – zapytaj nauczyciela o adresację), bramę i DNS. Zmień nazwę komputera na `srv-nazwisko`. Sprawdź łączność z internetem poleceniem `ping`.

**Zadanie 10 – Kopia zapasowa**
Utwórz skompresowane archiwum katalogu `~/szkola` o nazwie `szkola_backup.tar.gz` w katalogu `/tmp`. Wyświetl zawartość archiwum bez rozpakowywania. Sprawdź, ile miejsca zajmuje archiwum i ile wolnego miejsca jest na dysku.

---

## Rozwiązania

<details>
<summary><b>Zadanie 1</b></summary>

```bash
whoami > ~/system.txt
hostname >> ~/system.txt
cat /etc/os-release >> ~/system.txt
uname -r >> ~/system.txt
ip a >> ~/system.txt
cat ~/system.txt
```
</details>

<details>
<summary><b>Zadanie 2</b></summary>

```bash
sudo whoami                 # jeśli błąd "not in the sudoers file":
su -
apt install sudo
usermod -aG sudo uczen
exit
exit                        # wyloguj się i zaloguj ponownie
groups                      # powinno być "sudo"
sudo whoami                 # wynik: root
```
</details>

<details>
<summary><b>Zadanie 3</b></summary>

```bash
cd ~
mkdir -p szkola/inf02/linux szkola/inf02/windows szkola/notatki
echo "pwd" > szkola/inf02/linux/komendy.txt
echo "ls -la" >> szkola/inf02/linux/komendy.txt
echo "cd .." >> szkola/inf02/linux/komendy.txt
cat szkola/inf02/linux/komendy.txt
tree szkola                 # jeśli zainstalowano tree
```
</details>

<details>
<summary><b>Zadanie 4</b></summary>

```bash
cp /etc/hosts ~/szkola/notatki/hosts_kopia
mv ~/szkola/inf02/windows ~/szkola/inf02/win
rmdir ~/szkola/inf02/win    # katalog jest pusty, więc wystarczy rmdir
ls ~/szkola/inf02
```
</details>

<details>
<summary><b>Zadanie 5</b></summary>

```bash
find /etc -name "*.conf" 2>/dev/null | wc -l
grep uczen /etc/passwd
```
</details>

<details>
<summary><b>Zadanie 6</b></summary>

```bash
sudo addgroup technikum
sudo adduser adam
sudo adduser ewa
sudo usermod -aG technikum adam
sudo usermod -aG technikum ewa
sudo chage -d 0 ewa
getent group technikum      # sprawdzenie: technikum:x:1003:adam,ewa
```
Następnie w VirtualBox: **prawy Ctrl + F2**, logowanie jako `adam`. Powrót: **prawy Ctrl + F1**.
</details>

<details>
<summary><b>Zadanie 7</b></summary>

```bash
sudo mkdir -p /projekty/technikum
sudo chown adam:technikum /projekty/technikum
sudo chmod 750 /projekty/technikum
ls -ld /projekty/technikum
# wynik: drwxr-x--- 2 adam technikum 4096 ... /projekty/technikum
```
</details>

<details>
<summary><b>Zadanie 8</b></summary>

```bash
sudo apt update
sudo apt install -y htop openssh-server
sudo systemctl status ssh           # szukaj: active (running)
systemctl is-enabled ssh            # wynik: enabled
sudo ss -tulpn | grep ssh           # nasłuchuje na porcie 22
```
</details>

<details>
<summary><b>Zadanie 9</b></summary>

```bash
ip a                                            # sprawdź nazwę interfejsu
sudo cp /etc/network/interfaces /etc/network/interfaces.bak
sudo nano /etc/network/interfaces
```
Przykładowa zawartość (dostosuj adresy i nazwę interfejsu):
```
auto lo
iface lo inet loopback

auto enp0s3
iface enp0s3 inet static
    address 192.168.1.100/24
    gateway 192.168.1.1
```
```bash
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
sudo systemctl restart networking
ip a
ip r
sudo hostnamectl set-hostname srv-kowalski
sudo nano /etc/hosts                            # zmień nazwę w linii 127.0.1.1
ping -c 4 debian.org
```
> `sudo echo ... > plik` **nie zadziała** – przekierowanie wykonuje się bez uprawnień roota. Dlatego używamy `| sudo tee plik`.
</details>

<details>
<summary><b>Zadanie 10</b></summary>

```bash
tar -czvf /tmp/szkola_backup.tar.gz ~/szkola
tar -tzvf /tmp/szkola_backup.tar.gz
ls -lh /tmp/szkola_backup.tar.gz
df -h
```
</details>

---

*Opracował: Bartosz Bryniarski*
