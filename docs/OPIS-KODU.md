# Struktura i działanie serwisów

Oba serwisy używają Pythona 3.12 i FastAPI. Każdy można budować i uruchamiać oddzielnie; żaden nie importuje kodu drugiego ani nie zagląda do jego bazy. Moduły `security.py`, `http_support.py` i `discovery.py` występują w obu usługach, dzięki czemu obrazy nie wymagają wspólnego pakietu. Zmiany w tych modułach wymagają aktualizacji obu kopii.

## Podział odpowiedzialności

**Auth** przechowuje hasło i wydaje JWT. **Gateway** weryfikuje JWT i przekazuje tożsamość. Te funkcje należą do pozostałych komponentów sklepu. **User** zapisuje profil i adresy. **Cart** zapisuje wybrane produkty i ilości oraz pobiera cenę i stan. **Order** składa zamówienie, rezerwuje towar i po zakupie czyści koszyk.

Logowanie pozostaje w Auth, a rezerwacja towaru w Order. User i Cart nie korzystają z RabbitMQ. Zdarzenie `user.registered` publikuje Auth, natomiast zdarzenia zamówień publikuje Order.

## User Service — obsługa żądania

Przepływ dla `PUT /users/me`:

1. Uvicorn uruchamia obiekt `app` z `app/main.py`. Funkcja `create_app()` składa aplikację, bazę, router i obsługę błędów.
2. Middleware w `http_support.py` odczytuje lub tworzy `X-Request-Id`. Dzięki temu można połączyć logi tego samego żądania w różnych usługach.
3. FastAPI dopasowuje metodę w `routes.py`. Pydantic sprawdza JSON według `UpdateUser` z `schemas.py`.
4. Zależność `current_user()` z `security.py` odczytuje `X-User-Id` i role. ID właściciela nie pochodzi z JSON, więc użytkownik nie wybiera sobie cudzego profilu.
5. `session_dependency()` tworzy sesję SQLAlchemy na czas żądania i przekazuje ją do funkcji endpointu przez `Depends`.
6. `ProfileService.update()` rozpoczyna transakcję, pobiera profil przez repozytorium i ustawia imię i nazwisko.
7. `with session.begin()` zatwierdza zmianę po poprawnym zakończeniu; wyjątek powoduje wycofanie. SQLAlchemy generuje parametryzowane zapytanie SQL.
8. `UserView` jest serializowany do JSON. Pythonowe `first_name` wraca jako `firstName` dzięki aliasom Pydantic.

### Pliki User Service

Wszystkie ścieżki w tabeli są względem `services/user-service/`.

| Plik | Odpowiedzialność |
| --- | --- |
| `app/main.py` | Składa FastAPI, sesje bazy, rejestrację w Consulu, endpointy zdrowia i mapowanie błędów SQLAlchemy. |
| `app/config.py` | Odczytuje zmienne środowiskowe. Buduje URL PostgreSQL z bezpiecznym kodowaniem hasła. |
| `app/database.py` | Tworzy silnik SQLAlchemy, pulę połączeń i sesję na czas żądania. |
| `app/models.py` | Encje `UserProfile` i `Address`: opis tabel, kolumn, UUID, klucza obcego i unikalnego e-maila. |
| `app/schemas.py` | DTO Pydantic: pola wejścia/wyjścia, długości, e-mail, kraj i nazwy camelCase. |
| `app/repository.py` | Zapytania do bazy. `owned_address()` szuka jednocześnie po ID adresu i ID właściciela. |
| `app/service.py` | Reguły biznesowe, transakcje, tworzenie i edycja profilu oraz operacje adresów. |
| `app/routes.py` | Ścieżki HTTP, zależności `Depends`, wymagane uprawnienia, modele odpowiedzi i statusy 201/204. |
| `app/security.py` | Wymusza tożsamość oraz role CUSTOMER, ADMIN i SERVICE. Nie weryfikuje JWT. |
| `app/http_support.py` | Request ID, nagłówki odpowiedzi, logi JSON, jednolite błędy i dokumentacja nagłówków w OpenAPI. |
| `app/discovery.py` | Rejestruje i wyrejestrowuje instancję w Consulu; wspólny klient discovery. User nie wywołuje innych usług biznesowych. |
| `alembic.ini`, `migrations/env.py` | Konfiguracja narzędzia migracji; korzysta z tego samego adresu bazy co aplikacja. |
| `migrations/versions/0001_profiles.py` | Pierwsza migracja: tabele profili i adresów, UNIQUE e-maila, klucz obcy i indeks. |
| `migrations/script.py.mako` | Szablon następnych migracji Alembic. |
| `tests/conftest.py` | Izolowana baza testowa, zastosowanie migracji oraz klient HTTP testowanej aplikacji. |
| `tests/test_users.py` | Testy API profilu, walidacji, uprawnień, właściciela adresu, konfliktów i zdrowia. |

### Ważne decyzje w User

Profil korzysta z UUID konta nadanego przez Auth. Nowe adresy mają własne UUID. E-maila nie można zmienić przez `PUT /users/me`, ponieważ jest również loginem w Auth, a architektura nie określa przepływu jego synchronizacji.

`models.py` opisuje bazę, a `schemas.py` opisuje API. To rozdzielenie chroni przed przypadkowym ujawnieniem wewnętrznych pól i pozwala zmieniać sposób zapisu bez zmiany kontraktu HTTP. `extra="forbid"` odrzuca dodatkowe pola, np. próbę dopisania sobie roli ADMIN.

Alembic tworzy schemat osobnym poleceniem `alembic upgrade head`; aplikacja nie tworzy automatycznie tabel przy każdym starcie. Kolejne zmiany schematu są zapisywane jako nowe migracje. Zastosowana migracja `0001` pozostaje niezmienna.

## Cart Service — dodanie pozycji

1. `routes.py` przyjmuje `POST /cart/items`. Pydantic sprawdza UUID oraz całkowitą ilość 1–999. Odrzuca `true`, ułamki i cenę podaną przez klienta.
2. `security.py` ustala właściciela na podstawie nagłówka z Gateway.
3. `CartService.change()` woła `CatalogClient.product()` i `available()`.
4. `discovery.py` pobiera zdrowe instancje z Consula. Listę zapamiętuje na 10 sekund i wybiera kolejne instancje round robin.
5. `catalog.py` wykonuje GET z timeoutem, ponowieniami i przekazaniem kontekstu. Do ról dodaje SERVICE. Nazwa i cena muszą mieć poprawny format.
6. Serwis odczytuje obecną pozycję. POST dodaje ilość, PATCH zastępuje ją. Porównuje docelową ilość z dostępnym stanem.
7. `store.py` wykonuje w Redisie skrypt Lua: porównuje poprzednią wartość, zapisuje nową i ustawia TTL atomowo.
8. Jeśli inny zapis zdążył zmienić pozycję, serwis odczytuje ją jeszcze raz i ponownie oblicza ilość. Po ośmiu konfliktach zwraca 409.
9. Odpowiedź zawiera aktualny koszyk, sumę oraz `inventoryChecked`, czyli informację, czy udało się sprawdzić stan magazynowy.

### Pliki Cart Service

Ścieżki względem `services/cart-service/`.

| Plik | Odpowiedzialność |
| --- | --- |
| `app/main.py` | Tworzy klientów Redis/httpx, discovery i serwis; zamyka zasoby przy wyłączeniu; health checki. |
| `app/config.py` | Port, Redis, Consul, identyfikator instancji i opcjonalne lokalne adresy zależności. |
| `app/schemas.py` | Modele żądań, pozycji i koszyka; aliasy JSON; funkcje przeliczania kwot. |
| `app/routes.py` | Endpointy koszyka. Wewnętrzne czyszczenie wymaga SERVICE. |
| `app/service.py` | Ilości, pobranie ceny, dostępność, sumy i rozwiązywanie konfliktów równoległych zmian. |
| `app/store.py` | Wyłącznie Redis: hash `cart:{userId}`, odczyt, atomowy zapis CAS, usuwanie, TTL i ping. |
| `app/catalog.py` | GET do Product i Inventory, walidacja odpowiedzi, retry oraz określone fallbacki. |
| `app/circuit_breaker.py` | Liczy wyniki ostatnich 10 wywołań, otwiera obwód przy 50% błędów i po 30 s pozwala na jedną próbę powrotu. |
| `app/discovery.py` | Rejestracja instancji, cache zdrowych adresów i round robin. |
| `app/security.py`, `app/http_support.py` | Te same zasady nagłówków, tożsamości, błędów i logowania co w User. |
| `tests/conftest.py` | Prawdziwy Redis w bazie nr 15 oraz atrapy HTTP Consula, Product i Inventory. |
| `tests/test_cart.py` | Testy API, cen, ilości, TTL, uprawnień, awarii i współbieżności. |
| `tests/test_resilience.py` | Testy circuit breakera, powrotu po awarii, pojedynczej próby half-open i discovery. |

### Obsługa operacji wejścia i wyjścia

Cart często czeka na sieć: Redis i usługi HTTP. Podczas `await` Python może obsługiwać inne żądania zamiast blokować cały proces. User korzysta z synchronicznego SQLAlchemy i endpointów `def`, które FastAPI wykonuje w puli wątków. Zapytania synchroniczne nie blokują dzięki temu pętli zdarzeń.

### Współbieżne zmiany koszyka

Oddzielny odczyt i zapis ilości mogą powodować utratę równoległych zmian. Na przykład dwa procesy mogą odczytać ilość 1 i oba zapisać 2 zamiast 3. Lua wykonuje porównanie oraz zapis jednym atomowym krokiem w Redisie, także między różnymi replikami aplikacji. Blokada w samym Pythonie nie wystarczyłaby do ochrony kilku replik.

### Pieniądze i TTL

Kwoty są przeliczane przez `Decimal` na całkowitą liczbę groszy. Cena `"19.99"` to 1999 groszy, a trzy sztuki dają `"59.97"`. Obliczenia nie korzystają z `float`.

TTL wynosi 604800 sekund od ostatniej zmiany. POST, PATCH i skuteczne usunięcie pozycji go odświeżają. GET nie odnawia ważności. Po usunięciu ostatniej pozycji Redis usuwa pusty hash; API zwraca wtedy pustą listę i `"0.00"`.

### Awaria zależności

| Sytuacja | Zachowanie |
| --- | --- |
| Product nie działa lub podaje niepoprawną cenę | 503, bez zapisu pozycji. |
| Produkt nie istnieje | 404, bez ponawiania. |
| Inventory potwierdza zbyt mały stan | 409. |
| Inventory nie działa | Dodanie dozwolone, `inventoryChecked:false`; rezerwację wykona Order. |
| Redis nie działa | 503, operacja zgłaszana jako nieudana. |

Circuit breaker jest osobny dla Product i Inventory. Liczy wynik całego wywołania wraz z ponowieniami. 404 jest wynikiem biznesowym, więc nie liczy się jako awaria. Ponowienia obejmują wyłącznie GET, a nie operacje zapisu.

## Zależności i wdrożenie

Każda usługa ma `requirements.in` z zależnościami bezpośrednimi oraz `requirements.lock` z dokładnymi wersjami i hashami. Wariant `requirements-dev.lock` dodaje pytest. Polecenie `pip install --require-hashes` weryfikuje pobrane pliki.

Każdy `Dockerfile` uruchamia aplikację jako użytkownik bez roota. `compose.yaml` dostarcza lokalną infrastrukturę, osobny krok migracji i obie aplikacje. `deploy/k8s/` zawiera Deploymenty, ClusterIP, sondy, ograniczenia zasobów, NetworkPolicy oraz Job migracyjny. Infrastruktura i sekrety w klastrze są wspólne i wymagają uzgodnienia z architektem.

`postman/` zawiera kolekcje do lokalnych demonstracji API. Nie zastępują testu integracji z rzeczywistym Gateway i serwisami pozostałych osób.
