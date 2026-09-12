# docker-ente

Self-hosted [Ente](https://ente.io/) (Ende-zu-Ende-verschlüsselte Foto-/Video-Backup-App) für die Domain `cloud-works.ch`, integriert in [docker-traefik](https://github.com/rotarius/docker-traefik).

Basiert auf dem offiziellen [Ente Self-Hosting Quickstart](https://github.com/ente-io/ente/blob/main/server/docs/quickstart.md), angepasst für den Betrieb hinter Traefik statt direkt exponierter Ports.

## Architektur

```
Internet
  │
  ▼
Traefik (websecure, TLS via myresolver)
  ├── ente.cloud-works.ch         →  web (Photos-App, Port 3000)
  ├── ente-albums.cloud-works.ch  →  web (Public Albums, Port 3002)
  ├── ente-api.cloud-works.ch     →  museum (API, Port 8080)
  └── ente-s3.cloud-works.ch      →  minio (S3-API, Port 3200)

museum ──▶ postgres (internes Netzwerk, nicht öffentlich)
museum ──▶ minio    (Objektspeicher für Fotos/Videos)
```

| Service | Image | Beschreibung |
|---------|-------|-------------|
| **museum** | `ghcr.io/ente/server` | Ente API-Server |
| **web** | `ghcr.io/ente/web` | Photos-Webapp + Public-Albums-Webapp |
| **postgres** | `postgres:15` | Datenbank |
| **minio** | `minio/minio` | S3-kompatibler Objektspeicher für Fotos/Videos |

`postgres` bleibt intern (kein Traefik-Zugriff). `minio` wird bewusst öffentlich über Traefik geroutet, damit museum, die Webapp und die Mobile-Apps direkt signierte Upload-/Download-URLs auflösen können (path-style URLs, siehe `museum.yaml.example`).

## Voraussetzungen

- Docker & Docker Compose (≥ 2.30)
- Das externe Netzwerk `traefik_network` muss existieren (siehe [docker-traefik](https://github.com/rotarius/docker-traefik)):
  ```bash
  docker network create traefik_network
  ```

## Einrichtung

```bash
cp .env.example .env
cp museum.yaml.example museum.yaml
```

1. In `.env`: `POSTGRES_PASSWORD`, `MINIO_ROOT_USER` und `MINIO_ROOT_PASSWORD` setzen.
2. In `museum.yaml`: dieselben Werte bei `db.password` sowie bei jedem `s3.*.key` / `s3.*.secret` eintragen, und `key.encryption`, `key.hash`, `jwt.secret` mit zufälligen Werten füllen:
   ```bash
   openssl rand -base64 32   # key.encryption, jwt.secret
   openssl rand -base64 64   # key.hash
   ```
3. Starten:
   ```bash
   docker compose up -d
   ```
4. Account anlegen unter `https://ente.cloud-works.ch`. Der Bestätigungscode erscheint in den Logs von `museum`:
   ```bash
   docker compose logs -f museum
   ```

## Mobile Apps

In den Ente-Mobile-Apps unter "Custom Server" `https://ente-api.cloud-works.ch` eintragen.

## Wichtiger Hinweis

Laut [offizieller Doku](https://github.com/ente-io/ente/blob/main/server/docs/quickstart.md#caveat) ist dieses Setup (lokales MinIO + lokales Postgres) für den Einstieg gedacht. Für ernsthaften Produktivbetrieb empfiehlt Ente, einen externen S3-Provider und eine externe/verwaltete Datenbank zu verwenden, sowie eine funktionierende Backup-Strategie zu haben, bevor eigene Fotos ausschließlich hier gespeichert werden.

## Volumes sichern / zurücksetzen

```bash
# Backup der Volumes (postgres_data, minio_data) sowie museum.yaml nicht vergessen -
# ohne museum.yaml (Encryption-Keys) sind die Daten in den Volumes nicht mehr entschlüsselbar.

# Alles inkl. Daten löschen:
docker compose down --volumes
```
