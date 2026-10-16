# Jellyfin-Stack

Aufbau und Integration eines automatisierten Media-Server-Stacks mit Jellyfin, Radarr, Sonarr, Prowlarr, qBittorrent und Seerr auf Proxmox mit Debian-LXC – ohne Docker.

## Überblick

In diesem Projekt habe ich eine selbst gehostete Medienplattform aufgebaut, bei der mehrere Dienste miteinander verbunden sind, um Filme und Serien zu verwalten, Downloads zu automatisieren und Medien in einer zentralen Bibliothek bereitzustellen.

Als Grundlage diente Jellyfin, das ich mit einer eigenen Speicherstruktur, Nextcloud und verschiedenen Automatisierungsdiensten kombiniert habe. Dadurch gab es mehrere Möglichkeiten, neue Medien hinzuzufügen: entweder über den Upload in Nextcloud oder über einen automatisierten Workflow mit Seerr, Radarr, Sonarr, Prowlarr und qBittorrent.

Zusätzlich habe ich mit Unterstützung von Claude AI ein Hintergrundskript zur automatisierten Dateiübernahme sowie eine eigene Weboberfläche für das koordinierte Löschen von Medien entwickelt.

Der gesamte Stack wurde auf Proxmox mit Debian-LXC-Containern umgesetzt. Die Dienste wurden ohne Docker installiert und miteinander verbunden.

## 1. Infrastruktur und Speicherarchitektur

Die Grundlage des Systems bildete eine Debian-LXC-Umgebung auf Proxmox. Für die Mediendateien habe ich einen zusätzlichen Datenträger auf dem Proxmox-Host unter `/mnt/media` eingebunden und diesen Speicher über einen Bind-Mount im LXC-Container verfügbar gemacht.

Dadurch konnten die Anwendungen auf eine gemeinsame Medienablage zugreifen.

Die Speicherstruktur war in mehrere Verzeichnisse aufgeteilt:

- `/mnt/media/downloads` – Ablage für abgeschlossene und laufende Downloads
- `/mnt/media/filme` – zentrale Ablage für Filme
- `/mnt/media/serien` – zentrale Ablage für Serien

Jellyfin war der zentrale Dienst für die Wiedergabe der Medien. Ich habe dafür zwei separate Bibliotheken angelegt: eine für Filme und eine für Serien. Beide Bibliotheken griffen auf die entsprechenden Verzeichnisse des gemeinsamen Datenträgers zu.

Die einzelnen Anwendungen wurden als Systemdienste eingerichtet, sodass sie innerhalb der Debian-Umgebung unabhängig voneinander gestartet und verwaltet werden konnten.

## 2. Nextcloud als zusätzlicher Weg zur Medienübernahme

Neben Jellyfin habe ich Nextcloud als Dateiablage eingerichtet. Dafür gab es zwei getrennte Benutzerkonten mit unterschiedlichen Aufgaben.

Das erste Konto diente als normale persönliche Cloud, um Dateien und andere Daten zu speichern. Das zweite Konto war speziell für die Übernahme von Mediendateien vorgesehen.

### Automatisierte Dateiübernahme

Damit hochgeladene Videos nicht manuell aus Nextcloud heraus verschoben werden mussten, habe ich mit Unterstützung von Claude AI ein Hintergrundskript entwickelt.

Das Skript überprüfte einmal pro Minute, ob eine neue Videodatei im vorgesehenen Bereich vorhanden war. Dabei wurde berücksichtigt, dass ein Upload möglicherweise noch nicht abgeschlossen war.

Der Ablauf war folgendermaßen:

1. Das Skript überprüfte, ob eine neue Videodatei vorhanden war.
2. Anschließend wurde kontrolliert, ob der Upload vollständig abgeschlossen war.
3. War die Datei noch nicht vollständig übertragen, wartete das Skript, bevor es den Vorgang erneut prüfte.
4. Nach Abschluss des Uploads wurde die Videodatei in den Filmordner von Jellyfin verschoben.
5. Anschließend wurden die Medienbibliothek von Jellyfin und der Dateibestand von Nextcloud aktualisiert.

Durch die erneute Erfassung in Nextcloud wurden die Dateieinträge mit den tatsächlichen Änderungen im Dateisystem abgeglichen. Jellyfin konnte die neu hinzugefügte Datei dagegen in seine Bibliothek aufnehmen.

Damit war es möglich, Medien durch einen einfachen Upload in Nextcloud zur Jellyfin-Bibliothek hinzuzufügen, ohne die Datei anschließend manuell verschieben und die Bibliothek selbst aktualisieren zu müssen.

Das Hintergrundskript zur Dateiübernahme wird in diesem Repository nicht separat bereitgestellt.

## 3. Automatisierter Medien-Workflow

Der zentrale Teil des Projekts war die Integration von Seerr, Radarr, Sonarr, Prowlarr und qBittorrent. Die einzelnen Dienste hatten unterschiedliche Aufgaben und wurden so miteinander verbunden, dass sie einen durchgängigen Workflow bildeten.

### Aufgaben der einzelnen Dienste

| Dienst | Aufgabe |
|---|---|
| **Seerr** | Weboberfläche für Film- und Serienanfragen |
| **Radarr** | Verwaltung von Filmen und deren Verfügbarkeit |
| **Sonarr** | Verwaltung von Serien und deren Verfügbarkeit |
| **Prowlarr** | Zentrale Verwaltung und Bereitstellung der konfigurierten Suchquellen |
| **qBittorrent** | Durchführung und Verwaltung der Downloads |
| **Jellyfin** | Organisation der Medienbibliothek und Wiedergabe der Inhalte |

### Ablauf einer Medienanfrage

**1. Anfrage über Seerr**

Über die Weboberfläche von Seerr konnte ein Film oder eine Serie angefragt werden. Abhängig vom Medientyp wurde die Anfrage an Radarr oder Sonarr weitergeleitet.

**2. Suche nach verfügbaren Medien**

Radarr und Sonarr suchten über die in Prowlarr konfigurierten Quellen nach passenden Angeboten. Dabei wurden die jeweils eingestellten Qualitätsanforderungen berücksichtigt.

**3. Übergabe an qBittorrent**

Wurde eine passende Quelle gefunden, übergab Radarr beziehungsweise Sonarr den Download-Auftrag an qBittorrent.

**4. Download und Sortierung**

qBittorrent speicherte die heruntergeladenen Dateien zunächst unter `/mnt/media/downloads`. Nach Abschluss des Downloads übernahmen Radarr oder Sonarr die Dateien und ordneten sie dem jeweiligen Medienbereich zu.

Filme wurden nach `/mnt/media/filme` und Serien nach `/mnt/media/serien` verschoben beziehungsweise importiert.

**5. Bereitstellung in Jellyfin**

Jellyfin griff auf die Medienverzeichnisse zu und stellte die enthaltenen Filme und Serien in den jeweiligen Bibliotheken zur Verfügung. Dadurch waren die Medien nach ihrer Verarbeitung zentral über Jellyfin erreichbar.

Durch diese Integration mussten die einzelnen Schritte nicht mehr separat in jedem Programm durchgeführt werden. Die Dienste übernahmen die jeweiligen Aufgaben und gaben die benötigten Informationen an die nächste Komponente weiter.

## 4. Eigene Weboberfläche zum koordinierten Löschen von Medien

Ein weiteres Problem trat beim Löschen von Filmen und Serien auf.

Wenn eine Datei direkt über Jellyfin gelöscht wurde, waren die entsprechenden Einträge in Radarr oder Sonarr unter Umständen weiterhin vorhanden. Bei späteren Überprüfungen konnten diese Anwendungen feststellen, dass die Datei fehlte, und einen erneuten Download veranlassen. Dadurch konnte ein bereits gelöschter Film kurze Zeit später wieder auf dem Datenträger erscheinen.

Auch in Seerr konnte eine Anfrage weiterhin als bestehend angezeigt werden, obwohl das zugehörige Medium eigentlich entfernt werden sollte.

Um dieses Problem zu lösen, habe ich mit Unterstützung von Claude AI eine zusätzliche Weboberfläche entwickelt. Sie koordinierte den Löschvorgang über die beteiligten Anwendungen hinweg.

### Ablauf des Löschvorgangs

**1. Medien auswählen**

Die Weboberfläche listete die verfügbaren Filme auf. Dort konnte ich einen Film zum Löschen auswählen.

**2. Löschung bestätigen**

Vor dem eigentlichen Vorgang musste die Löschung noch einmal bestätigt werden. Dadurch wurde verhindert, dass ein einzelner Klick unmittelbar die Entfernung auslöste.

**3. Zeitfenster für weitere Löschanträge**

Nach der Bestätigung startete ein Countdown von einer Minute. Während dieses Zeitfensters konnten weitere Filme zur Löschwarteschlange hinzugefügt werden.

Bei jedem weiteren Löschantrag wurde der Countdown erneut auf eine Minute zurückgesetzt. Außerdem bestand die Möglichkeit, den gesamten Vorgang abzubrechen.

**4. Löschwarteschlange verarbeiten**

Nach Ablauf des Countdowns wurde die Bearbeitung gestartet. Währenddessen waren vorübergehend keine weiteren Löschvorgänge möglich. Alle zuvor ausgewählten Medien wurden verarbeitet.

Dabei wurden nicht nur die entsprechenden Dateien entfernt, sondern auch die zugehörigen Einträge in Radarr und Sonarr gelöscht. Zusätzlich wurde die jeweilige Anfrage aus Seerr entfernt, sodass das Medium dort nicht mehr als bereits angefragt geführt wurde und erneut angefragt werden konnte.

**5. Jellyfin-Bibliothek aktualisieren**

Nach Abschluss des Löschvorgangs wurde ein Bibliotheksscan in Jellyfin ausgelöst. Dadurch wurden die entfernten Medien aus der Bibliotheksanzeige abgeglichen und nicht mehr vorhandene Dateien sollten nicht länger als verfügbare Inhalte angezeigt werden.

### Ziel der zusätzlichen Oberfläche

Die Weboberfläche sollte sicherstellen, dass ein Löschvorgang nicht nur die Datei auf dem Datenträger betrifft, sondern auch die dazugehörigen Einträge in den Verwaltungsdiensten berücksichtigt.

Dadurch wurden die Medienverwaltung, die tatsächlichen Dateien und die Anfragen in Seerr konsistenter gehalten. Gleichzeitig ermöglichte die Löschwarteschlange, mehrere Medien gesammelt zu entfernen, ohne jeden Vorgang sofort einzeln auszuführen.

## 5. Technische Herausforderungen

Beim Aufbau und bei der Integration des Stacks mussten verschiedene Probleme gelöst werden.

### Speicherberechtigungen in LXC

Da der zusätzliche Datenträger auf dem Proxmox-Host eingebunden und anschließend in einen unprivilegierten LXC-Container durchgereicht wurde, mussten die Berechtigungen für den Zugriff auf die Medienverzeichnisse korrekt eingerichtet werden. Andernfalls konnten einzelne Dienste Dateien nicht wie vorgesehen lesen, verschieben oder verwalten.

### Kommunikation zwischen den Diensten

Die einzelnen Anwendungen mussten korrekt miteinander verbunden werden. Dazu gehörten die Konfiguration der Weboberflächen, die entsprechenden Verbindungen zwischen den Diensten und die notwendigen API-Zugänge.

In der verwendeten Umgebung war bei bestimmten Verbindungen außerdem die erreichbare LAN-Adresse erforderlich, anstatt sich auf `localhost` zu verlassen.

### Suchquellen und Indexer

Bei der Anbindung der Suchquellen traten Probleme mit einzelnen Indexern und Cloudflare-Schutzmechanismen auf. Diese mussten bei der Konfiguration und Fehlersuche berücksichtigt werden.

### Stabilität der Anwendungsversionen

Auch die verwendete Version von Seerr spielte eine Rolle. Nach Problemen mit dem Entwicklungsstand der Hauptversion wurde eine stabilere Version verwendet, um die Integration zuverlässiger betreiben zu können.

Diese Herausforderungen betrafen nicht nur die Installation einzelner Anwendungen, sondern vor allem deren Zusammenspiel innerhalb einer gemeinsamen Infrastruktur.

## 6. Ergebnis

Mit diesem Projekt entstand ein integrierter Media-Server-Stack, der Medienablage, Verwaltung, Download-Automatisierung und Wiedergabe miteinander verband.

Zu den wesentlichen Bestandteilen gehörten:

- Eine zentrale Medienbibliothek mit Jellyfin
- Eine gemeinsame Speicherstruktur auf einem zusätzlichen Datenträger unter Proxmox
- Nextcloud als Dateiablage mit automatisierter Übernahme hochgeladener Videos
- Ein integrierter Workflow aus Seerr, Radarr, Sonarr, Prowlarr und qBittorrent
- Eine mit Unterstützung von Claude AI entwickelte Weboberfläche zur koordinierten Medienlöschung
- Die Abstimmung der Dateibestände und Einträge zwischen den beteiligten Diensten
- Der Betrieb unter Debian-LXC auf Proxmox ohne Docker

Das Projekt umfasste damit nicht nur die Einrichtung einzelner Dienste, sondern auch die Integration mehrerer Anwendungen, die Verwaltung gemeinsamer Speicherpfade sowie die Entwicklung zusätzlicher Automatisierung für Aufgaben, die mit den vorhandenen Weboberflächen allein nicht zuverlässig abgedeckt waren.
