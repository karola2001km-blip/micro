# Weryfikacja wersji Python — 4 listopada 2026

## Wykonane

| Sprawdzenie | Wynik |
| --- | --- |
| User: pytest + PostgreSQL 17 | **17 testów**, wszystkie przeszły, 0 pominiętych. |
| User: pytest + tymczasowa SQLite | **17 testów**, wszystkie przeszły, 0 pominiętych. To alternatywny przebieg tego samego zestawu. |
| Cart: pytest + Redis 7.4 | **32 testy**, wszystkie przeszły, 0 pominiętych. |
| Instalacja z locków i hashy | Sukces; zgodne przypięte wersje Pydantic/FastAPI. |
| Obraz Docker User | Zbudowany, Python 3.12, aplikacja działa bez roota. |
| Obraz Docker Cart | Zbudowany, Python 3.12, aplikacja działa bez roota. |
| Uruchomienie Compose ze zbudowanymi obrazami | Sukces: PostgreSQL, Redis, Consul, jednorazowa migracja oraz obie aplikacje. |
| Rzeczywisty Consul | Obie instancje zarejestrowane; ich health checki mają stan passing. |
| User po HTTP w Compose | Tworzenie i odczyt profilu, dodanie i usunięcie adresu działają. |
| Cart po HTTP w Compose | Dodanie, zmiana ilości, kwoty, odczyt przez Order i czyszczenie tylko z SERVICE działają. |
| Discovery zależności w Compose | Cart znalazł atrapy Product i Inventory przez prawdziwy Consul i pobrał dane po rzeczywistym HTTP. |
| Powtórzenie migracji i restart User | Sukces; zapisany profil nadal dostępny po restarcie aplikacji. |
| Pliki Kubernetes YAML i Postman JSON | Poprawnie sparsowane. |

Łącznie **49 różnych testów** wersji Python (17 User + 32 Cart). Dodatkowy przebieg User na SQLite nie zwiększa liczby różnych testów. W User testowany jest pełny pipeline FastAPI przez TestClient; w Cart — ASGITransport z prawdziwym Redisem. Testy obejmują walidację, uprawnienia, własność danych, konflikty, ceny, TTL, współbieżność, fallbacki i odporność na awarie. Testy circuit breakera sprawdzają też dokładne okno 10 wyników oraz pojedynczą próbę half-open.

W testach automatycznych Product, Inventory i Consul korzystają z transportu testowego httpx. Oddzielny test Compose używa prawdziwego Consula, a Product i Inventory zastępuje tymczasowy serwer HTTP w osobnym kontenerze. Nie są to implementacje usług innych członków zespołu.

Naprawiono lokalny problem Consula za proxy: nazwy wewnętrznych serwisów Compose zostały dopisane do NO_PROXY, aby health checki nie trafiały do proxy internetowego. Weryfikacja TLS oraz hashy pakietów pozostała włączona. Certyfikat proxy przekazano do budowy obrazów jako BuildKit secret.

Środowisko: Python 3.12; FastAPI 0.115.12, Pydantic 2.11.7, SQLAlchemy 2.0.41, Alembic 1.16.2, redis 6.2.0 i httpx 0.28.1. TestClient zgłasza jedno ostrzeżenie DeprecationWarning dotyczące aliasu AnyIO. Nie wpływa ono na wynik testów.

## Granice weryfikacji

- Nie wykonano wdrożenia w Kubernetes/Rancher ani testu egzekwowania NetworkPolicy przez docelowe CNI.
- Nie sprawdzono integracji z rzeczywistymi implementacjami Gateway, Auth, Product, Inventory, Order i Notification. Szczegółowe kontrakty są w KONTRAKTY.md.
- Nie uruchamiano kolekcji w Postman/Newman; ich format jest poprawny, a odpowiednie operacje API sprawdzają testy opisane wyżej.
- Nie przenoszono istniejącej bazy wersji Java/Flyway do Alembic. Weryfikowano nowe, izolowane bazy wersji Python.
- Publikacja źródeł na gałęzi `python-user-cart` w GitHub nie oznacza wdrożenia aplikacji ani opublikowania nowej wersji środowiska.

Kontenery, atrapy oraz dane utworzone wyłącznie na potrzeby tej weryfikacji są usuwane po testach. Kod źródłowy, dokumentacja i zbudowane obrazy pozostają dostępne.
