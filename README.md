# User Service i Cart Service w Pythonie

Twoja część sklepu z dokumentu „Architektura sklepu mikroserwisowego” (0.1): dwa niezależne serwisy **Python 3.12 + FastAPI**. User korzysta z PostgreSQL, Cart z Redisa. Zachowano biznesowe ścieżki API, pola JSON i kody odpowiedzi poprzedniej wersji.

| Serwis | Odpowiedzialność | Technologie | Port |
| --- | --- | --- | --- |
| User | Profile i adresy dostawy; bez haseł i logowania | FastAPI, Pydantic, SQLAlchemy, Alembic, PostgreSQL | 8082 |
| Cart | Pozycje i ilości, aktualna cena z Product, sprawdzenie Inventory, TTL 7 dni | FastAPI, Pydantic, httpx, Redis | 8085 |

**Zacznij od [opisu kodu](docs/OPIS-KODU.md)** — wyjaśnia odpowiedzialność każdego pliku i przebieg żądań krok po kroku.

- [Jak opowiedzieć o projekcie na zajęciach](docs/JAK-OPOWIEDZIEC-O-PROJEKCIE.md)
- [Uruchomienie i testy](docs/URUCHOMIENIE.md)
- [Kontrakty API i ustalenia z zespołem](docs/KONTRAKTY.md)
- [Wyniki weryfikacji](docs/WERYFIKACJA.md)

## Najprostsze uruchomienie: Docker Compose

```sh
cp .env.example .env
# W pliku .env ustaw własne lokalne DB_PASSWORD.
docker compose up --build -d
curl --fail http://localhost:8082/health/ready
curl --fail http://localhost:8085/health/ready
```

Compose uruchamia PostgreSQL, Redis, Consul, migrację bazy User oraz obie aplikacje. Migracja kończy się przed startem User. Lokalnie porty są dostępne tylko na loopback.

Swagger obu serwisów: `/docs`; OpenAPI JSON: `/openapi.json` na ich portach. Dokumentacja zawiera pola do wpisania nagłówków podczas lokalnych testów. W pełnym sklepie nagłówki tożsamości ustawia Gateway po sprawdzeniu JWT.

Bez pozostałych usług można testować profile, adresy i pusty koszyk. Dodanie produktu wymaga Product zarejestrowanego w Consulu. Brak Inventory jest dopuszczalny i oznaczony `inventoryChecked:false`; brak Product daje 503. Testy koszyka uruchamiają własne atrapy HTTP tych zależności.

## Lokalny przykład profilu

```sh
curl --fail-with-body -X POST http://localhost:8082/users \
  -H 'Content-Type: application/json' -H 'X-User-Roles: SERVICE' \
  -d '{"id":"11111111-1111-4111-8111-111111111111","firstName":"Anna","lastName":"Nowak","email":"anna@example.com"}'
curl --fail http://localhost:8082/users/me \
  -H 'X-User-Id: 11111111-1111-4111-8111-111111111111' \
  -H 'X-User-Roles: CUSTOMER'
```

Te nagłówki symulują zaufany Gateway **wyłącznie lokalnie**. Nie wystawiaj User ani Cart bezpośrednio do Internetu; w Kubernetesie mają tylko ClusterIP i NetworkPolicy.

## Co oddajesz i ustalasz z zespołem

Kod: `services/`, Docker Compose i Dockerfile, manifesty `deploy/k8s/`, kolekcje `postman/`, dokumentacja `docs/`. Auth, Gateway, Product, Inventory, Order i Notification pozostają po stronie innych osób. User nie publikuje `user.registered` — robi to Auth. Cart nie rezerwuje towaru — robi to Order.

W tabeli architektury wpisz dla obu swoich usług: **Python 3.12 + FastAPI**. Uzasadnienie: jeden język, walidacja i automatyczne OpenAPI przy zachowaniu uzgodnionych API i oddzielnych baz. Szczegółowe pola JSON wymagające uzgodnienia są opisane w kontraktach. Oryginalny dokument architektury i plik `micro codex` nie zostały zmienione.
