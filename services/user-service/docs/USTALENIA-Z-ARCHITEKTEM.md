# Ustalenia przed integracją

Kod User i Cart znajduje się wyłącznie w katalogach właściciela. Poniższe zmiany dotyczą wspólnych plików i wymagają przeglądu architekta.

## Metadane repozytorium

W głównym README technologie obu usług powinny zostać ustawione na Python 3.12 / FastAPI. User korzysta z PostgreSQL, a Cart z Redisa. Właściciel: `@karola2001km-blip`; imię do uzupełnienia.

Proponowane wpisy CODEOWNERS:

```text
/services/user-service/          @karola2001km-blip @Michal-Jarek
/services/cart-service/          @karola2001km-blip @Michal-Jarek
```

## Infrastruktura

- Namespace `shop`.
- PostgreSQL: host `user-postgres`, baza `user_db`, użytkownik `user_service`; w przypadku wspólnego serwera trzeba dopasować DB_HOST w Deployment i Job migracyjnym.
- Secret `user-db-secret`: klucz `password`.
- Secret `cart-redis-secret`: klucz `url`.
- Consul pod nazwą `consul`, port 8500.
- Reguły sieciowe do wspólnego `deploy/k8s/`: propozycje w `services/user-service/docs/network-policy.proposed.yaml` i `services/cart-service/docs/network-policy.proposed.yaml`.
- Gateway usuwa prefiks `/api/v1` przed przekazaniem żądania. POST `/users` i DELETE `/cart/{userId}` pozostają endpointami wewnętrznymi.

## Kontrakty usług

Auth przekazuje UUID konta do POST `/users`. Product zwraca `name` i cenę `price` jako napis z dwoma miejscami po kropce. Inventory zwraca całkowite `available`. Order odczytuje koszyk z ID klienta i rolą SERVICE, a po zakupie wywołuje wewnętrzne czyszczenie.

Konfiguracja Consul KV, wspólne środowiska Postman i scenariusz zakupu przez Gateway nie wchodzą w tę zmianę. Szczegóły pól JSON są w `docs/KONTRAKT.md` obu usług.
