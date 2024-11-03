# Node-Red Stack Documentation

## Funktionsbeschreibung des Stacks

Der Node-Red Stack ermöglicht die Bereitstellung und Verwaltung von **Node-RED**, einer visuellen Entwicklungsumgebung für das IoT und die Automatisierung, in einer Docker Swarm Umgebung. Node-RED erleichtert die Erstellung von Flows zur Integration verschiedener Dienste und Geräte durch eine benutzerfreundliche, browserbasierte Oberfläche. Der Stack nutzt **GlusterFS** als persistenten Speicher und **Portainer BE** zur einfachen Verwaltung der Container. Mit den konfigurierten Ressourcenlimits und Sicherheitsmaßnahmen stellt der Stack eine robuste und skalierbare Lösung für kleine bis mittlere Umgebungen bereit.

- **Offizielle Dokumentation:** [Node-RED Documentation](https://nodered.org/docs/)
- **Docker Hub:** [Node-RED on Docker Hub](https://hub.docker.com/r/nodered/node-red)

### Image-Informationen

| Image Name       | Version | Erstellungsdatum |
|------------------|---------|-------------------|
| nodered/node-red | 4.0.5   | 2024-10-14        |

## YML-Datei Optimierung und Überprüfung

### Best Practices Umsetzung

Die folgende optimierte YML-Datei für den Node-Red Stack wurde überprüft und nach Best Practices strukturiert. Alle unnötigen Parameter wurden entfernt, die Reihenfolge der Abschnitte angepasst und relevante Bereiche kommentiert. Zudem wurden Docker Secrets integriert und die Konfiguration verbessert.

### Optimierte `docker-compose.yml` für Node-Red

- docker-compose.yml: <https://github.com/pat26-dotcom/portainer-compose/blob/main/node-red/docker-compose.yml>

### Kommentare und Empfehlungen

- **Version:** Gibt die Version des Docker Compose-Formats an.
- **Services:** Definiert den Node-RED-Dienst mit seinen Konfigurationen.
  - **image:** Verwendet das Node-RED-Image in der Version 4.0.5.
  - **user:** Läuft unter einer nicht-privilegierten Benutzer-ID für erhöhte Sicherheit.
  - **environment:** Konfiguriert Node-RED mit Zeitzone, Projektaktivierung und Verwendung von Docker Secrets für das Admin-Passwort.
  - **secrets:** Verwendet Docker Secrets für sensible Informationen wie das Admin-Passwort.
  - **volumes:** Bindet persistente Speicher für Daten und optional den Docker-Socket für die Integration.
  - **ports:** Exponiert den Node-RED-Port 1880 auf dem Host.
  - **deploy:** Definiert die Bereitstellungsstrategie inklusive Ressourcenlimits, Neustart-Policy und Rollback-Strategie.
  - **healthcheck:** Überwacht die Verfügbarkeit des Node-RED-Dienstes.
  - **networks:** Verbindet den Dienst mit dem definierten Overlay-Netzwerk.
- **Networks:** Definiert ein Overlay-Netzwerk für die interne Kommunikation der Dienste.
- **Volumes:** Nutzt GlusterFS für persistente Speicher mit eindeutigem Namenskonzept.
- **Secrets:** Verwendet externe Docker Secrets für sichere Handhabung von Passwörtern.

**Empfehlung zu Abhängigkeiten:**

- Da Node-RED in diesem Stack keine direkten Abhängigkeiten zu anderen Diensten hat, ist keine zusätzliche `depends_on`-Konfiguration erforderlich. Sollten zukünftig weitere Dienste hinzugefügt werden, die von Node-RED abhängen, sollte `depends_on` verwendet werden, um die Startreihenfolge zu steuern.

## Ressourcenlimits

### Überprüfung und Empfehlungen

Basierend auf den bereitgestellten Ressourcen (3 Nodes, je 16 GB RAM und 2 CPUs) sowie den Best Practices für Docker Swarm, sind die folgenden Ressourcenlimits für Node-RED empfohlen:

| Dienst    | CPUs | RAM | Beschreibung                           |
|-----------|------|-----|----------------------------------------|
| Node-Red  | 0.5  | 1G  | Visuelle Entwicklungsumgebung mit moderatem Traffic |

**Allgemeine Best Practices:**

- **Limits setzen:** Ressourcenlimits verhindern, dass der Node-RED-Dienst die gesamten verfügbaren Ressourcen eines Nodes beansprucht.
- **Reserve:** Sicherstellen, dass genügend Ressourcen für das Betriebssystem und andere laufende Prozesse verfügbar sind.
- **Monitoring:** Kontinuierliches Überwachen der Ressourcennutzung und Anpassung der Limits bei Bedarf.

## Storage

### GlusterFS Konfiguration

Der Node-Red Stack nutzt **GlusterFS** als persistenten Speicher. Das Docker Plugin [glusterfs-volume](https://github.com/chrisbecke/glusterfs-volume) ist installiert und erstellt die benötigten Volumes automatisch im Verzeichnis **/mnt/glusterfs**. Der verwendete Treibername lautet **"glusterfs"**.

#### Volume Namenskonzept

- **Format:** `%STACKNAME%_%VOLUMENAME%`
- **Beispiel:** Für den Stack "node-red" und das Volume "data" lautet der Volume-Name **"data"** und der Mountpoint **"/mnt/glusterfs/node-red_data"**.

#### YML-Beispiel für Volumes

```yaml
volumes:
  data:
    driver: glusterfs
    driver_opts:
      driver: glusterfs
      voluri: "localhost:/gv0"
      mountpoint: "/mnt/glusterfs/node-red_data"
```

**Hinweis:** In der YML-Datei wird nur das Suffix des Volumenamens verwendet, ohne den Stack- oder Containernamen.

## Secrets

### Namenskonzept und Erstellung

Bei der Verwendung von **Docker Secrets** wird ein durchgängiges und eindeutiges Namenskonzept angewendet, um die Secrets für mehrere Container eindeutig zu identifizieren.

#### Namensschema

- **Format:** `%SERVICE%_%SECRETNAME%`
- **Beispiel:** Für den Dienst "node-red" und das Secret "admin_password" lautet der Secret-Name **"node_red_admin_password"**.

#### Erstellung von Secrets

```bash
echo "mein_sicheres_admin_passwort" | docker secret create node_red_admin_password -
```

#### YML-Beispiel

```yaml
secrets:
  node_red_admin_password:
    external: true
```

## Netzwerk

### Port-Konfiguration und Best Practices

#### Überprüfung der Ports

Stellen Sie sicher, dass nur die notwendigen Ports nach außen freigegeben werden, um die Sicherheit zu erhöhen und Portkonflikte zu vermeiden.

#### Empfohlene Ports

| Dienst    | Interner Port | Externer Port | Beschreibung         |
|-----------|---------------|---------------|----------------------|
| node-red  | 1880          | 1880          | Node-RED API Zugriff |

#### Netzwerk-Typen

- **Overlay Netzwerke:** Für die Kommunikation zwischen den Services auf verschiedenen Nodes.
- **Ingress Netzwerke:** Für den externen Zugriff auf die Dienste.

#### YML-Beispiel für Netzwerke

```yaml
networks:
  node_red_network:
    driver: overlay
    name: swarm_node_red_network
    attachable: true
```
