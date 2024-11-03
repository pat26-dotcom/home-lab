# Jellyfin Stack Dokumentation

## Funktionsbeschreibung des Stacks

Der **Jellyfin** Stack stellt einen kostenlosen und quelloffenen Media-Server bereit, der es ermöglicht, Medieninhalte wie Filme, Serien, Musik und Fotos zentral zu speichern und auf verschiedenen Geräten zu streamen. Der Stack nutzt **Docker Swarm** zur Orchestrierung und **GlusterFS** als persistenten Speicher, wodurch eine hochverfügbare und skalierbare Umgebung geschaffen wird. **Portainer BE** dient als Verwaltungsoberfläche zur einfachen Handhabung und Überwachung der Container-Dienste über die drei Nodes hinweg.

- **Offizielle Dokumentation:** [Jellyfin Documentation](https://jellyfin.org/docs/)
- **Docker Hub:** [Jellyfin on Docker Hub](https://hub.docker.com/r/jellyfin/jellyfin)

### Image-Informationen

| Image Name         | Version | Erstellungsdatum |
|--------------------|---------|-------------------|
| jellyfin/jellyfin  | latest  | 2024-04-27        |
| glusterfs-volume   | v1.2.3  | 2024-04-27        |

## YML-Datei Optimierung und Überprüfung

### Optimierte `docker-compose.yml` für den Stack "Jellyfin"

- docker-compose.yml: <https://github.com/pat26-dotcom/portainer-compose/blob/main/jellyfin/docker-compose.yml>

### Kommentare und Empfehlungen

- **Version:** Gibt die Version des Docker Compose-Formats an.
- **Services:** Definiert den Jellyfin Container mit seinen Konfigurationen.
  - **image:** Verwendet das aktuelle Jellyfin-Image von Docker Hub.
  - **restart:** Stellt sicher, dass der Container neu startet, außer er wird explizit gestoppt.
  - **user:** Setzt den Benutzer und die Gruppe innerhalb des Containers für bessere Sicherheit.
  - **ports:** Bindet den internen Jellyfin-Port (8096) an den externen Port (8096).
  - **volumes:** Bindet die notwendigen Volumes für Konfiguration, Cache und Medien.
  - **environment:** Setzt Umgebungsvariablen für die Konfiguration des Servers.
  - **extra_hosts:** Fügt zusätzliche Host-Einträge hinzu, die für bestimmte Netzwerkfunktionen erforderlich sein können.
  - **deploy:** Enthält Deployment-Konfigurationen wie Replikate, Ressourcenlimits und Restart-Policies.
  - **healthcheck:** Überprüft die Verfügbarkeit des Jellyfin-Servers und stellt sicher, dass fehlerhafte Container neu gestartet werden.
- **Networks:** Definiert das Overlay-Netzwerk `jellyfin_network` für die interne Kommunikation der Dienste.
- **Volumes:** Definiert die persistenten Speichervolumes unter Verwendung von GlusterFS gemäß dem Namenskonzept.

**Empfehlung zu Abhängigkeiten:**

- Jellyfin hat keine direkten Abhängigkeiten zu anderen Diensten innerhalb dieses Stacks. Stellen Sie jedoch sicher, dass das GlusterFS-Volume ordnungsgemäß funktioniert, um Datenverluste zu vermeiden.

## Ressourcenlimits

### Überprüfung und Empfehlungen

Basierend auf den bereitgestellten Ressourcen (3 Nodes, je 16GB RAM und 2 CPUs) sowie den Best Practices für Docker Swarm, wurden die folgenden Ressourcenlimits empfohlen:

| Dienst   | CPUs  | RAM  | Beschreibung                            |
|----------|-------|------|-----------------------------------------|
| jellyfin | 1.0   | 2G   | Media-Server für Streaming und Verwaltung|

**Allgemeine Best Practices:**

- **Limits setzen:** Ressourcenlimits verhindern, dass ein einzelner Dienst die gesamte Ressourcenlast übernimmt.
- **Reserve:** Stellen Sie sicher, dass genügend Ressourcen für das Betriebssystem und andere laufende Prozesse verfügbar sind.
- **Monitoring:** Überwachen Sie die Ressourcennutzung kontinuierlich und passen Sie die Limits bei Bedarf an.

## Storage

### GlusterFS Konfiguration

Der Stack nutzt **GlusterFS** als persistenten Speicher. Das Docker Plugin [glusterfs-volume](https://github.com/chrisbecke/glusterfs-volume) ist installiert und erstellt die benötigten Volumes automatisch im Verzeichnis **/mnt/glusterfs**. Der verwendete Treibername lautet **"glusterfs"**.

#### Volume Namenskonzept

- **Format:** `%STACKNAME%_%VOLUMENAME%`
- **Beispiel:** Für den Stack "jellyfin" und das Volume "config" lautet der Volume-Name **"jellyfin_config"**.

#### YML-Beispiel für Volumes

```yaml
volumes:
  jellyfin_config:
    driver: glusterfs
    driver_opts:
      voluri: "10.11.2.51:/gv0"
      mountpoint: "/mnt/glusterfs/jellyfin_config"
```

**Hinweis:** In der YML-Datei wird nur das Suffix des Volumenamens verwendet, ohne den Stack- oder Containernamen.

## Secrets

### Namenskonzept und Erstellung

Bei der Verwendung von **Docker Secrets** wird ein durchgängiges und eindeutiges Namenskonzept angewendet, um die Secrets für mehrere Container eindeutig zu identifizieren.

#### Namensschema

- **Format:** `%SERVICE%_%SECRETNAME%`
- **Beispiel:** Für den Dienst "jellyfin" und das Secret "admin_password" lautet der Secret-Name **"jellyfin_admin_password"**.

#### Erstellung von Secrets

```bash
echo "IhrSicheresPasswort" | docker secret create jellyfin_admin_password -
```

#### YML-Beispiel

```yaml
secrets:
  jellyfin_admin_password:
    external: true
```

**Anpassung im YML:**

Passen Sie die `environment`-Variablen an, um Secrets zu verwenden:

```yaml
environment:
  - JELLYFIN_PublishedServerUrl_FILE=/run/secrets/jellyfin_admin_password
```

Und binden Sie das Secret im Dienst:

```yaml
secrets:
  - jellyfin_admin_password
```

## Netzwerk

### Port-Konfiguration und Best Practices

#### Überprüfung der Ports

Stellen Sie sicher, dass nur die notwendigen Ports nach außen freigegeben werden, um die Sicherheit zu erhöhen und Portkonflikte zu vermeiden.

#### Empfohlene Ports

| Dienst   | Interner Port | Externer Port | Beschreibung                   |
|----------|---------------|---------------|--------------------------------|
| jellyfin | 8096          | 8096          | Jellyfin Media Server          |
| portainer| 9000          | 9000          | Portainer Management Interface |

#### Netzwerk-Typen

- **Overlay Netzwerke:** Für die Kommunikation zwischen den Services auf verschiedenen Nodes.
- **Ingress Netzwerke:** Für den externen Zugriff auf die Dienste.

#### YML-Beispiel für Netzwerke

```yaml
networks:
  jellyfin_network:
    driver: overlay
```
