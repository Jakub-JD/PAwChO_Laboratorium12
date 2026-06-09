# Sprawozdanie z Laboratorium 12
**Przedmiot:** Programowanie Aplikacji w Chmurze Obliczeniowej 

**Autor:** Jakub Fus

## Cel zadania
Celem zadania obowiązkowego było uruchomienie trzech kontenerów z serwerem Nginx (o nazwach `web1`, `web2`, `web3`), podłączenie ich do wspólnej, definiowanej przez użytkownika sieci mostkowej oraz odpowiednie zmapowanie wolumenów typu *bind mount*. Wolumeny miały służyć do udostępnienia kontenerom statycznej strony HTML (w trybie tylko do odczytu) oraz do zapisu logów z każdego serwera do dedykowanych katalogów na systemie hosta.

---

## Krok 1: Przygotowanie środowiska i struktury katalogów

W pierwszej kolejności utworzyłem wymaganą strukturę katalogów oraz plik HTML, który będzie serwowany przez kontenery.

Użyte polecenia:
* `mkdir -p ~/lab12/web1_logs ~/lab12/web2_logs ~/lab12/web3_logs` - Tworzymy katalog lab12 wraz z podkatalogami do przechowywania logów kontenerów.
* `echo "<h1>Laboratorium 12</h1><p>Jakub Fus</p>" > ~/lab12/index.html` - towrzymy prosty html z nazwą Laboratorium oraz autorem (zapisujemy go w głównym folderze laboratorium)

<img width="1237" height="239" alt="Zrzut ekranu 2026-06-09 144949" src="https://github.com/user-attachments/assets/7cd1b0aa-d3a2-4a78-80f2-e914bdfd76a2" />

---

## Krok 2: Utworzenie sieci mostkowej

Następnie utworzyłem dedykowaną sieć dla kontenerów, co pozwala na ich odizolowanie oraz ewentualną komunikację między nimi po nazwach.

Użyte polecenie:
* `docker network create --driver bridge lab12net` - polecenie tworzy nową sieć definiowaną przez użytkownika. 
    * Atrybut `--driver bridge` wymusza użycie sterownika mostkowego, co jest standardem dla kontenerów działających na pojedynczym hoście. 
    * `lab12net` to wybrana nazwa dla tej sieci. Zwrócony ciąg znaków to unikalne ID nowej sieci.

<img width="923" height="165" alt="Zrzut ekranu 2026-06-09 145021" src="https://github.com/user-attachments/assets/5812db08-4bdd-495c-aaee-9e9a71a72df8" />

---

## Krok 3: Uruchomienie kontenerów Nginx

Kolejnym etapem było uruchomienie trzech serwerów WWW. Dla każdego z nich użyłem analogicznego zestawu parametrów, zmieniając jedynie nazwę, mapowany port oraz katalog docelowy na logi.

Rozbicie użytego polecenia `docker run` na atrybuty:
* `-d` (detach) - uruchamia kontener w tle, dzięki czemu terminal pozostaje do naszej dyspozycji.
* `--name web1` - nadaje kontenerowi nazwę.
* `--network lab12net` - podłącza kontener do utworzonej w poprzednim kroku sieci.
* `-p 8081:80` - przekierowuje ruch z portu `8081` na komputerze hosta na domyślny port `80` wewnątrz kontenera.
* `--mount type=bind,source=$HOME/lab12/index.html,target=/usr/share/nginx/html/index.html,readonly` - podłącza wolumen typu bind. Wskazuje lokalny plik z system hosta (`source`) i umieszcza go w domyślnym katalogu Nginx wewnątrz kontenera (`target`). Atrybut `readonly` zabezpiecza plik przed modyfikacją ze strony kontenera.
* `--mount type=bind,source=$HOME/lab12/web1_logs,target=/var/log/nginx` - mapuje katalog logów. Zawartość generowana przez serwer w ścieżce `/var/log/nginx` jest automatycznie zapisywana na dysku twardym komputera w dedykowanym katalogu.
* `nginx:latest` - określa obraz bazowy, na którym opiera się kontener.

Czynność powtórzyłem dla kontenerów `web2` i `web3` (zmieniając odpowiednio porty na `8082`, `8083` oraz ścieżki do logów).

<img width="1429" height="688" alt="Zrzut ekranu 2026-06-09 145254" src="https://github.com/user-attachments/assets/7dd9c4ae-50f4-4004-b949-ae00bb74083a" />

---

## Krok 4: Weryfikacja działania serwerów (Dostęp do strony HTML)

Aby sprawdzić, czy wszystkie kontenery działają poprawnie i czy dostęp do nich z zewnątrz jest możliwy, zweryfikowałem ich status oraz wysłałem do nich zapytania HTTP.

Użyte polecenia:
* `docker ps` - polecenie wyświetliło listę aktywnych kontenerów, potwierdzając, że web1, web2 i web3 są uruchomione ("Up") i mają poprawnie przydzielone porty.
* `curl localhost:8081` (oraz dla portów 8082 i 8083) - wysłanie żądania pobrania strony ze wskazanych portów. W odpowiedzi otrzymałem zdefiniowany wcześniej kod HTML (`<h1>Laboratorium 12</h1><p>Jakub Fus</p>`). 

Dowodzi to, że serwery poprawnie korzystają z podłączonego pliku (w trybie read-only) oraz ruch sieciowy dociera do kontenerów przez zdefiniowane porty.

<img width="2028" height="361" alt="Zrzut ekranu 2026-06-09 145400" src="https://github.com/user-attachments/assets/e00655a4-0b6f-4e23-8c8e-1cedfb52fc50" />

---

## Krok 5: Weryfikacja zapisu logów na systemie macierzystym

Ostatnim etapem zadania było udowodnienie, że logi serwera zapisują się bezpośrednio na maszynie hosta. Ponieważ w Kroku 4 odpytaliśmy serwery narzędziem `curl`, ruch ten powinien zostać odnotowany w logach dostępowych (`access.log`).

Użyte polecenie:
* `cat ~/lab12/web1_logs/access.log` (oraz dla pozostałych katalogów `web2_logs` i `web3_logs`) - wypisuje w terminalu zawartość pliku z logami `access.log` zapisanego dzięki mapowaniu na komputerze.

Wyniki w terminalu pokazały prawidłowe wpisy o ruchu HTTP (zawierające znacznik czasowy oraz informację o programie klienckim `curl/8.5.0`). Dowodzi to, że podłączenie wolumenów zadziałało prawidłowo i Nginx zapisuje swoje logi bezpośrednio w systemie plików używanego komputera.

<img width="1275" height="343" alt="Zrzut ekranu 2026-06-09 145555" src="https://github.com/user-attachments/assets/d5cf096b-3156-471d-a04c-9072ac516532" />

---
**Wniosek końcowy:** Wszystkie wymagania z instrukcji do laboratorium zostały pomyślnie zrealizowane. Kontenery pracują w izolowanej sieci, serwują wyznaczoną treść w trybie read-only, a dane zdarzeń logowane są do wyznaczonych folderów z użyciem wolumenów `bind mount`.
