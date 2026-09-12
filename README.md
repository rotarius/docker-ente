# docker-ente

Self-hosted [Ente](https://ente.io/) (Ende-zu-Ende-verschlüsselte Foto-/Video-Backup-App) für die Domain `cloud-works.ch`, integriert in [docker-traefik](https://github.com/rotarius/docker-traefik).

Basiert auf dem offiziellen [Ente Self-Hosting Quickstart](https://github.com/ente-io/ente/blob/main/server/docs/quickstart.md), angepasst für den Betrieb hinter Traefik statt direkt exponierter Ports, und für die geteilte Infrastruktur aus [docker-postgis](https://github.com/rotarius/docker-postgis) (Datenbank) und [docker-minio](https://github.com/rotarius/docker-minio) (Objektspeicher).

## Architektur

```
Internet
  │
  ▼
Traefik (websecure, TLS via myresolver)
  ├── ente.cloud-works.ch         →  web (Photos-App, Port 3000)
  ├── ente-albums.cloud-works.ch  →  web (Public Albums, Port 3002)
  └── ente-api.cloud-works.ch     →  museum (API, Port 8080)

museum ──▶ db    (aus docker-postgis, geteilte Postgres-Instanz, DB "ente_db")
museum ──▶ minio (aus docker-minio, geteilter Objektspeicher, s3.cloud-works.ch)
```

| Service | Image | Beschreibung |
|---------|-------|-------------|
| **museum** | `ghcr.io/ente/server` | Ente API-Server |
| **web** | `ghcr.io/ente/web` | Photos-Webapp + Public-Albums-Webapp |

Dieses Repo enthält selbst **keine** Datenbank und **keinen** Objektspeicher. Beides läuft zentral in [docker-postgis](https://github.com/rotarius/docker-postgis) bzw. [docker-minio](https://github.com/rotarius/docker-minio) und wird von mehreren Apps geteilt (analog zu `openproject`, das dieselbe Postgres-Instanz nutzt).

## Voraussetzungen

- Docker & Docker Compose (≥ 2.30)
- Das externe Netzwerk `traefik_network` muss existieren (siehe [docker-traefik](https://github.com/rotarius/docker-traefik)):
  ```bash
  docker network create traefik_network
  ```
- [docker-postgis](https://github.com/rotarius/docker-postgis) muss laufen (für die Datenbank).
- [docker-minio](https://github.com/rotarius/docker-minio) muss laufen (für den Objektspeicher).

## Einrichtung

```bash
cp museum.yaml.example museum.yaml
```

1. In `docker-postgis` eine eigene Rolle + Datenbank für Ente anlegen (analog zu `openproject`, nicht der geteilte Superuser):
   ```bash
   cd ../docker-postgis
   docker compose exec db psql -U docker -d gis -c "CREATE ROLE ente_db WITH LOGIN PASSWORD '<password>';"
   docker compose exec db psql -U docker -d gis -c "CREATE DATABASE ente_db OWNER ente_db;"
   ```
2. In `docker-minio` drei Buckets für Ente anlegen (siehe dessen README):
   ```bash
   docker exec -it minio mc alias set local http://localhost:9000 <MINIO_ROOT_USER> <MINIO_ROOT_PASSWORD>
   docker exec -it minio mc mb local/ente-b2-eu-cen
   docker exec -it minio mc mb local/ente-wasabi-eu-central-2-v3
   docker exec -it minio mc mb local/ente-scw-eu-fr-v3
   ```
3. In `museum.yaml`: `db.password` mit dem Passwort aus Schritt 1, `s3.*.key` / `s3.*.secret` mit den MinIO-Credentials aus Schritt 2 füllen, und `key.encryption`, `key.hash`, `jwt.secret` mit zufälligen Werten:
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

Laut [offizieller Doku](https://github.com/ente-io/ente/blob/main/server/docs/quickstart.md#caveat) ist ein selbst gehostetes Setup für den Einstieg gedacht. Für ernsthaften Produktivbetrieb empfiehlt Ente, eine funktionierende Backup-Strategie zu haben, bevor eigene Fotos ausschließlich hier gespeichert werden.

Die geteilte Nutzung von `docker-postgis` bedeutet außerdem: ein Neustart/eine Wartung dort (z.B. wegen eines anderen Dienstes) reißt kurzzeitig auch die Ente-DB-Verbindung mit runter. Der DB-Zugriff selbst ist über die eigene Rolle `ente_db` auf die eigene Datenbank beschränkt (siehe [DATABASES.md](https://github.com/rotarius/docker-postgis/blob/develop/DATABASES.md)).

## Backup

Es gibt hier kein eigenes Backup-Volume mehr für die Datenbank:
- Die Postgres-Daten (`ente_db`) werden automatisch vom `dbbackups`-Service in `docker-postgis` mitgesichert (dessen `DBLIST` sichert standardmäßig alle Datenbanken der Instanz).
- Die Fotos/Videos in `docker-minio` (`minio-data`-Volume) werden separat gesichert, siehe dessen README.
- `museum.yaml` selbst (Encryption-Keys) unbedingt außerhalb von Git sichern - ohne sie sind die Daten nicht mehr entschlüsselbar.
