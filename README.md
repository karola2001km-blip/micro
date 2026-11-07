# User i Cart — integracja ze sklep-mikroserwisy

Kod dopasowany do przesłanego szkieletu gałęzi `develop`. Wszystkie pliki przeznaczone do wspólnego repozytorium znajdują się w dwóch katalogach:

```text
services/
├── user-service/
│   ├── app/
│   ├── tests/
│   ├── migrations/
│   ├── k8s/
│   ├── postman/
│   ├── docs/
│   ├── dev/
│   ├── Dockerfile
│   ├── .env.example
│   └── README.md
└── cart-service/
    ├── app/
    ├── tests/
    ├── k8s/
    ├── postman/
    ├── docs/
    ├── Dockerfile
    ├── .env.example
    └── README.md
```

- [Paczka do wspólnego repo](paczki/user-cart-do-wspolnego-repo.zip) — zawiera tylko folder `services` z dwoma serwisami.
- [Instrukcja przeniesienia](JAK-PRZENIESC.md).
- [User Service](services/user-service/README.md).
- [Cart Service](services/cart-service/README.md).
- [Opis rozwiązania do prezentacji](services/user-service/docs/PREZENTACJA-USER-CART.md).

## Zmiany względem osobnej wersji

Manifesty aplikacji są teraz w `services/<usługa>/k8s/`, kolekcje Postman w `services/<usługa>/postman/`, a dokumentacja w `services/<usługa>/docs/`. Każda usługa ma własny README, plik wersji Pythona i `.env.example` bez wartości.

Manifesty wskazują namespace `shop`. Lokalne Compose znajduje się w `services/user-service/dev/`; uruchamia oba serwisy i ich infrastrukturę deweloperską. Propozycje polityk sieciowych są opisane w dokumentacji do uzgodnienia z architektem, ponieważ wspólne NetworkPolicy należą do `deploy/k8s/`.

W dopasowanej kopii szkieletu nie zmieniono głównego README, CODEOWNERS, wspólnych folderów `deploy`, `docs`, `postman`, frontendu ani katalogów pozostałych usług.

## Sprawdzenie

Po przeniesieniu uruchomiono 49 testów pytest, zbudowano oba obrazy Docker i sprawdzono start Compose. Obie usługi przechodzą health checki Consula. Newman wykonał 7 żądań User i 5 żądań Cart, wszystkie asercje przeszły. Product i Inventory podczas sprawdzania Cart były atrapami.

Wdrożenie Kubernetes, konfiguracja wspólnej infrastruktury i integracja z rzeczywistymi usługami zespołu pozostają oddzielnym etapem.
