# Docker Infrastruktur mit Ansible & Docker Swarm

## Inhaltsverzeichnis

1. Eigenständiger Docker Host mit AWX (Ansible GUI)
2. Docker Swarm Cluster mit GlusterFS und Portainer

---

## 1. Eigenständiger Docker Host mit AWX (Ansible GUI)

### Ziel

Bereitstellung eines eigenständigen Docker Hosts zur Verwaltung von Ansible-Playbooks über die Web-Oberfläche von AWX.

### Voraussetzungen

- Ubuntu 24.04 LTS
- Feste IP-Adresse: z. B. `10.11.2.60`
- DNS-Name (optional): `awx.lezit.de`

### Vorbereitung

#### SSH-Zugang sicherstellen

```bash
ssh ubuntu@10.11.2.60
```

#### System aktualisieren

```bash
sudo apt update && sudo apt upgrade -y
```

---

### AWX per Ansible installieren

#### 1. Lokales Repository vorbereiten

```bash
unzip awx-deploy-playbook.zip
cd awx-deploy
```

#### 2. Inventar anpassen (falls nötig)

`inventory/hosts.yml`

```yaml
all:
  hosts:
    awx-manager:
      ansible_host: 10.11.2.60
```

#### 3. Playbook starten

```bash
ansible-playbook -i inventory/hosts.yml install-awx.yml
```

#### 4. Zugriff auf Weboberfläche

- URL: `http://10.11.2.60:8080` oder `http://awx.lezit.de`
- Benutzer: `admin`
- Passwort: `changeme`

#### 5. Optional: DNS + Reverse Proxy

- `awx.lezit.de` als DNS-Eintrag setzen
- HAProxy konfigurieren mit SSL-Zertifikat (Let's Encrypt)

---

## 2. Docker Swarm Cluster mit GlusterFS und Portainer

### Ziel

Bereitstellung eines hochverfügbaren Swarm-Clusters mit persistentem Speicher und GUI-Verwaltung.

### Architektur

| Node        | IP-Adresse     | Rolle              |
|-------------|----------------|--------------------|
| swarm-node1 | 10.11.2.51     | Manager + Worker   |
| swarm-node2 | 10.11.2.52     | Manager + Worker   |
| swarm-node3 | 10.11.2.53     | Manager + Worker   |

### Voraussetzungen

- Ubuntu 24.04 auf allen Nodes
- SSH-Key-Zugriff von einem Ansible-Host
- DNS-Einträge (optional): `*.lezit.de` + `*.local.lezit.de`

### Ansible-Verzeichnisstruktur

```
swarm-cluster/
├── inventory/hosts.yml
├── group_vars/all.yml
├── roles/
│   ├── common/
│   ├── docker/
│   ├── glusterfs/
│   ├── swarm/
│   └── portainer/
├── site.yml
```

### Schrittweise Einrichtung

#### 1. Vorbereitung (Common Rolle)

- Systemupdates
- Hostname setzen
- Zeitzone

#### 2. Docker Rolle

- Docker CE installieren
- Benutzer zur Gruppe hinzufügen

#### 3. GlusterFS Rolle

- Installation und Konfiguration von GlusterFS
- Volume-Erstellung (replica 3)
- Mountpoints unter `/mnt/glusterfs`

#### 4. Swarm Rolle

- Swarm initialisieren
- Alle Nodes als Manager joinen
- Overlay-Netzwerke erstellen

#### 5. Portainer Rolle

- Deployment im Swarm
- Agent im globalen Modus
- Volume über GlusterFS
- Zugriff z. B. über `portainer.lezit.de`

### Zugriff auf Portainer

- URL: `http://portainer.lezit.de`
- Benutzer/Passwort: bei Erststart festlegen

---

### Hinweise zur Sicherheit

- SSH-Zugriff mit Key-Auth
- Ansible Vault für Secrets
- Firewalls: nur benötigte Ports öffnen
- Ressourcenlimits & Healthchecks in Docker Compose verwenden

---

### Nächste Schritte

- Integration mit GitLab/Gitea für Playbook-Verwaltung
- Nutzung von Secrets im Swarm (Docker & Ansible Vault)
- Automatisiertes Backup der GlusterFS Volumes
- Monitoring mit Grafana/Prometheus
