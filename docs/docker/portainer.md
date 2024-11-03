
# Installation von Portainer BE**

Portainer CE bietet eine grafische Benutzeroberfläche zur Verwaltung deines Docker Swarm Clusters.

## A. **Portainer-Volume erstellen**

Erstelle auf einem der Docker-Manager ein Docker-Volume für die Portainer-Daten:

```bash
docker volume create portainer_data
```

## B. **Portainer-Container starten**

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

## C. **Zugriff auf Portainer**

Nachdem Portainer gestartet ist, kannst du über deinen Webbrowser auf das Dashboard zugreifen:

1. Öffne einen Browser und gehe zu: `http://<IP-Adresse-des-Hosts>:9000`. Wenn du den Portainer-Container auf `swarm-node1` gestartet hast, wäre das:

   ```url
   http://10.11.2.51:9000
   ```

2. Beim ersten Zugriff wirst du aufgefordert, ein Admin-Passwort zu erstellen. Lege dieses fest und melde dich anschließend mit deinen Anmeldedaten an.

3. Wähle beim Setup "Docker" oder "Docker Swarm", je nachdem, was du verwalten möchtest (Swarm ist korrekt in deinem Fall, da du einen Swarm-Cluster betreibst).

## D. **Verwaltung des Docker Swarm Clusters**

Nach der Anmeldung kannst du im Portainer-Dashboard deinen Docker Swarm Cluster verwalten. Hier kannst du:

- Container und Services starten und stoppen.
- Netzwerke und Volumes verwalten.
- Den Zustand deines Swarm Clusters überwachen (z.B. aktive Nodes, laufende Services).

```yml
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
