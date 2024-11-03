# Ubuntu Installation und Konfiguration

## wichtige Links


- **Glances** is an open-source system cross-platform monitoring tool: [GitHub - nicolargo/glances: Glances an Eye on your system. A top/htop alternative for GNU/Linux, BSD, Mac OS and Windows operating systems.](https://github.com/nicolargo/glances)

Portainer BE License for 3 Nodes

`3-n6nMpmh+73S9X2XBQfr4hpU+mlrY9eDTHDAn9J8SWeBEmwbatuuFJHO8AJqoKv0gz7FQwf1MJTaS4nbleClkoEvcqqga9N+fbLJUbL7ygf5zzBz4egm+NUS+Q2tVcZTvkg==`

Hosts

| Name  | IP         | CPU | RAM | HDD | Beschreibung | 
| ----- | ---------- | --- | --- | --- | ------------ |
| node1 | 10.11.2.51 | 1   | 8   | 40  |              |
| node2 | 10.11.2.52 | 1   | 8   | 40  |              |
| node3 | 10.11.2.53 | 1   | 8   | 40  |              |
| node4 | 10.11.2.54 | 1   | 8   | 40  |              |

## Netzwerkkonfiguration

```

root@localhost:~# sudo apt install network-manager
root@localhost:~# sudo hostnamectl set-hostname swarm-node1
root@localhost:~# sudo nano /etc/hosts
root@localhost:~# sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.org
root@localhost:~# sudo nano /etc/netplan/01-netcfg.yaml

# create new
network:
  ethernets:
    # interface name
    ens33:
      dhcp4: false
      # IP address/subnet mask
      addresses: [10.11.2.52/24]
      # default gateway
      # [metric] : set priority (specify it if multiple NICs are set)
      # lower value is higher priority
      routes:
        - to: default
          via: 10.11.2.1
          metric: 100
      nameservers:
        # name server to bind
        addresses: [10.11.2.1]
        # DNS search base
        search: [local.lezit.de]
      dhcp6: false
  version: 2

root@localhost:~# sudo chmod 600 /etc/netplan/01-netcfg.yaml
root@localhost:~# sudo netplan apply
```

Bei vorhanden sein einer DHCP Adresse, folgenden Befehl ausführen:

```
sudo ip addr del 10.11.2.105/24 dev ens33
```

docker swarm join --token SWMTKN-1-5fefin3wx5rhk1mii681l21rmd9eh2geawgvfsufh0qtmnrpay-8edqqevjg8jc5sccv33boz3ed 10.11.2.51:2377

10.11.2.51  swarm-node1
10.11.2.52  swarm-node2
10.11.2.53  swarm-node3


Hier ist die aktualisierte Anleitung, einschließlich der detaillierten Partitionierungsschritte und den spezifischen IP-Adressen und Hostnamen, die du verwendest.

---

### **1. Konfigurieren des Ubuntu Hosts inkl. Storage**

#### A. **System-Updates durchführen**
Führe die folgenden Befehle aus, um sicherzustellen, dass dein Ubuntu-System auf dem neuesten Stand ist:
```bash
sudo apt update && sudo apt upgrade -y
```

#### B. **Netzwerk und Hostnamen konfigurieren**
Setze den Hostnamen für jeden deiner Docker Hosts (z.B. `swarm-node1`, `swarm-node2`, `swarm-node3`):
```bash
sudo hostnamectl set-hostname swarm-node1  # Auf swarm-node2 und swarm-node3 entsprechend ändern
```

#### C. **Festplatten konfigurieren**

##### **1. Partitionierung der Festplatten**

Du hast drei Festplatten pro Host:
- **/dev/sda (25 GB)** für das Betriebssystem (Ubuntu)
- **/dev/sdb (50 GB)** für Docker-Daten
- **/dev/sdc (100 GB)** für GlusterFS-Daten

Hier sind die Schritte zur Partitionierung von `/dev/sdb` und `/dev/sdc`.

##### **a. Partitionierung von /dev/sdb (50 GB für Docker-Daten)**

- **Starte `fdisk`**:
  ```bash
  sudo fdisk /dev/sdb
  ```

  Du befindest dich jetzt in der `fdisk`-Konsole. Verwende die folgenden Schritte, um eine neue Partition zu erstellen:

1. Tippe `n` (für "new partition") und drücke Enter.
2. Wähle `p` (für "primary partition") und drücke Enter.
3. Akzeptiere die Standardwerte für den Startsektor, indem du Enter drückst.
4. Akzeptiere die Standardwerte für den Endsektor (die Partition wird die gesamte Festplatte nutzen), indem du Enter drückst.
5. Tippe `w`, um die Änderungen zu speichern und die Partitionstabelle zu schreiben.

##### **b. Partitionierung von /dev/sdc (100 GB für GlusterFS-Daten)**

Wiederhole den gleichen Prozess für die Festplatte `/dev/sdc`:

- **Starte `fdisk`**:
  ```bash
  sudo fdisk /dev/sdc
  ```

  Dann wiederhole dieselben Schritte wie bei `/dev/sdb`:
1. Tippe `n`, wähle `p`, drücke Enter für die Standardwerte bei Start- und Endsektor.
2. Tippe `w`, um die Partitionstabelle zu speichern.

##### **2. Formatierung der Partitionen**

Nun müssen die Partitionen formatiert werden.

##### **a. Formatierung von `/dev/sdb1` als `ext4` (für Docker-Daten)**

- Erstelle das Dateisystem:
  ```bash
  sudo mkfs.ext4 /dev/sdb1
  ```

##### **b. Formatierung von `/dev/sdc1` als `xfs` (für GlusterFS-Daten)**

- Erstelle das Dateisystem:
  ```bash
  sudo mkfs.xfs /dev/sdc1
  ```

##### **3. Mountpoints erstellen und Partitionen mounten**

Jetzt müssen die Partitionen auf die gewünschten Verzeichnisse gemountet werden.

##### **a. Mounten von `/dev/sdb1` für Docker-Daten**

- **Erstelle den Mountpoint**:
  ```bash
  sudo mkdir -p /mnt/docker
  ```

- **Mounte die Partition**:
  ```bash
  sudo mount /dev/sdb1 /mnt/docker
  ```

##### **b. Mounten von `/dev/sdc1` für GlusterFS-Daten**

- **Erstelle den Mountpoint**:
  ```bash
  sudo mkdir -p /data/glusterfs
  ```

- **Mounte die Partition**:
  ```bash
  sudo mount /dev/sdc1 /data/glusterfs
  ```

##### **4. Einträge in `/etc/fstab` hinzufügen**

Damit die Partitionen beim Neustart automatisch gemountet werden, fügst du die entsprechenden Einträge in die Datei `/etc/fstab` ein.

- **Öffne `/etc/fstab`**:
  ```bash
  sudo nano /etc/fstab
  ```

- **Füge diese Zeilen hinzu**:
  ```
  /dev/sdb1 /var/lib/docker ext4 defaults 0 0
  /dev/sdc1 /data/glusterfs xfs defaults 0 0
  ```

- **Speichere und beende den Editor**.

---

### **2. Installation von Docker**

#### A. **Docker-Repository hinzufügen und Docker installieren**

- **Führe folgende Befehle aus, um das Docker-Repository hinzuzufügen**:
  ```bash
  sudo apt install apt-transport-https ca-certificates curl software-properties-common
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
  echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  ```

- **Installiere Docker**:
  ```bash
  sudo apt update
  sudo apt install docker-ce docker-ce-cli containerd.io
  ```

#### B. **Docker starten und aktivieren**

- **Docker-Dienst aktivieren und starten**:
  ```bash
  sudo systemctl enable docker
  sudo systemctl start docker
  ```

#### C. **Installation überprüfen**

- **Überprüfe, ob Docker korrekt installiert ist**:
  ```bash
  docker --version
  ```

---

### **3. Installation von Docker Swarm**

#### A. **Docker Swarm initialisieren**

Führe auf dem ersten Docker Host (`swarm-node1`) den folgenden Befehl aus, um Docker Swarm zu initialisieren:

```bash
docker swarm init --advertise-addr 10.11.2.51
```

Dies erstellt den ersten Manager-Knoten im Swarm-Cluster.

#### B. **Weitere Hosts dem Docker Swarm hinzufügen**

Füge die anderen Hosts als Manager hinzu, indem du den Befehl ausführst, der nach der Initialisierung angezeigt wird. Auf den anderen beiden Hosts (`swarm-node2` und `swarm-node3`) führst du den folgenden Befehl aus:

```bash
docker swarm join --token <manager-token> 10.11.2.51:2377
```

Falls du den Token nicht mehr zur Hand hast, kannst du ihn auf dem ersten Host abrufen:

```bash
docker swarm join-token manager
```

#### C. **Swarm-Knoten überprüfen**

Überprüfe, ob alle Knoten dem Cluster beigetreten sind, indem du auf einem der Manager den folgenden Befehl ausführst:

```bash
docker node ls
```

---

### **4. Installation von GlusterFS**

#### A. **GlusterFS installieren**

Installiere GlusterFS auf allen drei Docker Hosts:

- **Füge das GlusterFS-Repository hinzu**:
  ```bash
  sudo add-apt-repository ppa:gluster/glusterfs-10
  sudo apt update
  ```

- **Installiere GlusterFS**:
  ```bash
  sudo apt install glusterfs-server
  ```

#### B. **GlusterFS-Dienst starten und aktivieren**

- **Starte und aktiviere den GlusterFS-Dienst**:
  ```bash
  sudo systemctl start glusterd
  sudo systemctl enable glusterd
  ```

#### C. **GlusterFS-Cluster einrichten**

Führe die folgenden Schritte auf **swarm-node1** aus, um die anderen Hosts in den GlusterFS-Cluster aufzunehmen:

- **Peers hinzufügen**:
  ```bash
  sudo gluster peer probe 10.11.2.52
  sudo gluster peer probe 10.11.2.53
  ```

- **Erstelle ein GlusterFS-Volume**:
  ```bash
  sudo gluster volume create gv0 replica 3 transport tcp 10.11.2.51:/data/glusterfs 10.11.2.52:/data/glusterfs 10.11.2.53:/data/glusterfs
  ```

- **Starte das Volume**:
  ```bash
  sudo gluster volume start gv0
  ```

#### D. **GlusterFS-Volume mounten**

- **Erstelle den Mountpoint auf allen Hosts**:
  ```bash
  sudo mkdir -p /mnt/glusterfs
  ```

- **Mount das GlusterFS-Volume auf jedem Host**:
  ```bash
  sudo mount -t glusterfs localhost:/gv0 /mnt/glusterfs
  ```

- **Füge den Mount zur `/etc/fstab` hinzu**:
  ```bash
  localhost:/gv0 /mnt/glusterfs glusterfs defaults,_netdev,noauto,x-systemd.automount        0 0
  ```


### **5. Installation von Portainer CE**

Portainer CE bietet eine grafische Benutzeroberfläche zur Verwaltung deines Docker Swarm Clusters.

#### A. **Portainer-Volume erstellen**

Erstelle auf einem der Docker-Manager ein Docker-Volume für die Portainer-Daten:

```bash
docker volume create portainer_data
```

#### B. **Portainer-Container starten**

Starte den Portainer-Container mit dem folgenden Befehl. Dieser Container wird auf Port 9000 laufen und du kannst ihn später über den Browser erreichen.

```bash
docker run -d -p 9000:9000 --name=portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce
```

Dieser Befehl:
- Lädt den offiziellen Portainer CE Container aus dem Docker Hub.
- Bindet den Docker Socket (`/var/run/docker.sock`) ein, um Zugriff auf die Docker-API zu ermöglichen.
- Verwendet das zuvor erstellte Docker-Volume `portainer_data` zur Speicherung von Portainer-Daten.

#### C. **Zugriff auf Portainer**

Nachdem Portainer gestartet ist, kannst du über deinen Webbrowser auf das Dashboard zugreifen:

1. Öffne einen Browser und gehe zu: `http://<IP-Adresse-des-Hosts>:9000`. Wenn du den Portainer-Container auf `swarm-node1` gestartet hast, wäre das:
   ```
   http://10.11.2.51:9000
   ```

2. Beim ersten Zugriff wirst du aufgefordert, ein Admin-Passwort zu erstellen. Lege dieses fest und melde dich anschließend mit deinen Anmeldedaten an.

3. Wähle beim Setup "Docker" oder "Docker Swarm", je nachdem, was du verwalten möchtest (Swarm ist korrekt in deinem Fall, da du einen Swarm-Cluster betreibst).

#### D. **Verwaltung des Docker Swarm Clusters**

Nach der Anmeldung kannst du im Portainer-Dashboard deinen Docker Swarm Cluster verwalten. Hier kannst du:
- Container und Services starten und stoppen.
- Netzwerke und Volumes verwalten.
- Den Zustand deines Swarm Clusters überwachen (z.B. aktive Nodes, laufende Services).


```
version: '3.2'

services:
  agent:
    image: portainer/agent:2.21.2
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - agent_network
    deploy:
      mode: global
      placement:
        constraints: [node.platform.os == linux]

  portainer:
    image: portainer/portainer-ee:2.21.2
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    ports:
      - "9443:9443"
      - "9000:9000"
      - "8000:8000"
    volumes:
      - data:/data
    networks:
      - agent_network
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]

networks:
  agent_network:
    driver: overlay
    attachable: true

volumes:
  data:
    driver: glusterfs
    driver_opts:
      voluri: "localhost:/gv0"
      mountpoint: "/mnt/glusterfs/portainer_data"

```