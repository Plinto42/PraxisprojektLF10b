# PraxisprojektLF10b
Proxmox
 Projektablauf: Paperless-ngx Homelab auf Proxmox
Projekt: Aufbau eines containerbasierten Dokumentenmanagement- und Monitoring-Systems
Plattform: Proxmox VE auf einem alten Notebook

#Phase 1: Planung und Entscheidungen
Projektziel definiert: Proxmox und virtuelle Maschinen/Container verstehen, Dienste aufsetzen.

Hardware geprüft: Altes Notebook mit Proxmox VE 9.2.2, Intel i7-7500U, 15,39 GB RAM, ~94 GB SSD.

Entschieden: LXC-Container statt VMs (ressourcenschonend).

Entschieden: Ubuntu 22.04 als Container-Betriebssystem.

Entschieden: Dienste separat aufteilen (Datenbank, Cache, Anwendung getrennt).

Entschieden: Internes Netzwerk (vmbr1) für sichere Trennung.

#Phase 2: Netzwerk und Container-Grundlagen
Internes Netzwerk erstellt: Bridge vmbr1 mit 10.10.10.1/24 in Proxmox angelegt.

Container 200 (db) erstellt: Ubuntu 22.04, 2 GB RAM, IP 10.10.10.2.

Container 201 (redis) erstellt: Ubuntu 22.04, 1 GB RAM, IP 10.10.10.3.

Container 202 (paperless) erstellt: Ubuntu 22.04, 3 GB RAM, IP 10.10.10.4 (intern) + zweite Netzwerkkarte für Heimnetz.

Alle Container gestartet und Status in Proxmox geprüft.

#Phase 3: Datenbank und Cache einrichten
Container 200 (db) konfiguriert: PostgreSQL installiert.

Datenbank erstellt: paperless mit eigenem Benutzer und Passwort.

PostgreSQL konfiguriert: listen_addresses = '*' gesetzt.

Zugriff erlaubt: pg_hba.conf um Eintrag für 10.10.10.0/24 erweitert.

PostgreSQL neu gestartet.

Container 201 (redis) konfiguriert: Redis installiert.

Redis konfiguriert: bind 0.0.0.0 gesetzt.

Redis neu gestartet.

#Phase 4: Paperless-ngx installieren
Container 202 (paperless) vorbereitet: Docker und Docker Compose installiert.

Verzeichnis erstellt: /opt/paperless.

Docker-Compose-Datei heruntergeladen von offizieller Quelle.

Compose-Datei angepasst: Datenbank- und Redis-Abschnitte entfernt, Verbindung zu den separaten Containern eingetragen.

.env-Datei erstellt: Zugangsdaten für Datenbank, Redis und Zeitzone.

Ordner angelegt: data, media, export, consume.

Paperless gestartet mit docker compose up -d.

Test: Paperless im Browser geöffnet, Admin-Account erstellt.

Erstes Dokument hochgeladen und Verarbeitung geprüft.

#Phase 5: Automatisierung einrichten
Backup-Skript erstellt (paperless-db-backup.sh) im Container 200:

Erstellt pg_dump der Paperless-Datenbank.

Speichert in /var/backups/paperless/.

Behält die letzten 7 Backups.

Selbstheilungs-Skript erstellt (paperless-check.sh) auf dem Proxmox-Host:

Prüft alle 10 Minuten, ob Container 200/201/202 laufen.

Startet nicht-laufende Container automatisch.

Wartungs-Skript erstellt (paperless-wartung.sh) auf dem Proxmox-Host:

Ruft Backup und Check nacheinander auf.

Cronjobs eingerichtet:

Alle 10 Minuten: Selbstheilungs-Check.

Täglich 03:00 Uhr: Komplette Wartung mit Backup.

#Phase 6: Proxmox vzdump-Backup
Backup-Job in Proxmox-GUI erstellt:

Ziel: Lokaler Speicher.

Modus: Snapshot.

Komprimierung: ZSTD.

Auswahl: Alle Container.

Zeitplan: Täglich nachts.

Backup-Job getestet und Status geprüft.

#Phase 7: PatchMon installieren
Container 203 (patchmon) erstellt: Ubuntu 22.04, 2 GB RAM, Heimnetz per DHCP.

Docker und Docker Compose installiert.

Verzeichnis erstellt: /opt/patchmon.

Docker-Compose-Datei heruntergeladen von offizieller Quelle.

.env-Datei erstellt mit Datenbank-, Redis- und Server-Konfiguration.

PatchMon gestartet mit docker compose up -d.

Proxmox-Integration eingerichtet: Auto-Enrollment-Token erstellt, Befehl auf Proxmox-Host ausgeführt.

Alle LXC-Container in PatchMon sichtbar und überwacht.

#Phase 8: Feste IP-Adressen vergeben
Container 202 (paperless) angepasst: net1 auf statische IP im gewünschten Netz umgestellt.

Container 203 (patchmon) angepasst: net0 auf statische IP umgestellt.

Proxmox-Host angepasst: vmbr0 auf passende IP im neuen Netz umgestellt.

Docker-Konfigurationen geprüft: SERVER_HOST und CORS_ORIGIN in PatchMon .env angepasst.

#Phase 9: Dokumentation erstellt
Architektur dokumentiert: Schichtenmodell, Container-Übersicht, Datenfluss.

Software-Vergleiche erstellt: Backup, DMS, Patch-Monitoring, Automatisierung.

RTO definiert: Ziele für verschiedene Ausfallszenarien.

Monitoring-Konzept beschrieben: Host-, Container- und Dienst-Ebene.

Automatisierungskonzept dokumentiert: Stufen 1–3.

Zeitplan festgehalten: Rückblick und Ausblick.

Anleitungen und Tutorials gesammelt.

Spickzettel mit wichtigen Befehlen erstellt.

Notfall-Wiederherstellung beschrieben.

Zusammenfassung: Alle Container im Überblick
ID	Name	Aufgabe	IP (intern)	IP (extern)
200	db	PostgreSQL	10.10.10.2	–
201	redis	Redis-Cache	10.10.10.3	–
202	paperless	Paperless-ngx	10.10.10.4	statisch (z.B. 192.168.89.210)
203	patchmon	Patch-Monitoring	–	statisch (z.B. 192.168.89.211)
Alle Dienste im Überblick
Dienst	Zweck
Paperless-ngx	Dokumentenmanagement mit OCR
PostgreSQL	Datenbank für Paperless
Redis	Cache und Warteschlange
PatchMon	Patch- und Update-Monitoring
vzdump	Backup aller Container
Backup-Skript	Täglicher pg_dump
Selbstheilung	Automatischer Neustart bei Ausfall
Cronjobs	Zeitsteuerung aller Aufgaben
