# Paperless Stack Documentation

## Funktionsbeschreibung des Stacks

Der **Paperless** Stack ermöglicht die digitale Verwaltung und Archivierung von Dokumenten durch die Automatisierung von Dokumenten-Scans, OCR (Optical Character Recognition) und die Organisation in einer durchsuchbaren Datenbank. Der Stack nutzt **Docker Swarm** zur Orchestrierung und **GlusterFS** als persistenten Speicher, wodurch eine hochverfügbare und skalierbare Umgebung geschaffen wird. **Portainer BE** dient als Verwaltungsoberfläche zur einfachen Handhabung und Überwachung der Container-Dienste über die drei Nodes hinweg. 

- **Offizielle Dokumentation:** [Paperless-ng Documentation](https://paperless-ngx.readthedocs.io/)
- **Docker Hub:** [Paperless-ng on Docker Hub](https://hub.docker.com/r/paperlessngx/paperless-ngx)

### Image-Informationen

| Image Name                     | Version | Erstellungsdatum |
|--------------------------------|---------|-------------------|
| redis                          | 7       | 2024-04-27        |
| postgres                       | 16      | 2024-04-27        |
| ghcr.io/paperless-ngx/paperless-ngx | latest  | 2024-04-27        |
| glusterfs-volume               | v1.2.3  | 2024-04-27        |

## YML-Datei Optimierung und Überprüfung

### Optimierte `docker-compose.yml` für den Stack "Paperless"

```yaml
version: '3.8' # Version des Docker Compose

# =====================================
# Dienste
# =====================================
services:
  broker:
    image: docker.io/library/redis:7
    restart: unless-stopped
    volumes:
      - redisdata:/data
    networks:
      - paperless_network
    deploy:
      replicas: 1
      resources:
        limits:
          cpus: '0.3'
          memory: '256M'
      restart_policy:
        condition: on-failure

  db:
    image: docker.io/library/postgres:16
    restart: unless-stopped
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: paperless
      POSTGRES_USER: paperless
      POSTGRES_PASSWORD: paperless
    networks:
      - paperless_network
    deploy:
      replicas: 1
      resources:
        limits:
          cpus: '1.0'
          memory: '1G'
      restart_policy:
        condition: on-failure

  webserver:
    image: ghcr.io/paperless-ngx/paperless-ngx:latest
    restart: unless-stopped
    depends_on:
      - db
      - broker
    ports:
      - "8010:8000"
    volumes:
      - data:/usr/src/paperless/data
      - media:/usr/src/paperless/media
      - /mnt/paperless/export:/usr/src/paperless/export
      - /mnt/paperless/consume:/usr/src/paperless/consume
    environment:
      PAPERLESS_REDIS: redis://broker:6379
      PAPERLESS_DBHOST: db
      PAPERLESS_ADMIN_USER: '%USER%'
      PAPERLESS_ADMIN_PASSWORD: '%PASSWORD%'
      PAPERLESS_URL: '%URL%'
      PAPERLESS_CONSUMER_POLLING: 10
      PAPERLESS_OCR_USER_ARGS: '{"continue_on_soft_render_error": true}'
      PAPERLESS_FILENAME_FORMAT: '{created_year}/{correspondent}/{document_type}/{created_year}{created_month}{created_day}_{correspondent}_{title}'
      PAPERLESS_FILENAME_FORMAT_REMOVE_NONE: 'true'
      PAPERLESS_CONSUMER_ENABLE_BARCODES: 'true'
      PAPERLESS_CONSUMER_ENABLE_ASN_BARCODE: 'true'
      PAPERLESS_CONSUMER_ASN_BARCODE_PREFIX: 'ASN'
      USERMAP_UID: 1000
      USERMAP_GID: 100
    networks:
      - paperless_network
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: '512M'
      restart_policy:
        condition: on-failure
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      rollback_config:
        parallelism: 1
        delay: 10s
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3

# =====================================
# Netzwerkeinstellungen
# =====================================
networks:
  paperless_network:
    driver: overlay

# =====================================
# Volumes
# =====================================
volumes:
  data:
    driver: glusterfs
    driver_opts:
      driver: glusterfs
      voluri: "localhost:/gv0"
      mountpoint: "/mnt/glusterfs/paperless_data"

  media:
    driver: glusterfs
    driver_opts:
      driver: glusterfs
      voluri: "localhost:/gv0"
      mountpoint: "/mnt/glusterfs/paperless_media"

  pgdata:
    driver: glusterfs
    driver_opts:
      driver: glusterfs
      voluri: "localhost:/gv0"
      mountpoint: "/mnt/glusterfs/paperless_pgdata"

  redisdata:
    driver: glusterfs
    driver_opts:
      driver: glusterfs
      voluri: "localhost:/gv0"
      mountpoint: "/mnt/glusterfs/paperless_redisdata"
```

### Kommentare und Empfehlungen

- **Version:** Gibt die Version des Docker Compose-Formats an.
- **Services:** Definiert die einzelnen Container-Dienste mit ihren Konfigurationen.
  - **broker:** Redis-Server für Paperless.
  - **db:** PostgreSQL-Datenbank für Paperless.
  - **webserver:** Paperless Webserver mit den notwendigen Volumes und Umgebungsvariablen.
- **Networks:** Definiert das Overlay-Netzwerk `paperless_network` für die interne Kommunikation der Dienste.
- **Volumes:** Definiert die persistenten Speichervolumes unter Verwendung von GlusterFS.
- **Deploy:** Enthält Deployment-Konfigurationen wie Replikate, Ressourcenlimits und Restart-Policies.
- **Healthcheck:** Überprüft die Verfügbarkeit des Webservers und stellt sicher, dass fehlerhafte Container neu gestartet werden.

**Empfehlung zu Abhängigkeiten:**

- Dienste, die voneinander abhängen (z.B. Webserver von Datenbank und Broker), sind mit `depends_on` konfiguriert, um sicherzustellen, dass die abhängigen Dienste zuerst gestartet werden.

## Ressourcenlimits

### Überprüfung und Empfehlungen

Basierend auf den bereitgestellten Ressourcen (3 Nodes, je 16GB RAM und 2 CPUs) sowie den Best Practices für Docker Swarm, wurden die folgenden Ressourcenlimits empfohlen:

| Dienst    | CPUs  | RAM   | Beschreibung                                 |
|-----------|-------|-------|----------------------------------------------|
| broker    | 0.3   | 256M  | Redis-Server für Messaging und Caching       |
| db        | 1.0   | 1G    | PostgreSQL-Datenbank für Paperless           |
| webserver | 0.5   | 512M  | Paperless Webserver für Dokumentenverwaltung |

**Allgemeine Best Practices:**

- **Limits setzen:** Ressourcenlimits verhindern, dass ein einzelner Dienst die gesamte Ressourcenlast übernimmt.
- **Reserve:** Stellen Sie sicher, dass genügend Ressourcen für das Betriebssystem und andere laufende Prozesse verfügbar sind.
- **Monitoring:** Überwachen Sie die Ressourcennutzung kontinuierlich und passen Sie die Limits bei Bedarf an.

## Storage

### GlusterFS Konfiguration

Der Stack nutzt **GlusterFS** als persistenten Speicher. Das Docker Plugin [glusterfs-volume](https://github.com/chrisbecke/glusterfs-volume) ist installiert und erstellt die benötigten Volumes automatisch im Verzeichnis **/mnt/glusterfs**. Der verwendete Treibername lautet **"glusterfs"**.

#### Volume Namenskonzept

- **Format:** `%STACKNAME%_%VOLUMENAME%`
- **Beispiel:** Für den Stack "paperless" und das Volume "data" lautet der Volume-Name **"paperless_data"**.

#### YML-Beispiel für Volumes

```yaml
volumes:
  paperless_data:
    driver: glusterfs
    driver_opts:
      voluri: "10.11.2.51:/gv0"
      mountpoint: "/mnt/glusterfs/paperless_data"
```

**Hinweis:** In der YML-Datei wird nur das Suffix des Volumenamens verwendet, ohne den Stack- oder Containernamen.

## Secrets

### Namenskonzept und Erstellung

Bei der Verwendung von **Docker Secrets** wird ein durchgängiges und eindeutiges Namenskonzept angewendet, um die Secrets für mehrere Container eindeutig zu identifizieren.

#### Namensschema

- **Format:** `%STACK%_%SERVICE%_%SECRETNAME%`
- **Beispiel:** Für den Dienst "db" und das Secret "password" lautet der Secret-Name **"db_password"**.

#### Erstellung von Secrets

```bash
echo "%PASSWORD%" | docker secret create paperless_webserver_paperless_admin_password -
```

#### YML-Beispiel

```yaml
secrets:
  webserver_paperless_admin_password:
    external: true
```

**Anpassung im YML:**

Passen Sie die `environment`-Variablen an, um Secrets zu verwenden:

```yaml
environment:
  PAPERLESS_ADMIN_PASSWORD_FILE: /run/secrets/webserver_paperless_admin_password
```

Und binden Sie das Secret im Dienst:

```yaml
secrets:
  - webserver_paperless_admin_password
```

## Netzwerk

### Port-Konfiguration und Best Practices

#### Überprüfung der Ports

Stellen Sie sicher, dass nur die notwendigen Ports nach außen freigegeben werden, um die Sicherheit zu erhöhen und Portkonflikte zu vermeiden.

#### Empfohlene Ports

| Dienst     | Interner Port | Externer Port | Beschreibung                      |
|------------|---------------|---------------|-----------------------------------|
| webserver  | 8000          | 8010          | Paperless Webserver               |
| postgres   | 5432          | -             | Datenbankzugriff (intern)         |
| redis      | 6379          | -             | Cache-Zugriff (intern)            |
| portainer  | 9000          | 9000          | Portainer Management Interface    |

#### Netzwerk-Typen

- **Overlay Netzwerke:** Für die Kommunikation zwischen den Services auf verschiedenen Nodes.
- **Ingress Netzwerke:** Für den externen Zugriff auf die Dienste.

#### YML-Beispiel für Netzwerke

```yaml
networks:
  paperless_network:
    driver: overlay
```

## Monitoring und Logging

### Empfehlungen für Monitoring- und Logging-Dienste

**Prometheus** und **Grafana** sind empfohlene Tools für das Monitoring, während der **ELK Stack** (Elasticsearch, Logstash, Kibana) für das Logging geeignet ist.

#### Prometheus Konfiguration

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - prometheus_data:/prometheus
    networks:
      - monitoring
    deploy:
      replicas: 1
      resources:
        limits:
          cpus: '0.5'
          memory: '512M'
      restart_policy:
        condition: on-failure

volumes:
  prometheus_data:
    driver: glusterfs
    driver_opts:
      voluri: "10.11.2.51:/gv0"
      mountpoint: "/mnt/glusterfs/prometheus_data"
```

#### Grafana Konfiguration

```yaml
services:
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - monitoring
    deploy:
      replicas: 1
      resources:
        limits:
          cpus: '0.5'
          memory: '512M'
      restart_policy:
        condition: on-failure

volumes:
  grafana_data:
    driver: glusterfs
    driver_opts:
      voluri: "10.11.2.51:/gv0"
      mountpoint: "/mnt/glusterfs/grafana_data"
```

### Integrationsempfehlungen

- **Prometheus:** Sammeln von Metriken aus den verschiedenen Diensten.
- **Grafana:** Visualisierung der gesammelten Metriken.
- **ELK Stack:** Zentralisiertes Logging zur Analyse und Fehlerbehebung.

## Rollback-Strategie

### Best Practices für Rollback und Update-Konfiguration

Im Falle eines fehlerhaften Deployments ist eine Rollback-Strategie entscheidend, um die Stabilität der Umgebung zu gewährleisten.

#### Update-Konfiguration

```yaml
deploy:
  update_config:
    parallelism: 1
    delay: 10s
    failure_action: rollback
  rollback_config:
    parallelism: 1
    delay: 10s
```

#### Best Practices

- **Failure Action:** Setzen Sie `failure_action` auf `rollback`, um bei Fehlern automatisch zum vorherigen Zustand zurückzukehren.
- **Parallelism:** Definieren Sie die Anzahl der gleichzeitig aktualisierten Container, um die Verfügbarkeit sicherzustellen.
- **Delay:** Fügen Sie Verzögerungen hinzu, um den Services Zeit zur Stabilisierung zu geben.

## Service Scaling

### Automatisches Skalieren der Dienste

Docker Swarm unterstützt das manuelle Skalieren von Diensten mittels `replicas`. Für automatisches Skalieren kann eine externe Lösung wie **Docker Swarm Autoscaler** verwendet werden.

#### Konfiguration von Replicas

```yaml
services:
  webserver:
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: '512M'
```

#### Empfehlungen

- **Replicas:** Starten Sie mit einer Basiszahl von Replikaten und passen Sie diese basierend auf der Last an.
- **Autoscaling:** Implementieren Sie einen externen Autoscaler, der die Anzahl der Replikate basierend auf Metriken wie CPU- und Speicherauslastung automatisch anpasst.

## Security Hardening

### Maßnahmen zur Absicherung der Umgebung

#### Container-Isolation

- **Namespaces und Cgroups:** Nutzen Sie Docker's eingebaute Isolationstechniken.
- **User Permissions:** Führen Sie Container nicht als Root aus.

#### Network-Policies

- **Firewall Regeln:** Beschränken Sie den Zugriff auf notwendige Ports.
- **Overlay Netzwerke:** Nutzen Sie verschlüsselte Overlay Netzwerke für die interne Kommunikation.

#### Zusätzliche Sicherheitsmechanismen

- **Image Scanning:** Überprüfen Sie Docker-Images regelmäßig auf Sicherheitslücken.
- **Secrets Management:** Nutzen Sie Docker Secrets für sensible Daten.
- **Updates:** Halten Sie Docker und die Container-Images stets auf dem neuesten Stand.

## Backup und Restore von Volumes

### Strategien zur Sicherung und Wiederherstellung

#### Backup-Strategien

- **Snapshot-basierte Backups:** Nutzen Sie GlusterFS's Snapshot-Funktionalität.
- **Regelmäßige Backups:** Planen Sie regelmäßige Backups der Volumes.

#### Restore-Strategien

- **Automatisierte Skripte:** Erstellen Sie Skripte, die bei Bedarf Volumes aus Backups wiederherstellen.
- **Redundante Storage-Nodes:** Implementieren Sie mehrere GlusterFS-Nodes zur Erhöhung der Ausfallsicherheit.

#### YML-Beispiel für Backup-Service

```yaml
services:
  backup:
    image: alpine:latest
    volumes:
      - paperless_data:/mnt/glusterfs/paperless_data
      - backup_volume:/backup
    command: ["sh", "-c", "cp -r /mnt/glusterfs/paperless_data /backup"]
    networks:
      - paperless_network
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure

volumes:
  backup_volume:
    driver: glusterfs
    driver_opts:
      voluri: "10.11.2.51:/gv0"
      mountpoint: "/mnt/glusterfs/backup_volume"
```

## Sonstiges

### Performanceverbesserungen

- **Caching:** Implementieren Sie Caching-Mechanismen wie Redis, um die Antwortzeiten zu verbessern.
- **Load Balancing:** Nutzen Sie Load Balancer, um den Traffic gleichmäßig auf die Replikate zu verteilen.
- **Optimierte Images:** Verwenden Sie schlanke Docker-Images, um die Startzeiten zu verkürzen.

### Sicherheitsmaßnahmen

- **Eindeutiges Bezeichnungsschema für Secrets:**
  - **Format:** `%SERVICE%_%SECRETNAME%`
  - **Beispiel:** `webserver_paperless_admin_password`
- **Erstellung von Secrets:**

  ```bash
  echo "*******" | docker secret create webserver_paperless_admin_password -
  ```

### Ausfallsicherheit

- **Redundante Services:** Stellen Sie sicher, dass kritische Dienste mehrfach repliziert sind.
- **Failover Mechanismen:** Implementieren Sie Mechanismen, die bei Ausfällen automatisch auf alternative Dienste umschalten.

### Implementierung von Health Checks

#### YML-Beispiel für Health Checks

```yaml
services:
  webserver:
    image: ghcr.io/paperless-ngx/paperless-ngx:latest
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      replicas: 2
```

#### Empfehlungen

- **Regelmäßige Prüfungen:** Definieren Sie regelmäßige Health Checks für alle kritischen Dienste.
- **Fehlerhafte Container neu starten:** Konfigurieren Sie Docker Swarm so, dass bei fehlgeschlagenen Health Checks automatisch neue Container gestartet werden.