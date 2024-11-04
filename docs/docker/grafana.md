# Grafana Stack Dokumentation

## Funktionsbeschreibung

Der Grafana Stack dient zur Überwachung und Visualisierung von Metriken und Logs innerhalb der Docker Swarm Umgebung. Grafana ermöglicht die Erstellung von Dashboards zur Echtzeit-Analyse und -Überwachung der Systemleistung sowie der Applikationen. In dieser Konfiguration wird Grafana als einzelner Dienst in einem Docker Swarm Cluster mit GlusterFS als persistentem Speicher eingesetzt. Die Verwaltung der Umgebung erfolgt über Portainer BE, wodurch eine einfache Handhabung und hohe Verfügbarkeit gewährleistet wird.

Weitere Informationen finden Sie in der offiziellen Dokumentation:

- [Offizielle Grafana Dokumentation](https://grafana.com/docs/)
- [Grafana Docker Hub](https://hub.docker.com/r/grafana/grafana)

## Image Details

| Image Name         | Version          | Erstellungsdatum    |
|--------------------|------------------|---------------------|
| grafana/grafana    | 11.2.2-ubuntu    | 2024-04-27          |

---

## YAML-Datei Optimierung

Die folgende `docker-compose.yml` Datei für den Grafana Stack wurde optimiert, um Fehler zu beheben, doppelte Einträge zu entfernen und den Best Practices von Docker Swarm zu entsprechen. Alle unnötigen Parameter wurden entfernt, die Lesbarkeit wurde verbessert und relevante Bereiche wurden kommentiert.

### Optimierte `docker-compose.yml`

- docker-compose.yml: <https://github.com/pat26-dotcom/portainer-compose/blob/main/grafana/docker-compose.yml>

### Empfehlungen zu Service-Abhängigkeiten

Da der Grafana Stack in diesem Fall nur aus einem Dienst besteht, bestehen keine direkten Abhängigkeiten zu anderen Services innerhalb des Stacks. Sollten in Zukunft weitere Dienste integriert werden, wie beispielsweise eine Datenbank oder Prometheus, empfiehlt es sich, Abhängigkeiten entsprechend zu definieren oder sicherzustellen, dass Dienste ihre Abhängigkeiten selbstständig prüfen.

```yaml
services:
  grafana:
    depends_on:
      - prometheus # Beispiel für zukünftige Abhängigkeit
```

## Ressourcenlimits

### Empfohlene Ressourcenlimits

Basierend auf den allgemeinen Best Practices und den Empfehlungen der Grafana Entwickler für kleine bis mittlere Umgebungen, werden folgende Ressourcenlimits vorgeschlagen:

| Dienst  | CPUs | RAM   |
|---------|------|-------|
| grafana | 1.0  | 1G    |

Diese Limits sollten basierend auf der tatsächlichen Nutzung und den spezifischen Anforderungen von Grafana angepasst werden.

### Best Practices für Docker Swarm

- **Setzen von Limits und Reservierungen**: Definieren Sie sowohl `limits` als auch `reservations`, um sicherzustellen, dass der Grafana Dienst genügend Ressourcen hat und keine anderen Dienste übermäßig Ressourcen beanspruchen.
- **Monitoring**: Überwachen Sie die Ressourcennutzung kontinuierlich, um Engpässe frühzeitig zu erkennen und Anpassungen vorzunehmen.

## Storage

### GlusterFS Konfiguration

GlusterFS wird als persistenter Speicher für Grafana verwendet. Das Docker Plugin [glusterfs-volume](https://github.com/chrisbecke/glusterfs-volume) ist installiert und erstellt Volumes automatisch im Verzeichnis `/mnt/glusterfs`. Der verwendete Treibername lautet `glusterfs`.

!!! note annotate "Gluster FS"

#### Volume Namenskonzept

- **Format**: `%STACKNAME%_%VOLUMENAME%`
- **Beispiel**: Für den Stack `grafana` und das Volume `data` wird der Mountpoint `/mnt/glusterfs/grafana_data` verwendet.

#### YAML-Konfiguration

```yaml
volumes:
  grafana_data:
    driver: glusterfs
    driver_opts:
      voluri: "localhost:/gv0"
      mountpoint: "/mnt/glusterfs/grafana_data"
```

*Hinweis*: Verwenden Sie nur das Suffix des Volumennamens ohne den Stack- oder Containernamen.

## Secrets

### Namenskonzept für Docker Secrets

Verwenden Sie ein durchgängiges und eindeutiges Namensschema, um Secrets eindeutig zu identifizieren. Ein empfohlenes Schema ist:

```bash
<stackname>_<service>_<secretname>
```

**Beispiel**:

- `grafana_admin_password`

### Erstellung von Secrets

1. Erstellen Sie die Secret-Datei:

    ```bash
    echo "sicheres_passwort" | docker secret create grafana_admin_password -
    ```

2. Referenzieren Sie das Secret in der `docker-compose.yml`:

    ```yaml
    secrets:
      - grafana_admin_password
    ```

3. Aktualisieren Sie die `environment` Variablen, um das Secret zu verwenden:

    ```yaml
    environment:
      - GF_SECURITY_ADMIN_PASSWORD_FILE=/run/secrets/grafana_admin_password
    secrets:
      - grafana_admin_password
    ```

## Netzwerk

### Port-Konfiguration

Überprüfen Sie die erforderlichen Ports für Grafana und stellen Sie sicher, dass nur notwendige Ports nach außen freigegeben werden. Verwenden Sie ein Overlay-Netzwerk für die interne Kommunikation.

#### Beispielhafte Portfreigabe

| Dienst  | Interner Port | Externer Port |
|---------|---------------|---------------|
| grafana | 3000          | 3000          |

### Best Practices

- **Minimieren der offenen Ports**: Öffnen Sie nur die Ports, die unbedingt benötigt werden.
- **Vermeidung von Portkonflikten**: Verwenden Sie konsistente Portzuweisungen und überprüfen Sie vorhandene Portnutzungen.
- **Netzwerksegmentierung**: Nutzen Sie separate Netzwerke für unterschiedliche Dienstgruppen, um die Sicherheit zu erhöhen.
