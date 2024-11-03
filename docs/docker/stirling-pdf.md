# Stirling PDF Dokumentation

## Funktionsbeschreibung des Stacks

Der **Stirling PDF** Docker Swarm Stack bietet eine umfassende Lösung zur Verarbeitung und Verwaltung von PDF-Dokumenten. Er ermöglicht das Hinzufügen zusätzlicher OCR-Sprachen und die Nutzung benutzerdefinierter Konfigurationen. Der Stack umfasst einen PDF-Dienst, der auf dem `frooodle/s-pdf:latest` Image basiert und über spezifische Umgebungsvariablen sowie persistente Speicher für OCR-Daten, Konfigurationen und benutzerdefinierte Dateien verfügt.

Weitere Informationen finden Sie in der [offiziellen Stirling PDF Dokumentation](https://github.com/frooodle/s-pdf) und auf dem [Docker Hub](https://hub.docker.com/r/frooodle/s-pdf).

### Image Übersicht

| Image Name         | Version | Erstellungsdatum |
|--------------------|---------|-------------------|
| frooodle/s-pdf     | latest  | 2024-04-27        |

## Optimierte YML-Datei

- docker-compose.yml: <https://github.com/pat26-dotcom/portainer-compose/blob/main/stirling-pdf/docker-compose.yml>

### Empfehlungen zu Service-Abhängigkeiten

- **Unabhängigkeit prüfen:** Der Stirling PDF Dienst ist unabhängig und benötigt keine anderen Dienste zum Starten.
- **Startreihenfolge festlegen:** Da keine direkten Abhängigkeiten bestehen, kann der Dienst jederzeit gestartet werden.

## Ressourcenlimits

### Empfohlene Ressourcenlimits

Basierend auf den bereitgestellten Node-Ressourcen (16GB RAM und 2 CPUs pro Node) und allgemeinen Best Practices für Docker Swarm, werden folgende Ressourcenlimits empfohlen:

| Service         | CPU Limit | RAM Limit |
|-----------------|-----------|-----------|
| stirling-pdf    | 0.5       | 512M      |

### Best Practices

- **Setzen von Limits:** Definieren Sie CPU- und RAM-Limits für jeden Service, um Ressourcenübernutzung zu verhindern.
- **Überwachung:** Nutzen Sie Monitoring-Tools, um die Ressourcenauslastung kontinuierlich zu überwachen und anzupassen.
- **Reserve Ressourcen:** Stellen Sie sicher, dass genügend Ressourcen für das Betriebssystem und die Docker Swarm Verwaltung reserviert sind.

## Storage

### GlusterFS Konfiguration

- **Docker Plugin:** [glusterfs-volume](https://github.com/chrisbecke/glusterfs-volume)
- **Treibername:** `glusterfs`
- **Mountpoint-Verzeichnis:** `/mnt/glusterfs`
- **Naming-Konzept:**
  - **Volume:** `%STACKNAME%_%VOLUMENAME%`
  - **Beispiel:** Für den Stack `stirling_pdf` und das Volume `trainingData` wird das Verzeichnis `stirling_pdf_trainingData` verwendet.

### Beispiel Volume Definition in YML

```yaml
volumes:
  stirling_pdf_trainingData:
    driver: glusterfs
    driver_opts:
      driver: glusterfs
      voluri: "localhost:/gv0"
      mountpoint: "/mnt/glusterfs/stirling_pdf_trainingData"  # Mountpoint für OCR-Daten

  stirling_pdf_extraConfigs:
    driver: glusterfs
    driver_opts:
      driver: glusterfs
      voluri: "localhost:/gv0"
      mountpoint: "/mnt/glusterfs/stirling_pdf_extraConfigs"  # Mountpoint für zusätzliche Konfigurationen

  stirling_pdf_customFiles:
    driver: glusterfs
    driver_opts:
      driver: glusterfs
      voluri: "localhost:/gv0"
      mountpoint: "/mnt/glusterfs/stirling_pdf_customFiles"  # Mountpoint für benutzerdefinierte Dateien
```

### Empfehlungen

- **Eindeutige Namen:** Verwenden Sie eindeutige und leserliche Volumennamen, um Dopplungen zu vermeiden.
- **Automatisierung:** Nutzen Sie das GlusterFS Docker Plugin zur automatischen Erstellung und Verwaltung der Volumes.

## Secrets

### Namenskonzept für Docker Secrets

Das Namensschema für Docker Secrets folgt dem Muster: `%STACK%_%SERVICE%_%SECRET%`

### Beispiel

- **Initial-Login Passwort für Stirling PDF:** `stirling_pdf_security_initiallogin_password`

### Erstellung von Secrets

```bash
echo "securestirlingpdfpassword" | docker secret create stirling_pdf_security_initiallogin_password -
```

### Verwendung in der YML-Datei

```yaml
secrets:
  stirling_pdf_security_initiallogin_password:
    external: true
```

## Netzwerk

### Ports Überprüfung und Konfiguration

- **Interne Ports:** Nur notwendige interne Ports sind definiert, um die Kommunikation zwischen den Services zu ermöglichen.
- **Externe Ports:** Der Stirling PDF Dienst exponiert den Port `8087` nach außen, um den Zugriff auf die PDF-Verarbeitungsschnittstelle zu ermöglichen.

### Beispiel Ports Definition

```yaml
services:
  stirling-pdf:
    ports:
      - "8087:8080"  # HTTP Zugriff auf Stirling PDF
```

### Best Practices

- **Port-Mapping:** Verwenden Sie eindeutige externe Ports, um Konflikte zu vermeiden.
- **Firewall-Regeln:** Implementieren Sie Firewall-Regeln, um den Zugriff auf notwendige Ports zu beschränken.
- **Netzwerksegmente:** Nutzen Sie verschiedene Netzwerke (z.B. `paperless_network`), um den Datenverkehr zu segmentieren und zu sichern.
