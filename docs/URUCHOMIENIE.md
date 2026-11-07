# Uruchomienie wersji Python

## 1. Zalecany start: Docker Compose

Wymagania: Docker z Compose, np. Rancher Desktop z silnikiem dockerd. Otwórz terminal w głównym katalogu projektu.

```sh
cp .env.example .env
# Edytuj .env i ustaw DB_PASSWORD na własne hasło lokalne.
docker compose up --build -d
docker compose ps -a
```

W PowerShell zamiast `cp` możesz użyć `Copy-Item .env.example .env`.

`user-migrate` powinien zakończyć się kodem 0 — to poprawne, nie jest serwerem. Pozostałe kontenery powinny działać. Dopiero po wykonaniu migracji uruchamia się User Service. PostgreSQL i Redis zachowują dane w nazwanych wolumenach.

Sprawdź:

```sh
curl --fail http://localhost:8082/health/ready
curl --fail http://localhost:8085/health/ready
```

Oczekiwany wynik obu wywołań: `{"status":"UP"}`. Swagger działa pod `/docs`, OpenAPI JSON pod `/openapi.json` na portach 8082 i 8085. To wspólne ścieżki FastAPI, zastępujące dawne `/v3/api-docs` i `/docs-json` wersji Java/Node.

W Swaggerze przy endpointach wpisz `X-User-Roles`, a przy operacjach na własnych danych również `X-User-Id`. Lokalnie symulujesz w ten sposób Gateway. W sklepie rolę i ID ustawia wyłącznie Gateway po weryfikacji JWT.

Przykładowa kolejność demonstracji:

1. User: POST `/users`, rola SERVICE, ID UUID i dane profilu z przykładu w README.
2. User: GET `/users/me`, to samo ID i rola CUSTOMER.
3. User: POST `/users/me/addresses`, następnie lista adresów.
4. Cart: GET `/cart`, to samo ID i rola CUSTOMER.
5. Po podłączeniu Product dodaj produkt do koszyka i zmień jego ilość.

Product musi rejestrować się w Consulu i zwracać pola opisane w kontraktach. Bez niego dodanie do koszyka zwraca 503; to nie oznacza awarii Redisa ani błędnej instalacji Cart. Testy Cart nie wymagają usług innych osób, bo mają atrapy odpowiedzi HTTP.

Zatrzymanie:

```sh
docker compose down
```

Nie dodawaj `-v`, jeśli chcesz zachować dane. `down -v` usuwa wolumeny. Porty Compose są związane tylko z loopback; nie zmieniaj tego w celu publicznego udostępniania usług.

## 2. Praca bez budowania obrazów aplikacji

Wymagany Python **3.12** oraz działające PostgreSQL, Redis i opcjonalnie Consul. Możesz uruchomić samą infrastrukturę:

```sh
docker compose up -d user-postgres cart-redis consul
```

Użyj osobnego terminala i środowiska `.venv` dla każdej usługi. Pracuj w istniejącym checkoutcie; zadanie chmurowe już jest izolowane i nie trzeba tworzyć Git worktree.

### User Service

```sh
cd services/user-service
python -m venv .venv
# Linux/macOS:
source .venv/bin/activate
# Windows PowerShell zamiast poprzedniej linii:
# .\.venv\Scripts\Activate.ps1
python -m pip install --require-hashes -r requirements-dev.lock
```

Ustaw konfigurację w terminalu albo w lokalnym `.env` w katalogu usługi (ten plik jest ignorowany przez Git):

```dotenv
DB_HOST=localhost
DB_PORT=5432
DB_NAME=user_db
DB_USER=user_service
# DB_PASSWORD wpisz lokalnie: musi odpowiadać hasłu z głównego .env.
CONSUL_ENABLED=false
```

Następnie:

```sh
alembic upgrade head
uvicorn app.main:app --host 127.0.0.1 --port 8082 --reload --no-access-log
```

`CONSUL_ENABLED=false` ułatwia samodzielną pracę na hoście. Żeby użyć discovery, ustaw `CONSUL_ENABLED=true`, `CONSUL_URL` i `SERVICE_ADDRESS` osiągalny z kontenera Consula. Jego `127.0.0.1` oznacza sam kontener, a nie komputer. Pełny Compose ustawia poprawne nazwy w jednej sieci.

### Cart Service

W drugim terminalu:

```sh
cd services/cart-service
python -m venv .venv
# Aktywuj .venv tak jak powyżej.
python -m pip install --require-hashes -r requirements-dev.lock
uvicorn app.main:app --host 127.0.0.1 --port 8085 --reload --no-access-log
```

Opcjonalny lokalny `.env`:

```dotenv
REDIS_URL=redis://localhost:6379/0
CONSUL_ENABLED=false
# Ustaw tylko jeśli masz działające lokalnie Product i Inventory:
PRODUCT_SERVICE_URL=http://localhost:8083
INVENTORY_SERVICE_URL=http://localhost:8084
```

Bez nadpisań adresy zależności pochodzą z Consula. Manifesty produkcyjne nie ustawiają tych nadpisań. Nie dodano fałszywych produktów do serwisu: cena zawsze musi przyjść z prawdziwego Product albo z atrapy wyłącznie podczas testu.

## 3. Testy pytest

Uruchamiaj testy **osobno, w katalogu konkretnej usługi**. Obie mają własny pakiet `app`, więc wspólne uruchomienie z katalogu głównego mieszałoby moduły.

User — domyślnie tymczasowa SQLite, bez działającej infrastruktury:

```sh
cd services/user-service
# Aktywuj właściwe .venv i zainstaluj requirements-dev.lock.
python -m pytest -q
```

User — test na PostgreSQL (wyłącznie baza testowa):

```sh
TEST_DATABASE_URL='postgresql+psycopg://uzytkownik:haslo@localhost:5432/test_user_db' python -m pytest -q
```

Wpisz własne dane połączenia lokalnie. W PowerShell ustaw `$env:TEST_DATABASE_URL='...'`, a następnie `python -m pytest -q`. Jeśli hasło ma znaki specjalne, zakoduj je w URL. Testy wykonują migracje i dodają rekordy; nigdy nie wskazuj bazy produkcyjnej.

Cart — wymaga prawdziwego Redisa, używa oddzielnej bazy nr 15:

```sh
cd services/cart-service
TEST_REDIS_URL='redis://localhost:6379/15' python -m pytest -q
```

Domyślnie testy wskazują `redis://127.0.0.1:6379/15`, więc przy lokalnym Redisie wystarczy `python -m pytest -q`. Nie używają `FLUSHDB`: usuwają tylko losowe klucze utworzone w danym teście. Product, Inventory i Consul są zastępowane transportem testowym httpx. Testy circuit breakera używają kontrolowanego zegara, więc nie czekają 30 sekund.

## 4. Migracje i dane poprzedniej wersji

Alembic przechowuje wersję w tabeli `alembic_version`. Kolejne zmiany twórz poleceniem `alembic revision --autogenerate -m 'opis'`, przejrzyj wygenerowany kod, a następnie wykonaj `alembic upgrade head`.

Jeżeli wcześniej uruchomiłaś wersję Java i masz bazę z tabelami Flyway, **nie stosuj automatycznie tej pierwszej migracji do istniejących tabel**. Zachowaj dane i uzgodnij migrację. Schematy są podobne, ale oznaczenie istniejącej bazy przez `alembic stamp 0001` jest poprawne dopiero po sprawdzeniu zgodności tabel i ograniczeń. Najprostszy pierwszy start wersji Python to nowa, pusta baza. Nie usuwamy starych danych automatycznie.

W Compose migracja jest osobnym jednorazowym kontenerem. W Kubernetesie uruchom ją jako Job **przed** rolloutem aplikacji. Nie uruchamiaj migracji równolegle w każdej replice.

## 5. Postman

Importuj kolekcje z `postman/`. W User pierwszy POST zakłada nowy profil; drugi z tym samym ID lub e-mailem daje 409. Przy nowym przebiegu zmień `userId` i e-mail. Dodanie adresu zapisuje zmienną `addressId` do późniejszego usunięcia.

W Cart ustaw `productId` istniejącego produktu. Lokalne nagłówki w kolekcjach symulują Gateway i wywołania wewnętrzne. Wspólna kolekcja zakupu przez Gateway wymaga integracji z Auth i Order.

## 6. Kubernetes / Rancher Desktop

Wspólna infrastruktura, sekrety i zabezpieczenia klastra należą do uzgodnień z architektem. W `deploy/k8s/` są manifesty Twoich aplikacji i migracji, nie całego sklepu.

1. Zbuduj obrazy `user-service:0.1.0` i `cart-service:0.1.0` z katalogów usług i udostępnij je silnikowi Rancher Desktop lub rejestrowi zespołu.
2. W tym samym namespace zapewnij `user-postgres:5432` (baza `user_db`, użytkownik `user_service`), Redis i `consul:8500`. Dane baz wymagają trwałych wolumenów.
3. Utwórz bezpiecznie Secret `user-db-secret` z kluczem `password` i `cart-redis-secret` z kluczem `url`. Nie zapisuj wartości do Git.
4. Uzgodnij etykiety `app`: polityki używają `api-gateway`, `auth-service`, `notification-service`, `order-service`, `product-service`, `inventory-service`, `consul`, `user-postgres`, `cart-redis`.
5. Zastosuj polityki sieciowe, potem `user-migration-job.yaml`. Poczekaj: `kubectl wait --for=condition=complete job/user-db-migration --timeout=120s`.
6. Po sukcesie migracji zastosuj manifesty Deployment/Service User i Cart. Przy następnych migracjach nadaj Jobowi nową nazwę lub usuń zakończony Job przed kolejnym wykonaniem.
7. Sprawdź readiness, wpisy w Consulu, ruch przez Gateway oraz blokowanie ruchu z nieuprawnionego poda. NetworkPolicy wymaga egzekwowania przez CNI. Nie dodawaj Ingressu, NodePort ani LoadBalancer dla tych aplikacji.

## 7. Zależności i proxy

`requirements.in` opisuje zależności bezpośrednie. Pliki `.lock` zawierają dokładne wersje i hashe. Do instalacji używaj `--require-hashes`; narzędzie weryfikuje pobrane pliki. Odświeżanie locków po świadomej zmianie wersji: `pip-compile --generate-hashes --output-file requirements.lock requirements.in` i analogicznie dla wariantu `requirements-dev.in`.

Nie wyłączaj TLS, aby obejść problem certyfikatu. Dockerfile obsługuje opcjonalny BuildKit secret `proxy_ca`; w zarządzanym środowisku można użyć `docker build --secret id=proxy_ca,src="$CODEX_PROXY_CERT" -t user-service:0.1.0 services/user-service` i analogicznie Cart. Certyfikat nie jest zapisywany w obrazie. Potem `docker compose up --no-build -d` używa już zbudowanych obrazów. Normalnie na komputerze wystarcza standardowe `docker compose up --build -d`.

Consul również wykonuje wewnętrzne health checki HTTP. Compose dopisuje nazwy `user-service`, `cart-service` i `consul` do `NO_PROXY`, zachowując istniejące wyjątki. Dzięki temu kontenery kontaktują się wewnątrz swojej sieci; proxy internetowe pozostaje aktywne dla pozostałych adresów.
