# Jak opowiedzieć o projekcie na zajęciach

Ten dokument jest krótkim przewodnikiem do prezentacji. Szczegółowe odpowiedzialności wszystkich plików znajdziesz w [OPIS-KODU.md](OPIS-KODU.md), a instrukcję pracy w [URUCHOMIENIE.md](URUCHOMIENIE.md).

## 1. Co obejmuje Twoja część

Twoja część to dwa osobne programy serwerowe napisane w Pythonie 3.12 i FastAPI:

- **User Service** przechowuje profil klienta i jego adresy dostawy w PostgreSQL.
- **Cart Service** przechowuje zawartość koszyka w Redisie, pobiera cenę z Product i sprawdza dostępność w Inventory.

Każdy program ma własny proces, port, konfigurację, testy i obraz Docker. Mogą być uruchamiane i rozwijane niezależnie. To podstawowa cecha architektury mikrousługowej. Korzystanie z jednego języka nie łączy ich w jedną usługę.

## 2. Krótka wypowiedź na około dwie minuty

> Moja część projektu obejmuje User Service i Cart Service. Oba zostały przygotowane w Pythonie z FastAPI. Zachowują uzgodnione endpointy REST, dzięki czemu pozostałe usługi nie muszą używać tego samego języka.
>
> User Service odpowiada za dane profilu: imię, nazwisko, e-mail i adresy dostawy. Przechowuje je w PostgreSQL. Dane wejściowe sprawdza Pydantic, operacje na bazie wykonuje SQLAlchemy, a schemat bazy wersjonuje Alembic. Hasła i logowanie należą do oddzielnego Auth Service.
>
> Cart Service przechowuje pozycje koszyka w Redisie. Dodanie produktu zwiększa ilość, a PATCH ustawia konkretną ilość. Serwis pobiera nazwę i cenę z Product oraz sprawdza stan w Inventory. Cena nie pochodzi od klienta. Kwoty są obliczane w groszach z użyciem Decimal. Koszyk wygasa po siedmiu dniach bez zmian.
>
> Obie usługi korzystają z Consula, który przechowuje adresy działających instancji. Cart ma timeouty, ograniczone ponowienia zapytań GET i circuit breaker. Jeśli Inventory nie działa, można dodać produkt ze znacznikiem braku potwierdzenia stanu; ostatecznej rezerwacji dokona Order. Jeżeli Product nie dostarczy ceny, dodanie jest odrzucone.
>
> Tożsamość użytkownika przekazuje zaufany Gateway w nagłówkach. Serwisy pilnują własności danych i uprawnień, dlatego klient nie może usuwać cudzego adresu ani wywołać wewnętrznego czyszczenia koszyka.
>
> Przygotowane są Dockerfile, Compose, manifesty Kubernetes, dokumentacja OpenAPI i kolekcje Postman. Sprawdzono 49 różnych testów oraz działanie przez Compose. Wspólne wdrożenie w Kubernetes i integracja z rzeczywistymi usługami zespołu pozostają do wykonania.

Używaj tego jako planu wypowiedzi i opisuj faktycznie zrozumiane fragmenty. Sposób korzystania z pomocy przy kodzie powinien być zgodny z zasadami prowadzącego.

## 3. Przykład działania User

Klient chce zmienić nazwisko:

1. Frontend wysyła `PUT /api/v1/users/me` do Gateway, dołączając JWT.
2. Gateway sprawdza JWT, usuwa klienckie nagłówki tożsamości i ustawia własne `X-User-Id` oraz `X-User-Roles`.
3. User otrzymuje `PUT /users/me`. Pydantic sprawdza imię i nazwisko.
4. `routes.py` wywołuje `ProfileService.update()` z `service.py`.
5. SQLAlchemy pobiera profil z PostgreSQL i zapisuje zmiany w transakcji.
6. User odsyła JSON ze zmienionymi danymi.

`/me` oznacza użytkownika z nagłówka ustawionego przez Gateway. Nie pozwalamy podać w JSON dowolnego ID właściciela. W adresach dostawy sprawdzamy jednocześnie ID adresu i właściciela.

## 4. Przykład działania Cart

Klient dodaje dwie sztuki produktu:

1. Cart otrzymuje `POST /cart/items` z `productId` i `quantity: 2`.
2. Pydantic sprawdza UUID i całkowitą ilość od 1 do 999. Nie przyjmuje ceny ani ID właściciela od klienta.
3. Cart znajduje Product i Inventory przez Consula.
4. Product zwraca np. nazwę i cenę `"19.99"`, Inventory np. 10 dostępnych sztuk.
5. Cart oblicza nową ilość i zapisuje pozycję w Redisie. Dwie sztuki kosztują `"39.98"`.
6. Zapis odświeża TTL koszyka. Zwykły odczyt go nie odświeża.

Zapis jest atomowy: Lua w Redisie sprawdza poprzednią wartość i zapisuje nową w jednym kroku. Chroni to przed utratą zmian, gdy dwa żądania dodają produkt jednocześnie. To działa również przy kilku replikach serwisu.

## 5. Mapa plików, którą warto zapamiętać

| Plik | Proste wyjaśnienie |
| --- | --- |
| `main.py` | Składa aplikację i zarządza jej startem oraz zamknięciem. |
| `routes.py` | Określa, która funkcja obsługuje dany adres HTTP. |
| `schemas.py` | Określa poprawny kształt danych JSON i waliduje wejście. |
| `service.py` | Zawiera reguły biznesowe: co wolno zrobić i jak obliczyć wynik. |
| `models.py` w User | Opisuje strukturę danych zapisywanych w tabelach. |
| `repository.py` w User | Wykonuje zapytania do bazy. |
| `database.py` w User | Zapewnia połączenie i sesję SQLAlchemy. |
| `migrations/` w User | Przechowuje historię zmian schematu bazy. |
| `store.py` w Cart | Zapisuje i odczytuje koszyk w Redisie. |
| `catalog.py` w Cart | Pobiera cenę i dostępność przez HTTP. |
| `circuit_breaker.py` w Cart | Ogranicza wywoływanie niedziałającej zależności. |
| `discovery.py` | Rejestruje instancję i znajduje adresy usług w Consulu. |
| `security.py` | Odczytuje tożsamość oraz sprawdza role. |
| `http_support.py` | Ujednolica błędy, request ID, logi i opisy nagłówków. |
| `tests/` | Sprawdza poprawne działanie i ścieżki błędów. |

## 6. Pojęcia, które warto rozumieć

- **REST API** — sposób komunikacji przez HTTP. GET pobiera dane, POST tworzy lub dodaje, PUT zastępuje dane profilu, PATCH zmienia ilość, DELETE usuwa.
- **Endpoint** — konkretna metoda i ścieżka, np. `GET /cart`.
- **Walidacja** — sprawdzenie, czy dane są poprawne, np. ilość nie może być ułamkiem.
- **Autoryzacja** — sprawdzenie, czy wywołujący ma prawo wykonać operację.
- **ORM / SQLAlchemy** — mapowanie obiektów Pythona na tabele i generowanie zapytań SQL.
- **Transakcja** — grupa operacji zatwierdzana razem albo wycofywana przy błędzie.
- **Migracja / Alembic** — zapisana i ponumerowana zmiana struktury bazy.
- **TTL** — czas, po którym Redis automatycznie usuwa koszyk.
- **Discovery / Consul** — znalezienie aktualnego adresu zdrowej instancji usługi.
- **Retry** — ograniczone powtórzenie nieudanego zapytania GET.
- **Circuit breaker** — czasowe przerwanie wywołań do niedziałającej usługi. Tutaj próg to co najmniej pięć błędów w ostatnich dziesięciu wywołaniach, a przerwa trwa 30 sekund.
- **Liveness** — proces żyje. **Readiness** — usługa może obsługiwać żądania, w tym ma dostęp do własnego magazynu danych.

## 7. Co pokazać podczas demonstracji

1. Swagger User pod `/docs`: utworzenie profilu z rolą SERVICE, odczyt jako CUSTOMER oraz dodanie adresu.
2. Próbę odczytu dowolnego profilu jako CUSTOMER: odpowiedź 403.
3. Próbę przesłania niepoprawnych danych: odpowiedź 400 ze wspólnym formatem błędu.
4. Swagger Cart: pusty koszyk; po podłączeniu Product dodanie produktu, zmianę ilości i usunięcie pozycji.
5. Próbę wewnętrznego czyszczenia bez SERVICE: odpowiedź 403.
6. Wynik pytest uruchomionego osobno w katalogu każdej usługi.
7. Consul pokazujący zdrowe instancje oraz kontenery uruchomione przez Compose.

Nagłówki w Swaggerze służą do bezpośredniej lokalnej demonstracji. We wdrożonym sklepie serwisy muszą być osłonięte przez Gateway i izolację sieciową. Role w nagłówkach nie są samodzielnym sposobem uwierzytelniania publicznego API.

## 8. Czego nie przypisywać tej części projektu

Nie ma tu implementacji logowania/JWT, całego frontendu, płatności, rezerwacji towaru ani wysyłki. Dla Product i Inventory testy używają atrap. Zapisana cena koszyka nie gwarantuje ceny końcowego zakupu. Przy sprawdzaniu dostępności nie rezerwujemy towaru. Manifesty Kubernetes są przygotowane, ale nie zostały wdrożone w docelowym klastrze. Te granice warto powiedzieć wprost podczas prezentacji.
