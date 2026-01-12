### Überblick: Was ist NFSv4 Locking?

**NFSv4** integriert **Datei-Locking vollständig in das Protokoll.**
Im Gegensatz zu **NFSv2/v3 gibt es keinen separaten Lock-Daemon** (rpc.statd, lockd) mehr.

👉 Locking ist zustandsbehaftet (stateful) und Teil der normalen NFS-Kommunikation.
___

### Architektur auf einen Blick

#### Zentrale Komponenten

|Komponente|Aufgabe|
|----------|-------|
NFSv4 Client	Verwaltet Lock-States, Open-States und Sessions
NFSv4 Server	Hält Lock-Zustände persistent
Lease-Mechanismus	Zeitlich begrenzte Gültigkeit der Locks
Compound RPCs	Mehrere Operationen in einem Request

Locking-Workflow (vereinfacht)

OPEN

Client öffnet eine Datei

Server erzeugt einen Open State

LOCK

Client fordert einen Byte-Range-Lock an

Server vergibt Lock, sofern verfügbar

Lease

Lock ist an eine Lease-Zeit gebunden

Client muss regelmäßig bestätigen (Renew)

UNLOCK / CLOSE

Lock wird explizit oder implizit freigegeben

Lease-Mechanismus (wichtig!)

Standard-Lease: 30–90 Sekunden

Wenn der Client nicht rechtzeitig erneuert:

Server verwirft alle Locks

Schutz gegen:

Client-Crashes

Netzwerkausfälle

Recovery nach Server-Neustart
Problem

Server verliert Lock-State im RAM

Lösung in NFSv4

Clients erkennen den Neustart

Reclaim Locks

Kein statd-Recovery nötig

Unterschiede zu NFSv3
Feature	NFSv3	NFSv4
Locking	Extern (lockd)	Integriert
Zustand	Stateless	Stateful
Firewall-freundlich	❌	✅ (Port 2049)
Recovery	Komplex	Automatisch
Typische Lock-Arten

Advisory Locks

Kooperativ

Applikationsabhängig

Byte-Range Locks

Teilbereiche von Dateien

Share Reservations

Verhindert konkurrierendes Öffnen

Vorteile der NFSv4-Locking-Architektur

✅ Weniger Daemons
✅ Firewall-freundlich
✅ Konsistentes Locking
✅ Bessere Crash-Recovery
✅ Für Cluster & HA geeignet

Häufige Probleme & Ursachen
ProblemUrsacheLocks verschwindenLease abgelaufen„Stale stateid“Server-RestartPerformanceZu kurze Lease-ZeitHänger bei IOLock-Contention

Praxis-Tipps


Lease-Zeit prüfen
cat /proc/fs/nfsd/nfsv4leasetime



Für Cluster:


Stabile Zeit (NTP!)


Gemeinsamer Storage




Für Datenbanken:


Besser lokales FS oder Cluster-FS




Hier ist NFSv4-Locking erklärt am konkreten Szenario „VM-Storage“ (z. B. mehrere Hypervisor greifen auf ein gemeinsames NFS-Share zu).

Szenario: VM-Storage auf NFSv4
Ausgangslage

Mehrere Hypervisor-Hosts (z. B. KVM/Proxmox)

Gemeinsames NFSv4-Share

VM-Disks liegen als Dateien (vm-100-disk-0.qcow2) auf NFS

Nur ein Host darf eine VM gleichzeitig schreiben

➡️ Locking ist kritisch, sonst droht Dateisystem-Korruption

Schritt-für-Schritt: Locking beim VM-Start
1️⃣ VM wird gestartet (Host A)

Ablauf:

Hypervisor öffnet VM-Disk

NFSv4:

OPEN → Open State

LOCK → Exclusive Byte-Range Lock (0–EOF)

Server vergibt Lock

VM startet

📌 Ergebnis:

Host A besitzt exklusiven Schreibzugriff

Host B wird blockiert

2️⃣ Zweiter Startversuch (Host B)

Host B versucht:

OPEN + LOCK

Server antwortet:

❌ Lock denied (EAGAIN / EWOULDBLOCK)

➡️ VM kann nicht doppelt gestartet werden
➡️ Datenschutz & Konsistenz gewährleistet

Lease-Mechanismus im VM-Betrieb
Während die VM läuft:

Host A erneuert regelmäßig seine Lease

Erfolgt automatisch über normale IOs

Lease-Typisch: 30–90 Sekunden

Vorteil:

Kein permanenter Heartbeat nötig

Geringe Netzwerk-Last

Crash-Szenario: Hypervisor stürzt ab
Problem

Host A ist weg

Lock ist noch aktiv

Lösung durch NFSv4

Lease von Host A läuft ab

Server verwirft:

Open State

Lock State

Host B kann VM starten

⏱️ Downtime = Lease-Zeit

📌 Kein manuelles Unlock nötig

Server-Neustart (Storage-Reboot)
Was passiert?

Server verliert Lock-State im RAM

NFSv4-Recovery:

Server signalisiert „Grace Period“

Clients:

Reclaim Locks

Laufende VMs:

IO kurz pausiert

Danach Weiterbetrieb

⚠️ Voraussetzung:

Stabile Client-IDs

Zeit-Synchronisation (NTP!)

Warum NFSv4 gut für VM-Storage ist
Vorteil	Bedeutung
Stateful Locking	Schutz vor Doppelstarts
Lease-basiert	Automatisches Recovery
Port 2049	Firewall-freundlich
Kein lockd/statd	Weniger Fehlerquellen
Byte-Range Locks	Ganze Disk exklusiv sperrbar
Typische Fehler & ihre Ursachen
Symptom	Ursache
VM „hängt“ beim Start	Lock-Contention
VM startet zu früh nach Crash	Lease zu kurz
IO-Freezes	Server in Grace Period
„stale stateid“	Storage-Reboot
Best Practices für VM-Storage auf NFSv4

✅ NFSv4.1 oder 4.2 verwenden
✅ Lease nicht zu kurz konfigurieren
✅ NTP auf allen Hosts
✅ Keine gleichzeitigen Mounts als rw außerhalb des Clusters
✅ Für DB-intensive VMs ggf. lokale Disks bevorzugen

Kurzfassung

NFSv4-Locking ist der Sicherheitsgurt für VM-Storage
Ohne Locking → Datenverlust
Mit NFSv4 → kontrollierter, clusterfähiger Betrieb
