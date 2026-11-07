# User Service i Cart Service

Serwisy profili użytkowników i koszyka dla sklepu mikroserwisowego. Implementacja w Pythonie 3.12 z FastAPI, zgodna z podziałem odpowiedzialności opisanym w architekturze projektu.

| Serwis | Zakres | Baza | Port |
| --- | --- | --- | --- |
| User | Profile klientów i adresy dostawy | PostgreSQL | 8082 |
| Cart | Pozycje koszyka, ilości, ceny i dostępność | Redis | 8085 |

## Uruchomienie

Wymagany Docker z Compose.

```sh
cp .env.example .env
# Ustaw DB_PASSWORD w .env.
docker compose up --build -d
```

Compose uruchamia PostgreSQL, Redis, Consul, migrację bazy i obie aplikacje. User startuje po zakończeniu migracji. Porty lokalne są związane z interfejsem loopback.

Sprawdzenie gotowości:

```sh
curl --fail http://localhost:8082/health/ready
curl --fail http://localhost:8085/health/ready
```

Swagger jest dostępny pod `/docs`, a OpenAPI pod `/openapi.json` na porcie danej usługi.

## Integracja

User obsługuje dane profilu. Konta, hasła i JWT należą do Auth. Cart pobiera cenę z Product i sprawdza stan w Inventory. Rezerwację towaru oraz utworzenie zamówienia obsługuje Order.

Dodanie produktu wymaga działającego Product zarejestrowanego w Consulu. Brak ceny powoduje odpowiedź 503. Przy niedostępnym Inventory zapis jest dopuszczalny z polem `inventoryChecked:false`. Koszyk wygasa po siedmiu dniach od ostatniej zmiany.

Tożsamość przekazuje Gateway przez `X-User-Id` i `X-User-Roles`. Bezpośrednie ustawianie tych nagłówków jest przeznaczone do testów lokalnych. W klastrze usługi wymagają izolacji sieciowej i dostępu przez zaufane komponenty.

## Dokumentacja

- [Struktura i działanie kodu](docs/OPIS-KODU.md)
- [Uruchomienie i testy](docs/URUCHOMIENIE.md)
- [Kontrakty API](docs/KONTRAKTY.md)
- [Zakres prezentacji](docs/PREZENTACJA.md)
- [Wyniki weryfikacji](docs/WERYFIKACJA.md)

Kolekcje Postman znajdują się w `postman/`, a manifesty Kubernetes w `deploy/k8s/`. Opis konfiguracji klastra i wymaganych sekretów zawiera instrukcja uruchomienia.
