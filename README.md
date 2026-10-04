Projekt: Aufbau eines containerbasierten Dokumentenmanagement- und Monitoring-Systems
Plattform: Proxmox VE 9.2.2 auf einem Notebook


Phase 1: Planung und Entscheidungen
Projektziel definiert: Verständnis von Proxmox VE, LXC-Containern und deren Netzwerkanbindung; Aufbau eines produktiv nutzbaren Dokumentenmanagement-Systems.

Hardware geprüft:

CPU: Intel Core i7-7500U (2 Kerne, 4 Threads via Hyper-Threading), 2,70 GHz Basistakt

RAM: 15,39 GiB (verfügbar für Proxmox und alle Gäste)

Speicher: ~94 GB SSD (Proxmox-System + Container-RootFS)

Boot-Modus: EFI mit Secure Boot

Virtualisierungsentscheidung: LXC statt KVM/QEMU

LXC teilt den Kernel des Hosts → kein eigener Kernel pro Gast

Geringerer RAM- und CPU-Overhead (kein vollständiges Hardware-Emulationslayer)

Schnellere Boot-Zeiten (Sekunden statt Minuten)

Ideal für ressourcenschonenden Dauerbetrieb auf Notebook-Hardware

Betriebssystem-Entscheidung: Ubuntu 22.04 LTS

Langzeit-Support (bis 2027)

Python 3.10 vorinstalliert (Voraussetzung für viele Dienste)

Breite Dokumentation und Paketverfügbarkeit

Kompatibilität mit Docker und Docker Compose

Architektur-Entscheidung: Dienste-Trennung (Separation of Concerns)

Datenbank, Cache und Anwendung jeweils in eigenem Container

Vorteil: Fehlerisolation, unabhängige Updates, klare Ressourcenzuordnung

Netzwerk-Entscheidung: Internes Bridge-Netzwerk vmbr1 mit 10.10.10.0/24

Kein NAT, kein DHCP → statische IPs für deterministische Erreichbarkeit

Nur Container 202 (Paperless) und 203 (PatchMon) haben zusätzlich Zugang zum Heimnetz




Phase 2: Netzwerk und Container-Grundlagen
Internes Netzwerk erstellt:

Linux-Bridge vmbr1 in Proxmox angelegt

IP-Adresse des Hosts in diesem Netz: 10.10.10.1/24 (dient als Gateway)

Bridge ist nicht an eine physische Netzwerkkarte gebunden → rein virtuelles Netz

Alle Container im 10.10.10.0/24-Netz können sich gegenseitig erreichen

Container 200 (db) erstellt:

Template: ubuntu-22.04-standard_22.04-1_amd64.tar.zst

Ressourcen: 2 GB RAM, 2 CPU-Kerne, 8 GB RootFS

Netzwerk: eth0 an vmbr1, IP 10.10.10.2/24, Gateway 10.10.10.1

Hostname: db

Container 201 (redis) erstellt:

Ressourcen: 1 GB RAM, 1 CPU-Kern, 6 GB RootFS

Netzwerk: eth0 an vmbr1, IP 10.10.10.3/24, Gateway 10.10.10.1

Hostname: redis

Container 202 (paperless) erstellt:

Ressourcen: 3 GB RAM, 2 CPU-Kerne, 12 GB RootFS

Netzwerk: zwei Interfaces

eth0 an vmbr1, IP 10.10.10.4/24 (Kommunikation mit db und redis)

eth1 an vmbr0, DHCP (Zugang zum Heimnetz für Browser-Zugriff)

Hostname: paperless

Alle Container gestartet und Status in Proxmox geprüft (pct list).




Phase 3: Datenbank und Cache einrichten
Container 200 (db) – PostgreSQL installieren:

apt update && apt install postgresql -y

Installierte Version: PostgreSQL 14 (Standard in Ubuntu 22.04)

Dienst läuft als Systemd-Unit postgresql.service

Standard-Port: 5432

Datenbank und Benutzer erstellt:

CREATE DATABASE paperless;

CREATE USER paperless WITH PASSWORD '...';

GRANT ALL PRIVILEGES ON DATABASE paperless TO paperless;

Benutzer hat volle Rechte auf die Datenbank, aber keine Superuser-Rechte

PostgreSQL für Netzwerkzugriff konfiguriert:

In /etc/postgresql/14/main/postgresql.conf: listen_addresses = '*'

PostgreSQL lauscht damit auf allen Netzwerk-Interfaces (vorher nur localhost)

Zugriffskontrolle in pg_hba.conf:

Eintrag: host paperless paperless 10.10.10.0/24 md5

Bedeutung: Host-basierte Authentifizierung mit MD5-Passwort-Hash für alle IPs im internen Netz

PostgreSQL neu gestartet: systemctl restart postgresql

Container 201 (redis) – Redis installieren:

apt install redis-server -y

Installierte Version: Redis 6.x

Standard-Port: 6379

Redis für Netzwerkzugriff konfiguriert:

In /etc/redis/redis.conf: bind 0.0.0.0

Redis lauscht damit auf allen Interfaces

Redis neu gestartet: systemctl restart redis-server




Phase 4: Paperless-ngx installieren
Container 202 (paperless) – Docker vorbereiten:

apt install ca-certificates curl gnupg lsb-release -y

Docker-GPG-Schlüssel importiert nach /usr/share/keyrings/docker-archive-keyring.gpg

Docker-Repository in /etc/apt/sources.list.d/docker.list hinzugefügt

Installation: docker-ce, docker-ce-cli, containerd.io, docker-compose-plugin

Verzeichnis erstellt: /opt/paperless (Arbeitsverzeichnis für Docker Compose)

Docker-Compose-Datei heruntergeladen:

Quelle: offizielles Paperless-ngx-Repository

Datei: docker-compose.yml

Compose-Datei angepasst:

Abschnitte db (PostgreSQL) und redis entfernt (laufen separat)

Umgebungsvariablen für externe Verbindungen gesetzt:

PAPERLESS_DBHOST: "10.10.10.2"

PAPERLESS_DBNAME: "paperless"

PAPERLESS_DBUSER: "paperless"

PAPERLESS_DBPASS: "<Passwort>"

PAPERLESS_REDIS: "redis://10.10.10.3:6379"

.env-Datei erstellt:

PAPERLESS_SECRET_KEY (für Session-Verschlüsselung)

PAPERLESS_TIME_ZONE=Europe/Berlin

Volume-Ordner angelegt: data, media, export, consume

consume = Eingangsordner für neue PDFs (wird von Paperless überwacht)

media = Ablage der verarbeiteten Dokumente

data = Index- und Suchdaten

Paperless gestartet: docker compose up -d

Docker zieht das Image ghcr.io/paperless-ngx/paperless-ngx:latest

Container wird im Hintergrund gestartet (-d = detached)

Test: Paperless im Browser geöffnet (http://<IP>:8000), Admin-Account erstellt.

Erstes Dokument hochgeladen: PDF in consume-Ordner gelegt → Paperless erkennt, OCR-verarbeitet und indexiert automatisch.

Phase 5: Automatisierung einrichten
Backup-Skript erstellt (/usr/local/bin/paperless-db-backup.sh im Container 200):

pg_dump paperless > /var/backups/paperless/paperless-db-<Datum>.sql

Rotation: ls -t *.sql | tail -n +8 | xargs rm → behält die letzten 7 Backups

Ausführung als postgres-Benutzer via su - postgres -c

Selbstheilungs-Skript erstellt (/usr/local/bin/paperless-check.sh auf dem Proxmox-Host):

Array CONTAINER_IDS=(200 201 202)

Für jede ID: pct status $ID | awk '{print $2}'

Wenn Status ≠ running: pct start $ID

Läuft auf dem Host, nicht im Container → kann Container steuern

Wartungs-Skript erstellt (/usr/local/bin/paperless-wartung.sh auf dem Proxmox-Host):

Ruft zuerst Backup im Container auf: pct exec 200 -- /usr/local/bin/paperless-db-backup.sh

Ruft dann Check auf: /usr/local/bin/paperless-check.sh

Cronjobs eingerichtet (crontab -e auf dem Proxmox-Host):

*/10 * * * * → Selbstheilungs-Check alle 10 Minuten

0 3 * * * → Komplette Wartung täglich um 03:00 Uhr

Ausgabe jeweils in Log-Dateien umgeleitet (>> /var/log/... 2>&1)




Phase 6: Proxmox vzdump-Backup
Backup-Job in Proxmox-GUI erstellt:

Ziel: Lokaler Speicher (local)

Modus: Snapshot → Container läuft während des Backups weiter

Komprimierung: ZSTD → gute Balance zwischen Geschwindigkeit und Dateigröße

Auswahl: Alle Container (200–203)

Zeitplan: Täglich nachts

Aufbewahrung: Maximal X Backups (einstellbar)

Backup-Job getestet: Manuell ausgeführt, Status in Proxmox-GUI geprüft

Ausgabeformat: .tar.zst für LXC-Container

Enthält: Container-Konfiguration + gesamtes RootFS

Technischer Hintergrund vzdump:

Proxmox-internes Tool für Backup und Restore

Nutzt bei LXC den konfigurierten Modus (Snapshot/Suspend/Stop)

Snapshot-Modus erfordert LVM-Thin, ZFS oder Ceph

Restore über GUI: Speicher → Backups → Restore




Phase 7: PatchMon installieren
Container 203 (patchmon) erstellt:

Ressourcen: 2 GB RAM, 2 CPU-Kerne, 10 GB RootFS

Netzwerk: eth0 an vmbr0, DHCP

Hostname: patchmon

Docker und Docker Compose installiert (analog zu Phase 4, Schritt 20)

Verzeichnis erstellt: /opt/patchmon

Docker-Compose-Datei heruntergeladen von offizieller PatchMon-Quelle

.env-Datei erstellt:

POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB (für interne DB)

DATABASE_URL=postgresql://user:pass@database:5432/db (Verbindung zum DB-Container)

REDIS_HOST=redis, REDIS_PORT=6379, REDIS_PASSWORD

JWT_SECRET (für Token-Signierung)

SERVER_HOST, SERVER_PORT=3000, CORS_ORIGIN

PatchMon gestartet: docker compose up -d

Docker Compose liest die .env-Datei und ersetzt Variablen in der Compose-Datei

Proxmox-Integration eingerichtet:

In PatchMon: Settings → Integrations → Proxmox Auto-Enrollment-Token erstellt

Ergebnis: Ein curl-Befehl mit Token, der auf dem Proxmox-Host ausgeführt wird

Der Befehl installiert den PatchMon-Agenten in allen laufenden LXC-Containern

Alle LXC-Container in PatchMon sichtbar und überwacht

Anzeige: Ausstehende Updates, Kernel-Version, Uptime, Paketanzahl




Phase 8: Feste IP-Adressen vergeben
Container 202 (paperless) angepasst:

In Proxmox-GUI: Network → net1 (eth1) bearbeiten

IPv4: von DHCP auf Static umgestellt

IPv4/CIDR: 192.168.89.210/24, Gateway: 192.168.89.1

Wichtig: net0 (intern, 10.10.10.4) bleibt unverändert

Container 203 (patchmon) angepasst:

In Proxmox-GUI: Network → net0 (eth0) bearbeiten

IPv4: Static, 192.168.89.211/24, Gateway: 192.168.89.1

Proxmox-Host angepasst:

System → Network → vmbr0 bearbeiten

IP auf eine freie Adresse im 89er-Netz gesetzt (z. B. 192.168.89.10/24)

Gateway: 192.168.89.1

Wichtig: /etc/hosts prüfen, damit der Hostname auf die neue IP zeigt

Docker-Konfigurationen geprüft:

PatchMon .env: SERVER_HOST und CORS_ORIGIN auf neue IP angepasst

docker compose up -d zum Neustart mit neuen Werten

Technischer Hintergrund feste IPs in Proxmox:

Proxmox verwaltet die Container-Netzwerkkonfiguration zentral

Bei Ubuntu-Containern schreibt Proxmox in systemd-networkd-kompatible Dateien

Nach Änderung: Container neu starten oder systemctl restart systemd-networkd im Container




Phase 9: Dokumentation erstellt
Architektur dokumentiert:

Schichtenmodell: Hardware → Proxmox → Bridges → Container

Container-Übersicht mit IDs, IPs, Ressourcen

Datenfluss: Browser → Paperless → PostgreSQL/Redis

Software-Vergleiche erstellt:

Backup: vzdump vs. Veeam vs. Iperius

DMS: Paperless-ngx vs. Mayan EDMS vs. Docspell

Patch-Monitoring: PatchMon vs. Action1 vs. apt-dater vs. Ansible

Automatisierung: Shell+Cron vs. Ansible vs. Puppet

RTO definiert:

Container-Ausfall: ≤ 10 Min (Selbstheilung)

Datenbank defekt: ~30 Min (pg_dump-Restore)

SSD-Ausfall: 4–8 h (vzdump-Restore von extern)

Monitoring-Konzept beschrieben:

Host: CPU, RAM, Temperatur, SMART

Container: Status, Updates, Ressourcen

Dienste: HTTP-Erreichbarkeit, DB-Verbindung, Backup-Erfolg

Automatisierungskonzept dokumentiert (3 Stufen):

Stufe 1 (umgesetzt): Selbstheilung, Backup, Patch-Monitoring

Stufe 2 (geplant): USB-Backup, Temperatur-Logging, E-Mail-Alerting

Stufe 3 (perspektivisch): Prometheus/Grafana, Ansible, Backup-Verifikation





Anleitungen und Tutorials gesammelt:

Proxmox VE Wiki, Paperless-ngx Docs, PatchMon Docs

Docker Compose Docs, PostgreSQL Docs, Ansible Best Practices

Spickzettel mit wichtigen Befehlen erstellt (siehe unten)

Notfall-Wiederherstellung beschrieben:

Container prüfen → PostgreSQL prüfen → Backup einspielen → vzdump-Restore

Technische Zusammenfassung
Container-Übersicht (technisch)
ID	Name	OS	RAM	CPU	RootFS	IP intern	IP extern
200	db	Ubuntu 22.04	2 GB	2	8 GB	10.10.10.2/24	–
201	redis	Ubuntu 22.04	1 GB	1	6 GB	10.10.10.3/24	–
202	paperless	Ubuntu 22.04	3 GB	2	12 GB	10.10.10.4/24	192.168.89.210/24
203	patchmon	Ubuntu 22.04	2 GB	2	10 GB	–	192.168.89.211/24
Netzwerk-Topologie
text
vmbr0 (physisch, Heimnetz 192.168.89.0/24)
 ├── Proxmox-Host: 192.168.89.10
 ├── Container 202 (eth1): 192.168.89.210
 └── Container 203 (eth0): 192.168.89.211

vmbr1 (virtuell, intern 10.10.10.0/24)
 ├── Proxmox-Host: 10.10.10.1 (Gateway)
 ├── Container 200: 10.10.10.2
 ├── Container 201: 10.10.10.3
 └── Container 202 (eth0): 10.10.10.4
Ports und Dienste
Dienst	Port	Protokoll	Erreichbar von
Paperless Web	8000	HTTP	Heimnetz
PatchMon Web	3000	HTTP	Heimnetz
PostgreSQL	5432	TCP	nur intern (10.10.10.0/24)
Redis	6379	TCP	nur intern (10.10.10.0/24)
Wichtige Befehle (technisch)
bash

# Backup manuell auslösen
/usr/local/bin/paperless-wartung.sh

# Docker-Status im Container
cd /opt/paperless && docker compose ps
cd /opt/paperless && docker compose logs -f
cd /opt/paperless && docker compose restart

# PostgreSQL-Status im Container 200
systemctl status postgresql
psql -U paperless -d paperless -c "\dt"

# Redis-Status im Container 201
systemctl status redis-server
redis-cli ping

# Logs prüfen
cat /var/log/paperless-check.log
cat /var/log/paperless-wartung.log
tail -f /var/log/syslog
Cronjob-Übersicht
Zeit	Befehl	Zweck
*/10 * * * *	/usr/local/bin/paperless-check.sh	Selbstheilung
0 3 * * *	/usr/local/bin/paperless-wartung.sh	Backup + Check
