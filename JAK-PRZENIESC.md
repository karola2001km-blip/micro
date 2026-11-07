# Przeniesienie User i Cart do wspólnego repo

## Pliki do dodania

Paczka `paczki/user-cart-do-wspolnego-repo.zip` zawiera tylko `services/user-service/` i `services/cart-service/`. Po rozpakowaniu należy zachować te ścieżki. W istniejących katalogach zmienią się ich README i dojdą pliki aplikacji. Pozostałe usługi i wspólne pliki nie są częścią paczki.

## Dodanie przez GitHub

1. Pobierz ZIP i rozpakuj go na komputerze. Na stronie pliku ZIP w GitHub służy do tego przycisk „Download raw file” ze strzałką w dół.
2. Otwórz wspólne repo i przejdź do jego katalogu głównego, klikając nazwę `sklep-mikroserwisy`. Nie dodawaj plików wewnątrz `docs`.
3. Wybierz `develop` i utwórz z niego gałąź `feature/user-cart`. Jeśli taka gałąź już zawiera wcześniejszą wersję tych serwisów, sprawdź ją przed kolejnym dodaniem.
4. Na nowej gałęzi wybierz **Add file → Upload files**.
5. Przeciągnij folder **services** z rozpakowanej paczki. Nie przesyłaj samego ZIP-a, całego folderu nadrzędnego ani pojedynczego `app/`.
6. Sprawdź, czy lista zmian dotyczy wyłącznie `services/user-service/...` i `services/cart-service/...`.
7. Zapisz commit: `Dodanie User i Cart Service w Pythonie`.
8. Utwórz Pull Request: **base: develop**, **compare: feature/user-cart**. Poproś Michała o przegląd. Połączenie PR powinno odbyć się zgodnie z zasadami zespołu.

Główne README i CODEOWNERS pozostają w wersji zespołu. Zmiany wspólnej konfiguracji należy uzgodnić z Michałem.

## Opis Pull Request do skopiowania

```markdown
## Co zmieniam

Dodaję User Service i Cart Service w Pythonie 3.12 / FastAPI. User obsługuje profile i adresy w PostgreSQL, a Cart koszyk w Redisie oraz pobieranie ceny i dostępności przez REST. Manifesty, kolekcje Postman i dokumentacja znajdują się w katalogach usług zgodnie ze strukturą repozytorium.

## Której usługi dotyczy

- services/user-service
- services/cart-service

## Jak sprawdziłem

- [x] Usługa się uruchamia lokalnie przez Docker Compose
- [x] /health/live i /health/ready odpowiadają 200
- [x] Kolekcja Postman usługi przechodzi — Newman, 12 żądań i asercji bez błędów
- [x] Brak haseł i kluczy w zmianie — .env.example bez wartości

Dodatkowo: 49 testów pytest, budowanie obu obrazów i rejestracja w Consulu. Product i Inventory w weryfikacji Cart były atrapami HTTP. Docelowy Kubernetes i integracja z usługami zespołu wymagają osobnego sprawdzenia.

## Do uzgodnienia

Wpisy technologii i właścicieli w głównym README/CODEOWNERS, nazwy baz i sekretów, NetworkPolicy oraz kontrakty Auth/Product/Inventory/Order.
```

## Lokalne uruchomienie po pobraniu wspólnego repo

Z katalogu głównego:

```sh
cp services/user-service/.env.example services/user-service/.env
# W services/user-service/.env ustaw DB_PASSWORD.
docker compose --env-file services/user-service/.env -f services/user-service/dev/compose.yaml up --build -d
```

W PowerShell pierwsze polecenie: `Copy-Item services/user-service/.env.example services/user-service/.env`.

Dokumentacja API: `/docs` na portach 8082 i 8085. Dostępne są profile, adresy i pusty koszyk. Dodanie produktu wymaga podłączonego Product albo uruchomienia testów z jego atrapą. Nie należy wpisywać cen ani produktów na stałe w kodzie Cart.
