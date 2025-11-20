## Veilgate – Quickstart (Docker + docker-compose)

Ten dokument opisuje, jak uruchomić gotowy pakiet release Veilgate przy użyciu Dockera i docker-compose.

### Wymagania

- Zainstalowany Docker (Desktop / Engine).
- Zainstalowany docker-compose (wbudowany w `docker compose` lub osobny `docker-compose`).

### Zawartość paczki

Po rozpakowaniu archiwum `veilgate-<VERSION>.zip` powinieneś zobaczyć strukturę:

- `docker-compose.yml` – definicja usług Veilgate + demo upstream.
- `config/veilgate.yaml` – konfiguracja gatewaya.
- `docs/QUICKSTART.md` – ten plik.
- `VERSION` – wersja release.
  (obrazy Dockera będą pobierane z rejestru, np. GitHub Container Registry).

### Uruchomienie

1. Rozpakuj archiwum:

```bash
unzip veilgate-<VERSION>.zip
cd veilgate-<VERSION>
```

2. Uruchom usługi:

```bash
docker-compose up
```

Pierwsze uruchomienie może chwilę potrwać, ponieważ Docker musi zbudować obrazy.

3. Zweryfikuj działanie:

- Gateway (demo trasa za API gatewayem):

```bash
curl -H 'X-API-Key: demo-secret-key' http://localhost:8080/
```

Powinieneś otrzymać odpowiedź JSON z demo upstreamu.

- Dashboard i admin:
  - `http://localhost:9090/dashboard` – wbudowany dashboard Veilgate.
  - `http://localhost:9090/healthz` – endpoint liveness.
  - `http://localhost:9090/readyz` – endpoint readiness.
  - `http://localhost:9090/metrics` – metryki Prometheus.

### Konfiguracja

Konfigurację gatewaya możesz zmieniać w pliku:

- `config/veilgate.yaml`

Typowe rzeczy do zmiany:

- sekcja `upstreams` – adresy Twoich backendów,
- sekcja `routes` – ścieżki, metody i przypisanie do upstreamów,
- sekcja `security.api_keys` – klucze API (zmień `demo-secret-key` na własne),
- sekcja `rate_limit` – limity per route / IP / API key.

Po zmianie konfiguracji zrestartuj kontenery:

```bash
docker-compose down
docker-compose up
```

### Zatrzymanie

Aby zatrzymać usługi:

```bash
docker-compose down
```


