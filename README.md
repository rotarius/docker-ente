# docker-ente

Self-hosted [Ente](https://ente.io/) (Ende-zu-Ende-verschlüsselte Foto-/Video-Backup-App) für die Domain `cloud-works.ch`, integriert in [docker-traefik](https://github.com/rotarius/docker-traefik).

Basiert auf dem offiziellen [Ente Self-Hosting Quickstart](https://github.com/ente-io/ente/blob/main/server/docs/quickstart.md), angepasst für den Betrieb hinter Traefik statt direkt exponierter Ports, und für den gemeinsam genutzten Objektspeicher aus [docker-minio](https://github.com/rotarius/docker-minio).

## Architektur

```
Internet
  │
  ▼
Traefik (websecure, TLS via myresolver)
  ├── ente.cloud-works.ch         →  web (Photos-App, Port 3000)
  ├── ente-albums.cloud-works.ch  →  web (Public Albums, Port 3002)
  └── ente-api.cloud-works.ch     →  museum (API, Port 8080)

museum ──▶ postgres (internes Netzwerk, nicht öffentlich, nur in diesem Repo)
museum ──▶ minio    (aus docker-minio, geteilter Objektspeicher, s3.cloud-works.ch)
```

| Service | Image | Beschreibung |
|---------|-------|-------------|
| **museum** | `ghcr.io/ente/server` | Ente API-Server |
| **web** | `ghcr.io/ente/web` | Photos-Webapp + Public-Albums-Webapp |
| **postgres** | `postgres:15` | Datenbank (eigene Instanz, nur für Ente) |

Der Objektspeicher (S3) für Fotos/Videos läuft **nicht** in diesem Repo, sondern in [docker-minio](https://github.com/rotarius/docker-minio) - einer geteilten MinIO-Instanz für mehrere Apps, analog zur geteilten Postgres/PostGIS-Instanz in [docker-postgis](https://github.com/rotarius/docker-postgis).

## Voraussetzungen

- Docker & Docker Compose (≥ 2.30)
- Das externe Netzwerk `traefik_network` muss existieren (siehe [docker-traefik](https://github.com/rotarius/docker-traefik)):
  ```bash
  docker network create traefik_network
  ```
- [docker-minio](https://github.com/rotarius/docker-minio) muss laufen (für den Objektspeicher).

## Einrichtung

```bash
cp .env.example .env
cp museum.yaml.example museum.yaml
```

1. In `.env`: `POSTGRES_PASSWORD` setzen.
2. In docker-minio drei Buckets für Ente anlegen (siehe dessen README):
   ```bash
   docker exec -it minio mc alias set local http://localhost:9000 <MINIO_ROOT_USER> <MINIO_ROOT_PASSWORD>
   docker exec -it minio mc mb local/ente-b2-eu-cen
   docker exec -it minio mc mb local/ente-wasabi-eu-central-2-v3
   docker exec -it minio mc mb local/ente-scw-eu-fr-v3
   ```
3. In `museum.yaml`: `db.password` = `POSTGRES_PASSWORD`, `s3.*.key` / `s3.*.secret` = die MinIO-Credentials aus Schritt 2, und `key.encryption`, `key.hash`, `jwt.secret` mit zufälligen Werten füllen:
   ```bash
   openssl rand -base64 32   # key.encryption, jwt.secret
   openssl rand -base64 64   # key.hash
   ```
4. Starten:
   ```bash
   docker compose up -d
   ```
5. Account anlegen unter `https://ente.cloud-works.ch`. Der Bestätigungscode erscheint in den Logs von `museum`:
   ```bash
   docker compose logs -f museum
   ```

## Mobile Apps

In den Ente-Mobile-Apps unter "Custom Server" `https://ente-api.cloud-works.ch` eintragen.

## Wichtiger Hinweis

Laut [offizieller Doku](https://github.com/ente-io/ente/blob/main/server/docs/quickstart.md#caveat) ist ein Setup mit selbst gehostetem MinIO + Postgres für den Einstieg gedacht. Für ernsthaften Produktivbetrieb empfiehlt Ente, einen externen (verwalteten) S3-Provider und eine externe/verwaltete Datenbank zu verwenden, sowie eine funktionierende Backup-Strategie zu haben, bevor eigene Fotos ausschließlich hier gespeichert werden.

## Volumes sichern / zurücksetzen

```bash
# Backup des Volumes postgres_data sowie museum.yaml nicht vergessen -
# ohne museum.yaml (Encryption-Keys) sind die Fotos in docker-minio nicht mehr
# entschlüsselbar. Das Backup von docker-minio (minio-data) läuft separat, siehe
# dessen README.

# Alles inkl. Datenbank-Daten löschen (Fotos in docker-minio bleiben davon unberührt):
docker compose down --volumes
```
