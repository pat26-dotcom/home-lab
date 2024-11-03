# Dokumentation Homepage Stack

## YML-Datei

Die folgende optimierte Compose-Datei für den Stack **Homepage** wurde überprüft und verbessert. Alle unnötigen Parameter wurden entfernt, die Struktur entspricht den Best Practices, und die Lesbarkeit wurde erhöht. Relevante Bereiche sind kommentiert. Passwörter und sensible Daten verbleiben in den Umgebungsvariablen gemäß den Benutzeranforderungen.

- docker-compose.yml: <https://github.com/pat26-dotcom/portainer-compose/blob/main/homepage/docker-compose.yml>

### Änderungen und Optimierungen

- **Entfernung von Traefik-Parametern:** In der ursprünglichen Compose-Datei waren keine Traefik-Parameter vorhanden, daher waren keine Änderungen in diesem Bereich erforderlich.
  
- **Beibehaltung von Passwörtern:** Passwörter und API-Schlüssel bleiben in den Umgebungsvariablen, wie vom Benutzer gewünscht. Es wird empfohlen, den Zugriff auf die Compose-Datei entsprechend zu sichern, um unbefugten Zugriff zu verhindern.
  
- **Hinzufügen von Health Checks:** Ein Health Check wurde hinzugefügt, um die Verfügbarkeit des Dienstes zu überwachen und automatisch fehlerhafte Container neu zu starten.
  
- **Rollback-Strategie:** Die `rollback_config` wurde ergänzt, um im Falle eines fehlerhaften Deployments automatisch auf die vorherige Version zurückzurollen.
  
- **Kommentierung:** Relevante Bereiche der YML-Datei wurden kommentiert, um die Verständlichkeit zu erhöhen.

## Ressourcenlimits

### Überprüfung und Empfehlungen

Die Ressourcenlimits für den **Homepage**-Dienst wurden basierend auf den allgemeinen Best Practices von Docker Swarm und den Anforderungen typischer Anwendungen festgelegt.

| Parameter           | Wert | Beschreibung                                   |
|---------------------|------|------------------------------------------------|
| CPU-Limit           | 1.0  | Maximale CPU-Nutzung pro Container             |
| Speicherlimit       | 2G   | Maximale Speicherzuweisung pro Container       |
| CPU-Reservierung    | 0.5  | Garantierte CPU-Ressourcen pro Container       |
| Speicherreservierung| 1G   | Garantierte Speicherressourcen pro Container   |

### Best Practices von Docker Swarm

- **Setzen von CPU- und Speicherlimits:** Sowohl Limits als auch Reservierungen wurden definiert, um eine faire Ressourcenzuweisung zu gewährleisten und Ressourcenengpässe zu vermeiden.
- **Überwachung der Ressourcennutzung:** Es wird empfohlen, die Ressourcennutzung regelmäßig zu überwachen und die Limits bei Bedarf anzupassen.
- **Verwendung restriktiver Limits:** Beginnend mit restriktiven Limits können diese bei Bedarf erhöht werden, um die Stabilität der Umgebung zu gewährleisten.

## Storage

GlusterFS wird als persistenter Speicher für den **Homepage**-Dienst eingesetzt. Die Volumen sind gemäß dem vorgegebenen Namenskonzept konfiguriert.

### Volume-Namenskonzept

Der Mountpoint folgt dem Muster:

```bash
/mnt/glusterfs/%STACKNAME%_%VOLUMENAME%
```

Für den Stack **Homepage** und das Volume **config**:

```bash
/mnt/glusterfs/homepage_config
```

### YML-Konfiguration für Volumes

```yaml
volumes:
  config:
    driver: glusterfs
    driver_opts:
      voluri: "localhost:/gv0"
      mountpoint: "/mnt/glusterfs/homepage_config"
```

**Hinweis:** Der Stack- oder Containername erscheint nicht im Volumennamen in der YML-Datei, um Dopplungen zu vermeiden.

## Netzwerk

Die Netzwerkports wurden gemäß den Best Practices im Docker Swarm Umfeld konfiguriert, um Flexibilität und Sicherheit zu gewährleisten.

### Portkonfiguration

| Dienst        | Interner Port | Externer Port | Beschreibung                          |
|---------------|---------------|---------------|---------------------------------------|
| Homepage      | 3000          | 3001          | Hauptanwendung HTTP-Traffic           |
| Docker Socket | N/A           | N/A           | Interne Docker-Integration (nicht nach außen freigegeben) |

### Best Practices

- **Minimierung der freigegebenen Ports:** Nur notwendige Ports werden nach außen freigegeben, um die Angriffsfläche zu reduzieren.
- **Verwendung eines Overlay-Netzwerks:** Das Overlay-Netzwerk `homepage_network` sorgt für eine sichere Kommunikation zwischen den Diensten innerhalb des Swarms.
- **Vermeidung von Portkonflikten:** Externe Ports wurden sorgfältig gewählt, um Konflikte mit anderen Diensten zu vermeiden.
