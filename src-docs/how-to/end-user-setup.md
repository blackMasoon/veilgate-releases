---
title: Pełny setup Veilgate – od ZIP do uruchomienia w Dockerze (chmura)
description: Szczegółowy przewodnik krok po kroku dla użytkownika końcowego – pobranie paczki, konfiguracja, licencja, uruchomienie w Dockerze w dowolnej chmurze oraz weryfikacja.
---

# Pełny setup Veilgate – krok po kroku

Ten przewodnik przeprowadzi Cię przez cały proces uruchomienia Veilgate:
- pobranie binariów (ZIP) z wydań,
- przygotowanie i weryfikację licencji,
- wstępną konfigurację YAML,
- uruchomienie w Dockerze (lokalnie lub w dowolnej chmurze),
- podstawową diagnostykę i dobre praktyki bezpieczeństwa.

> Uwaga o licencjach: Veilgate wymaga ważnej licencji dla konfiguracji wykraczających poza tryb demo.
> W sprawie licencji (indywidualne warunki) skontaktuj się: adam@veilgate.tech

## 1) Wymagania wstępne

- System z Docker / Docker Desktop lub środowiskiem uruchomieniowym kontenerów (ECS/Fargate, GKE, AKS, K8s, Nomad, itp.).
- `curl` i `jq` (do szybkich testów i inspekcji JSON).
- Otwarty ruch do portów publicznych (domyślnie 80/443 dla gateway, 9090 dla panelu admin) – w chmurze zwykle za load balancerem z TLS.

## 2) Pobranie paczki (ZIP)

1. Przejdź do strony wydań Veilgate i pobierz najnowszą paczkę ZIP dla swojej platformy.
2. Rozpakuj ZIP – w katalogu znajdziesz binarium `veilgate` i przykładowe konfiguracje w `config/`.

Alternatywa: możesz użyć gotowego obrazu Docker dostarczanego wraz z wydaniem.

## 3) Pozyskanie i ustawienie licencji

Veilgate egzekwuje limity w trybie demo (np. max 1 upstream i 3 trasy). Aby odblokować pełną funkcjonalność:

- Skontaktuj się w sprawie licencji: adam@veilgate.tech (zalecane, warunki są indywidualne).
- Po otrzymaniu tokenu licencyjnego (zaczyna się od `VEIL-1.`) masz dwie opcje konfiguracji:
  - W pliku YAML (sekcja `license:`):

    ```yaml
    license:
      key: "VEIL-1.<twój-token>"
      instance_id: "moja-instancja-gateway"
      # server_url: "https://licenses.twojadomena"  # opcjonalnie, jeśli korzystasz z walidacji online
    ```

  - Jako zmienne środowiskowe (wygodne w chmurze):

    ```bash
    export VEILGATE_LICENSE_KEY="VEIL-1.<twój-token>"
    export VEILGATE_LICENSE_INSTANCE_ID="moja-instancja-gateway"
    # export VEILGATE_LICENSE_SERVER_URL="https://licenses.twojadomena"
    ```

Veilgate automatycznie odczyta powyższe zmienne w czasie startu.

## 4) Minimalna konfiguracja YAML

Plik `veilgate.yaml` (możesz go nazwać inaczej i wskazać przez `-config ...`).

```yaml
server:
  listen_address: ":8080"

admin:
  listen_address: ":9090"

logging:
  level: "info"

metrics:
  enabled: true

# Przykładowe zabezpieczenie kluczem API na trasie
security:
  api_keys:
    - id: "client-1"
      key: "super-tajny-klucz"

upstreams:
  - id: "backend-api"
    endpoints:
      - url: "https://api.twojadomena"

routes:
  - id: "root"
    path: "/"
    method: "GET"
    upstream_id: "backend-api"
    auth:
      api_key: true
```

W produkcji trasy zwykle rozszerzysz o CORS, limity, przepisywanie ścieżek, itp. – patrz „Reference: configuration”.

## 5) Uruchomienie w Dockerze (lokalnie i w chmurze)

### Opcja A – docker run

```bash
docker run --rm -p 8080:8080 -p 9090:9090 \
  -e VEILGATE_LICENSE_KEY="$VEILGATE_LICENSE_KEY" \
  -e VEILGATE_LICENSE_INSTANCE_ID="$VEILGATE_LICENSE_INSTANCE_ID" \
  -v $(pwd)/veilgate.yaml:/etc/veilgate/config.yaml:ro \
  ghcr.io/your-org/veilgate:<wersja> \
  /app/veilgate serve -config /etc/veilgate/config.yaml
```

W chmurze zamień mapowanie portów na odpowiednią definicję usługi / LB. Zalecamy TLS na warstwie LB (443) i HTTP między LB a kontenerem.

### Opcja B – docker-compose

`docker-compose.yml` (fragment):

```yaml
version: "3.9"
services:
  veilgate:
    image: ghcr.io/your-org/veilgate:<wersja>
    ports: ["8080:8080", "9090:9090"]
    environment:
      VEILGATE_LICENSE_KEY: ${VEILGATE_LICENSE_KEY}
      VEILGATE_LICENSE_INSTANCE_ID: ${VEILGATE_LICENSE_INSTANCE_ID}
    volumes:
      - ./veilgate.yaml:/etc/veilgate/config.yaml:ro
    command: ["/app/veilgate", "serve", "-config", "/etc/veilgate/config.yaml"]
```

Uruchom:

```bash
docker compose up -d
```

### Opcja C – dowolna chmura / orkiestrator

- Zbuduj/pushnij obraz do rejestru (ECR/GCR/ACR/ghcr.io).
- Zadeklaruj zmienne środowiskowe licencji i mount z konfiguracją YAML.
- Wystaw gateway za pomocą LB (HTTPS 443) – admin najlepiej za prywatnym LB/VPN lub zabezpieczony (w Veilgate: sesje admin i rate limit na /admin/*).

## 6) Weryfikacja działania

```bash
# Readiness admin (zalecane HTTPS przez LB)
curl -i http://<host>:9090/readyz

# Trasa chroniona API key (przykład)
curl -i -H "X-API-Key: super-tajny-klucz" https://<host>/
```

Jeśli używasz TLS na LB, pamiętaj o właściwych hostach/SNI przy rozmowie z backendami (opcja `proxy.tls` lub `preserve_host_header`).

## 7) Najczęstsze problemy i rozwiązania

- 502/504: `timeout awaiting response headers` – sprawdź dostępność i opóźnienia upstream, ewentualnie zwiększ `server.http.response_header_timeout_seconds` lub per-route `proxy.timeouts`.
- Błędy TLS do upstream: skonfiguruj `proxy.tls.insecure_skip_verify: false` (domyślnie) i/lub SNI (`preserve_host_header: true`), w razie prywatnego CA dostarcz trust store w obrazie.
- Komunikat o licencji: upewnij się, że `license.key` (lub `VEILGATE_LICENSE_KEY`) i `instance_id` są ustawione; rozmiar `routes`/`upstreams` musi mieścić się w limitach licencji.

## 8) Bezpieczeństwo i dobre praktyki

- Nie wystawiaj publicznie admina na 9090 bez ochrony – użyj prywatnego LB/VPN lub kontrolowanego dostępu.
- Konfiguracje wrażliwe trzymaj w zmiennych środowiskowych / managerze sekretów, nie w repo.
- Stosuj `min_machines_running` i autoscaling wg potrzeb (dla platform typu Fly/K8s), ale pamiętaj o cold‑startach.

## 9) Gdzie po pomoc i licencję

- Kontakt w sprawie licencji (zalecane, indywidualne ustalenia):
  - adam@veilgate.tech
- Zgłoszenia techniczne / wsparcie wdrożeniowe: ten sam adres lub kanał projektu.

