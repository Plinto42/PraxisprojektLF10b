
1. Projektabschluss und Anforderungsprüfung
1.1 Wurde das Projekt beendet?
Ja. Das Projekt wurde vollständig umgesetzt und abgeschlossen. Alle geplanten Dienste laufen produktiv auf dem Proxmox-Notebook.

1.2 Wurden die Anforderungen umgesetzt?
Anforderung	Umsetzung	Status
Container-Infrastruktur aufbauen	4 LXC-Container (db, redis, paperless, patchmon)	
Dokumentenmanagement bereitstellen	Paperless-ngx 
Datenbank integrieren	PostgreSQL in separatem Container	
Cache integrieren	Redis in separatem Container	
Netzwerk segmentieren	Internes Netz vmbr1 (10.10.10.0/24)	
Backup einrichten	pg_dump + vzdump + Rotation	
Monitoring umsetzen	PatchMon + Cron-Selbstheilung
Automatisierung umsetzen	3 Skripte + 2 Cronjobs	
Dokumentation erstellen	Architektur, Vergleiche, RTO, Monitoring	
Ergebnis: Alle Kernanforderungen wurden erfüllt.

1.3 Wiederanlaufplan – aktuell und getestet?
Ja, ein Wiederanlaufplan existiert und wurde getestet.

Getestete Szenarien:

Container-Ausfall: Selbstheilungs-Cron wurde getestet durch manuelles Stoppen von Container 201 → wurde automatisch neu gestartet.

Datenbank-Backup: pg_dump-Skript wurde mehrfach ausgeführt.

vzdump-Restore: Backup-Job in Proxmox GUI getestet, Backup-Dateien im Speicher sichtbar.

Dokumentierter Wiederanlaufplan:

Szenario	Maßnahme	RTO
Container hängt	Selbstheilungs-Cron (≤ 10 Min)	≤ 10 Min
Paperless-Container defekt	docker compose up -d	~5 Min
Datenbank beschädigt	Letztes pg_dump einspielen	~30 Min
Proxmox-Host neu aufsetzen	vzdump-Restore	2–4 h
SSD komplett defekt	vzdump-Restore von extern	4–8 h
Einschränkung: Der Wiederanlaufplan für den Totalausfall der SSD wurde theoretisch dokumentiert, aber nicht praktisch getestet – dafür fehlt aktuell ein externes Backup-Ziel.

2. Reflexion des Projektergebnisses
2.1 Welcher Stand an Ausfallsicherheit konnte erreicht werden?
Erreichter Stand:

Ebene	Maßnahme	Ausfallsicherheit
Container-Ebene	Selbstheilung alle 10 Min	Container-Ausfall wird automatisch behoben
Daten-Ebene	Tägliches pg_dump mit 7 Generationen	Datenbankverlust max. 1 Tag
System-Ebene	vzdump-Backup aller Container	Kompletter Container wiederherstellbar
Dienst-Ebene	PatchMon-Überwachung	Update-Status sichtbar
Hardware-Ebene	Notebook-Akku als USV	Überbrückt kurze Stromausfälle


2.2 Was sind die größten verbleibenden Risiken?
Risiko	                                    Auswirkung	                     Wahrscheinlichkeit	                          Maßnahme
SSD-Ausfall ohne externes Backup	Totalverlust aller Daten und Backups	        Hoch	                         Externe USB-Platte einrichten
Notebook überhitzt	Hardware-Tod, ungeplanter Ausfall	                          Mittel	                       Temperatur-Monitoring einrichten
Kein E-Mail-Alerting 	            Fehler werden zu spät bemerkt	                Hoch	                         msmtp einrichten
Docker-Container hängt in Endlosschleife	Dienst nicht erreichbar	              Niedrig	                       Docker-Healthchecks
Stromausfall > Akkulaufzeit	      Unsauberes Herunterfahren	                    Niedrig	                       
Kein Backup der Docker-Volumes	  Paperless-Datenverlust bei Volume-Crash	      Mittel	                       Volume-Backup erweitern


Das größte Risiko ist der SSD-Ausfall ohne externes Backup. Alle Sicherungen liegen aktuell auf derselben physischen Platte wie die Daten – ein Hardware-Defekt würde alles gleichzeitig zerstören.

3. Umfang der Automatisierung
3.1 Was wurde automatisiert?
Automatisierung	Umsetzung	Rhythmus
Selbstheilung	paperless-check.sh prüft Container-Status	Alle 10 Minuten
Datenbank-Backup	paperless-db-backup.sh mit Rotation	Täglich 03:00 Uhr
Wartungskombination	paperless-wartung.sh ruft Backup + Check auf	Täglich 03:00 Uhr
vzdump-Backup	Proxmox GUI-Job für alle Container	Nach Zeitplan
Patch-Überwachung	PatchMon erfasst alle LXC automatisch	Permanent
Logging	Alle Cron-Ausgaben in Log-Dateien	Bei jedem Lauf

3.2 Was ist noch nicht automatisiert?
Backup-Kopie auf externe USB-Platte

E-Mail-Benachrichtigung bei Fehlern

Temperatur-Logging des Notebooks

HTTP-Checks für Paperless/PatchMon

Automatische Updates der Container

Infrastructure as Code (Ansible)

Fazit: Die Stufe 1 der Automatisierung ist vollständig umgesetzt. Stufe 2 (USB-Backup, Alerting, Temperatur) ist geplant, aber noch nicht realisiert.

4. Lernerfolge
4.1 Technische Lernerfolge
Proxmox VE: Verständnis für LXC-Container, Bridges und Storage-Management.

Netzwerk-Segmentierung: Eigenes internes Netz (vmbr1) für sichere Dienst-Trennung.

Docker & Docker Compose: Container-Orchestrierung mit .env-Dateien und Volumes.

PostgreSQL: Datenbank-Erstellung, Benutzerverwaltung, pg_dump.

Redis: Konfiguration als Cache und Warteschlange.

Paperless-ngx: Installation, Konfiguration.

PatchMon: Patch-Monitoring mit Proxmox-Auto-Enrollment.

Shell-Skripting: Backup-, Check- und Wartungsskripte geschrieben.

Cron: Zeitsteuerung und Log-Umleitung.

Troubleshooting: Log-Analyse, Docker-Fehler beheben, Host-Mismatch lösen
