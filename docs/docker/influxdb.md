# InfluxDB Stack Documentation

## Funktionsbeschreibung des Stacks

Der **InfluxDB Stack** ermöglicht die Bereitstellung und Verwaltung einer hochleistungsfähigen Zeitreihen-Datenbank in einer Docker Swarm Umgebung. InfluxDB ist ideal für die Speicherung und Analyse von Zeitreihendaten, wie sie in Überwachungs-, IoT- und Echtzeitanwendungen vorkommen. Der Stack nutzt **GlusterFS** als persistenten Speicher und **Portainer BE** zur einfachen Verwaltung der Container. Mit den konfigurierten Ressourcenlimits und Sicherheitsmaßnahmen stellt der Stack eine robuste und skalierbare Lösung für kleine bis mittlere Umgebungen bereit.

- **Offizielle Dokumentation:** [InfluxDB Documentation](https://docs.influxdata.com/influxdb/)
- **Docker Hub:** [InfluxDB on Docker Hub](https://hub.docker.com/_/influxdb)

### Image-Informationen

| Image Name | Version | Erstellungsdatum |
|------------|---------|-------------------|
| influxdb   | 2.7.10  | 2024-10-14        |

## YML-Datei Optimierung und Überprüfung

### Best Practices Umsetzung

Die folgende optimierte YML-Datei für den InfluxDB Stack wurde überprüft und nach Best Practices strukturiert. Alle unnötigen Parameter wurden entfernt, die Reihenfolge der Abschnitte angepasst und relevante Bereiche kommentiert.

### Optimierte `docker-compose.yml` für InfluxDB

- docker-compose.yml: <https://github.com/pat26-dotcom/portainer-compose/blob/main/influxdb/docker-compose.yml>

### Kommentare und Empfehlungen

- **Version:** Gibt die Version des Docker Compose-Formats an.
- **Services:** Definiert den InfluxDB-Dienst mit seinen Konfigurationen.
  - **image:** Verwendet das InfluxDB-Image in der Version 2.7.10.
  - **user:** Läuft unter einer nicht-privilegierten Benutzer-ID für erhöhte Sicherheit.
  - **environment:** Konfiguriert InfluxDB initial mit Benutzername, Passwort und Admin-Token über Docker Secrets.
  - **secrets:** Verwendet Docker Secrets für sensible Informationen.
  - **volumes:** Bindet persistente Speicher für Daten und Konfiguration.
  - **ports:** Exponiert den InfluxDB-Port 8086 auf dem Host.
  - **deploy:** Definiert die Bereitstellungsstrategie inklusive Ressourcenlimits, Neustart-Policy und Rollback-Strategie.
  - **healthcheck:** Überwacht die Verfügbarkeit des InfluxDB-Dienstes.
  - **networks:** Verbindet den Dienst mit dem definierten Overlay-Netzwerk.
- **Networks:** Definiert ein Overlay-Netzwerk für die interne Kommunikation der Dienste.
- **Volumes:** Nutzt GlusterFS für persistente Speicher mit eindeutigem Namenskonzept.
- **Secrets:** Verwendet externe Docker Secrets für sichere Handhabung von Passwörtern und Tokens.

**Empfehlung zu Abhängigkeiten:**

- Da InfluxDB in diesem Stack keine direkten Abhängigkeiten zu anderen Diensten hat, ist keine zusätzliche `depends_on`-Konfiguration erforderlich. Sollten zukünftig weitere Dienste hinzugefügt werden, die von InfluxDB abhängen, sollte `depends_on` verwendet werden, um die Startreihenfolge zu steuern.

## Ressourcenlimits

### Überprüfung und Empfehlungen

Basierend auf den bereitgestellten Ressourcen (3 Nodes, je 16 GB RAM und 2 CPUs) sowie den Best Practices für Docker Swarm, sind die folgenden Ressourcenlimits für InfluxDB empfohlen:

| Dienst    | CPUs | RAM  | Beschreibung                          |
|-----------|------|------|---------------------------------------|
| InfluxDB2 | 1.0  | 2G   | Zeitreihen-Datenbank mit moderatem Datenvolumen |

**Allgemeine Best Practices:**

- **Limits setzen:** Ressourcenlimits verhindern, dass der InfluxDB-Dienst die gesamten verfügbaren Ressourcen eines Nodes beansprucht.
- **Reserve:** Sicherstellen, dass genügend Ressourcen für das Betriebssystem und andere laufende Prozesse verfügbar sind.
- **Monitoring:** Kontinuierliches Überwachen der Ressourcennutzung und Anpassung der Limits bei Bedarf.

## Storage

### GlusterFS Konfiguration

Der InfluxDB Stack nutzt **GlusterFS** als persistenten Speicher. Das Docker Plugin [glusterfs-volume](https://github.com/chrisbecke/glusterfs-volume) ist installiert und erstellt die benötigten Volumes automatisch im Verzeichnis **/mnt/glusterfs**. Der verwendete Treibername lautet **"glusterfs"**.

#### Volume Namenskonzept

- **Format:** `%STACKNAME%_%VOLUMENAME%`
- **Beispiel:** Für den Stack "influxdb" und das Volume "data" lautet der Volume-Name **"influxdb_data"**.

#### YML-Beispiel für Volumes

```yaml
volumes:
  influxdb_data:
    driver: glusterfs
    driver_opts:
      driver: glusterfs
      voluri: "localhost:/gv0"
      mountpoint: "/mnt/glusterfs/influxdb_data"

  influxdb_config:
    driver: glusterfs
    driver_opts:
      driver: glusterfs
      voluri: "localhost:/gv0"
      mountpoint: "/mnt/glusterfs/influxdb_config"
```

**Hinweis:** In der YML-Datei wird nur das Suffix des Volumenamens verwendet, ohne den Stack- oder Containernamen.

## Secrets

### Namenskonzept und Erstellung

Bei der Verwendung von **Docker Secrets** wird ein durchgängiges und eindeutiges Namenskonzept angewendet, um die Secrets für mehrere Container eindeutig zu identifizieren.

#### Namensschema

- **Format:** `%SERVICE%_%SECRETNAME%`
- **Beispiel:** Für den Dienst "influxdb" und das Secret "password" lautet der Secret-Name **"influxdb_init_password"**.

#### Erstellung von Secrets

```bash
echo "mein_sicheres_passwort" | docker secret create influxdb_init_password -
echo "mein_admin_token" | docker secret create influxdb_admin_token -
```

#### YML-Beispiel

```yaml
secrets:
  influxdb_init_password:
    external: true
  influxdb_admin_token:
    external: true
```

## Netzwerk

### Port-Konfiguration und Best Practices

#### Überprüfung der Ports

Stellen Sie sicher, dass nur die notwendigen Ports nach außen freigegeben werden, um die Sicherheit zu erhöhen und Portkonflikte zu vermeiden.

#### Empfohlene Ports

| Dienst   | Interner Port | Externer Port | Beschreibung                  |
|----------|---------------|---------------|-------------------------------|
| influxdb | 8086          | 8086          | InfluxDB API Zugriff          |
| portainer| 9000          | 9000          | Portainer Management Interface|

#### Netzwerk-Typen

- **Overlay Netzwerke:** Für die Kommunikation zwischen den Services auf verschiedenen Nodes.
- **Ingress Netzwerke:** Für den externen Zugriff auf die Dienste.

#### YML-Beispiel für Netzwerke

```yaml
networks:
  influxdb_network:
    driver: overlay
    name: swarm_influxdb_network
```
