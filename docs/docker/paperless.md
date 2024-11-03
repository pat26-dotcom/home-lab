# Paperless NGX Stack Dokumentation

## Funktionsbeschreibung des Stacks

Der **Paperless** Stack ermöglicht die digitale Verwaltung und Archivierung von Dokumenten durch die Automatisierung von Dokumenten-Scans, OCR (Optical Character Recognition) und die Organisation in einer durchsuchbaren Datenbank. Der Stack nutzt **Docker Swarm** zur Orchestrierung und **GlusterFS** als persistenten Speicher, wodurch eine hochverfügbare und skalierbare Umgebung geschaffen wird. **Portainer BE** dient als Verwaltungsoberfläche zur einfachen Handhabung und Überwachung der Container-Dienste über die drei Nodes hinweg.

- **Offizielle Dokumentation:** [Paperless-ngx Documentation](https://paperless-ngx.readthedocs.io/)
- **Docker Hub:** [Paperless-ngx on Docker Hub](https://hub.docker.com/r/paperlessngx/paperless-ngx)

### Image-Informationen

| Image Name                     | Version | Erstellungsdatum |
|--------------------------------|---------|-------------------|
| redis                          | 7       | 2024-04-27        |
| postgres                       | 16      | 2024-04-27        |
| ghcr.io/paperless-ngx/paperless-ngx | latest  | 2024-04-27        |
| glusterfs-volume               | v1.2.3  | 2024-04-27        |

## YML-Datei Optimierung und Überprüfung

### Optimierte `docker-compose.yml` für den Stack "Paperless"

<https://github.com/pat26-dotcom/portainer-compose/blob/main/paperless/docker-compose.yml>

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
      voluri: "localhost:/gv0"
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
