# Docker Swarm Umgebung Dokumentation: 

## Einleitung

Diese Dokumentation beschreibt die Einrichtung einer hochverfügbaren Docker Swarm Umgebung mit den folgenden Stacks:

- **Traefik** als Reverse Proxy für externen Zugriff
- **HAProxy** als Load Balancer für internen Zugriff
- **Grafana** für Visualisierungen
- **InfluxDB** als Zeitreihendatenbank
- **UniFi** für Netzwerkinfrastrukturverwaltung

Die Umgebung wird auf **virtuellen Servern** in einer **VMware vSphere 8** Umgebung betrieben. Zusätzlich wird **GlusterFS** als verteiltes Dateisystem für die persistente Speicherung der Daten verwendet. Die Konfiguration erfolgt im **Split DNS Modus**, sodass Dienste sowohl intern (`*.local.lezit.de`) als auch extern (`*.lezit.de`) erreichbar sind. Zudem wird beschrieben, wie ein **hochverfügbarer Zugriffspunkt** eingerichtet wird, um den Zugang auch bei Ausfall einzelner Hosts sicherzustellen.

## Voraussetzungen

### Hardware und Virtualisierung

- **VMware vSphere 8** Umgebung mit ausreichend Ressourcen für die Docker Swarm Nodes.
- **Virtuelle Maschinen (VMs)** für die Docker Swarm Manager und Worker Nodes:
  - **Swarm-Manager Nodes:**
    - **swarm-node1**: IP-Adresse `10.11.2.51`
    - **swarm-node2**: IP-Adresse `10.11.2.52`
    - **swarm-node3**: IP-Adresse `10.11.2.53`
  - **Empfohlene Mindestanforderungen pro VM:**
    - **CPU:** 2 vCPUs
    - **RAM:** 4 GB
    - **Speicher:** 50 GB SSD (für Betriebssystem und Docker)
    - **Netzwerk:** Mindestens zwei Netzwerkkarten (eine für Management, eine für Datenverkehr)
- **Hohe Verfügbarkeit (HA) in VMware vSphere:**
  - **Cluster mit redundanten Hosts:** Mindestens drei Hosts für Fault Tolerance.
  - **Shared Storage:** Gemeinsamer Speicher für VM-Templates und Daten (z.B. GlusterFS).

### Installation der VMware Tools

Die Installation der **VMware Tools** auf den Docker Swarm Nodes ist essenziell für die optimale Leistung und Verwaltung der virtuellen Maschinen. VMware Tools bieten verbesserte Treiber, Synchronisierung der Uhrzeit, bessere Integration mit dem Host und erweiterte Verwaltungsfunktionen.

#### Schritte zur Installation der VMware Tools (Ubuntu 24.04.1 LTS Server)

1. **VMware Tools ISO mounten:**
   - Öffne den vSphere Client und navigiere zu der gewünschten VM.
   - Klicke auf **Aktionen > Gastbetriebssystem > VMware Tools installieren**.
   - Dadurch wird das VMware Tools ISO-Image als CD-ROM in der VM gemountet.

2. **VM in der VM betreiben:**
   - **Für Linux-basierte VMs (Ubuntu 24.04.1 LTS):**

     ```bash
     sudo apt-get update
     sudo apt-get install open-vm-tools -y
     sudo systemctl enable --now open-vm-tools.service
     ```

     - **Falls Fehler auftreten (z.B. Alias oder symbolischer Link):**
       1. **Überprüfe den Dienststatus:**

          ```bash
          sudo systemctl status open-vm-tools.service
          ```

       2. **Starte den Dienst manuell:**

          ```bash
          sudo systemctl start open-vm-tools.service
          ```

       3. **Aktiviere den Dienst beim Systemstart:**

          ```bash
          sudo systemctl enable open-vm-tools.service
          ```

       4. **Falls weiterhin Probleme auftreten:**
          - **Deinstalliere und installiere die Open-VM-Tools neu:**

            ```bash
            sudo apt-get remove --purge open-vm-tools -y
            sudo apt-get autoremove -y
            sudo apt-get install open-vm-tools -y
            sudo systemctl enable --now open-vm-tools.service
            ```

          - **Prüfe die Logs für weitere Informationen:**

            ```bash
            sudo journalctl -u open-vm-tools.service -b
            ```

   - **Für Windows-basierte VMs:**
     - Navigiere im Explorer zur gemounteten CD-ROM (meist `D:`).
     - Führe `setup.exe` aus und folge den Installationsanweisungen.
     - Starte die VM nach der Installation neu.

3. **Überprüfung der Installation:**
   - **Für Linux:**

     ```bash
     sudo systemctl status open-vm-tools.service
     ```

     - Sollte den Status `active (running)` anzeigen.
   - **Für Windows:**
     - Überprüfe den Task-Manager auf den Prozess **vmtoolsd.exe**.
     - Stelle sicher, dass die VMware Tools in den installierten Programmen aufgeführt sind.

4. **Automatische Updates:**
   - Stelle sicher, dass VMware Tools bei Updates der VM automatisch aktualisiert werden, um Sicherheits- und Leistungsverbesserungen zu erhalten.

#### Vorteile der Installation der VMware Tools

- **Verbesserte Leistung:** Optimierte Netzwerk- und Grafiktreiber.
- **Synchronisierte Uhrzeit:** Genauere Zeitsteuerung zwischen Host und Gast.
- **Verbesserte Verwaltung:** Bessere Integration mit vSphere, wie Snapshots und Live-Migration.
- **Clipboard und Drag & Drop (für Desktop-Versionen):** Erleichtert die Interaktion mit der VM.

### Installation und Konfiguration von GlusterFS

**GlusterFS** ist ein verteiltes Dateisystem, das zur Erstellung eines skalierbaren und hochverfügbaren Speichersystems für Docker Swarm verwendet wird. Es ermöglicht die gemeinsame Nutzung von Speichervolumes über mehrere Knoten hinweg.

#### Schritte zur Installation und Konfiguration von GlusterFS

1. **GlusterFS auf allen Swarm Nodes installieren:**
   - **Für Ubuntu 22.04 LTS und höher:**

     ```bash
     sudo apt-get update
     sudo apt-get install -y glusterfs-server
     sudo systemctl enable --now glusterd
     ```

2. **GlusterFS Cluster einrichten:**
   - **Cluster-Knoten identifizieren:**
     - In deinem Setup sind die drei Swarm Nodes:
       - `swarm-node1`: `10.11.2.51`
       - `swarm-node2`: `10.11.2.52`
       - `swarm-node3`: `10.11.2.53`
   - **Nodes zum Cluster hinzufügen:**
     - Auf `swarm-node1`:

       ```bash
       sudo gluster peer probe swarm-node2
       sudo gluster peer probe swarm-node3
       ```

     - Überprüfe die Cluster-Mitglieder:

       ```bash
       sudo gluster peer status
       ```

   - **Replikationsvolume erstellen:**
     - Erstelle ein Verzeichnis auf jedem Node für das Volume:

       ```bash
       sudo mkdir -p /glusterfs/volume1
       ```

     - Erstelle das replizierte Volume:

       ```bash
       sudo gluster volume create gv0 replica 3 swarm-node1:/glusterfs/volume1 swarm-node2:/glusterfs/volume1 swarm-node3:/glusterfs/volume1 force
       ```

     - Starte das Volume:

       ```bash
       sudo gluster volume start gv0
       ```

     - Überprüfe den Volume-Status:

       ```bash
       sudo gluster volume status gv0
       ```

3. **Docker Volume Plugin für GlusterFS installieren:**
   - **Installation des Plugins:**

     ```bash
     docker plugin install gluster/docker-volume-glusterfs
     ```

   - **Aktivieren des Plugins:**

     ```bash
     docker plugin enable gluster/docker-volume-glusterfs
     ```

4. **Docker Volume mit GlusterFS erstellen:**
   - Beispiel für das Erstellen eines Docker Volumes mit dem GlusterFS Plugin:

     ```bash
     docker volume create --driver gluster/docker-volume-glusterfs --name gv0 \
       -o glusterfs-volume=gv0 \
       -o glusterfs-server=swarm-node1:swarm-node2:swarm-node3 \
       -o glusterfs-volume-type=replica
     ```

   - **Hinweis:** Stelle sicher, dass die Namen und Adressen der Nodes korrekt sind.

5. **Integration von GlusterFS in Docker Compose Stacks:**
   - In den `docker-compose.yml` Dateien der jeweiligen Stacks sind die Volumes bereits auf `gv0` konfiguriert. Durch die oben genannten Schritte ist das Volume nun verfügbar und persistent.

6. **Testen der GlusterFS Integration:**
   - Starte die Stacks und überprüfe, ob die Daten korrekt in GlusterFS gespeichert werden.

     ```bash
     docker stack deploy -c traefik.yml traefik
     docker stack deploy -c grafana.yml grafana
     docker stack deploy -c influxdb.yml influxdb
     docker stack deploy -c unifi.yml unifi
     ```

   - Überprüfe die Verzeichnisse auf den GlusterFS Nodes (`/mnt/glusterfs/...`), um sicherzustellen, dass die Daten dort abgelegt werden.

#### Vorteile der Nutzung von GlusterFS

- **Skalierbarkeit:** Einfaches Hinzufügen weiterer Speicher-Knoten.
- **Hochverfügbarkeit:** Replikation sorgt für Datenredundanz und Ausfallsicherheit.
- **Flexibilität:** Unterstützt verschiedene Volume-Typen und -Konfigurationen.
- **Performance:** Optimiert für parallelen Zugriff und hohe Lasten.

### Software-Anforderungen

- **Betriebssystem:** Ubuntu 22.04 LTS (empfohlen) oder eine andere unterstützte Linux-Distribution.
- **Docker:** Installiert auf allen Swarm Nodes.
- **Docker Swarm:** Initialisiert und konfiguriert.
- **Traefik, HAProxy, Grafana, InfluxDB, UniFi:** Bereits konfigurierte Stack-Dateien (`traefik.yml`, `haproxy.yml`, `grafana.yml`, `influxdb.yml`, `unifi.yml`).
- **GlusterFS:** Installiert und konfiguriert auf allen Swarm Nodes.

---

## Netzwerkkonfiguration

### 1. Erstellen des Overlay-Netzwerks `traefik_network`

Dieses Netzwerk ermöglicht die Kommunikation zwischen Traefik, HAProxy und den Backend-Diensten (Grafana, InfluxDB, UniFi).

```bash
docker network create --driver=overlay --attachable traefik_network
```

**Details:**

- **Name:** `traefik_network`
- **Driver:** `overlay`
- **Attachable:** `true` (erlaubt das Anhängen von Containern nachträglich)
- **Subnetz:** `10.0.13.0/24` (kann nach Bedarf angepasst werden)

---

## DNS-Konfiguration

### Split DNS Setup

#### Interne DNS (nur innerhalb des Netzwerks erreichbar)

- **`traefik.local.lezit.de`**: Zeigt auf die interne IP-Adresse deines Traefik-Load Balancers.
- **`grafana.local.lezit.de`**: Zeigt auf die interne IP-Adresse deines Traefik-Load Balancers.
- **`influxdb.local.lezit.de`**: Zeigt auf die interne IP-Adresse deines Traefik-Load Balancers.
- **`unifi.local.lezit.de`**: Zeigt auf die interne IP-Adresse deines Traefik-Load Balancers.
- **`service.local.lezit.de`**: Zeigt auf die interne IP-Adresse deines HAProxy Load Balancers.
