### Überblick: Was ist NFSv4 Locking?

**NFSv4** integriert **Datei-Locking vollständig in das Protokoll.**
Im Gegensatz zu **NFSv2/v3 gibt es keinen separaten Lock-Daemon** (rpc.statd, lockd) mehr.

👉 Locking ist zustandsbehaftet (stateful) und Teil der normalen NFS-Kommunikation.
___

### Architektur auf einen Blick

#### Zentrale Komponenten

|Komponente|Aufgabe|
|----------|-------|
|**NFSv4 Client**|Verwaltet Lock-States, Open-States und Sessions|
|**NFSv4 Server**|Hält Lock-Zustände persistent|
|**Lease-Mechanismus**|Zeitlich begrenzte Gültigkeit der Locks|
|**Compound RPCs**|Mehrere Operationen in einem Request|

### Locking-Workflow (vereinfacht)

1. OPEN
  - Client öffnet eine Datei
  - Server erzeugt einen Open State

2. LOCK
  - Client fordert einen Byte-Range-Lock an
  -Server vergibt Lock, sofern verfügbar

3. Lease
   - Lock ist an eine Lease-Zeit gebunden
   - Client muss regelmäßig bestätigen (Renew)

4. UNLOCK / CLOSE
  - Lock wird explizit oder implizit freigegeben

___

### Lease-Mechanismus (wichtig!)

- Standard-Lease: 30–90 Sekunden
- Wenn der Client nicht rechtzeitig erneuert:
  - Server verwirft alle Locks

- Schutz gegen:
  - Client-Crashes
  - Netzwerkausfälle

___

### Recovery nach Server-Neustart

**Problem**
- Server verliert Lock-State im RAM

**Lösung in NFSv4**
- Clients erkennen den Neustart
- Reclaim Locks
- Kein statd-Recovery nötig
___

### Unterschiede zu NFSv3

|Feature|	NFSv3|	NFSv4|
|--------|-------|--------|
|Locking|	Extern (lockd)|	Integriert|
|Zustand|	Stateless|	Stateful|
|Firewall-freundlich|	❌|	✅ (Port 2049)|
|Recovery|	Komplex	|Automatisch<

___

### Typische Lock-Arten

- Advisory Locks
  - Kooperativ
  - Applikationsabhängig

- Byte-Range Locks
  - Teilbereiche von Dateien

- Share Reservations
  - Verhindert konkurrierendes Öffnen
___

### Vorteile der NFSv4-Locking-Architektur

✅ Weniger Daemons

✅ Firewall-freundlich

✅ Konsistentes Locking

✅ Bessere Crash-Recovery

✅ Für Cluster & HA geeignet
___

### Häufige Probleme & Ursachen

|Problem| Ursache|
|-------|--------|
|Locks verschwinden|Lease abgelaufen|
|„Stale stateid“|Server-Restart|
|Performance|Zu kurze Lease-Zeit|
|Hänger bei IO|Lock-Contention|
____

### Praxis-Tipps

- Lease-Zeit prüfen

```bash
cat /proc/fs/nfsd/nfsv4leasetime
```

- Für Cluster:
  - Stabile Zeit (NTP!)
  - Gemeinsamer Storage

- Für Datenbanken:
  - Besser lokales FS oder Cluster-FS

## NFSv4-Locking erklärt am konkreten Szenario „VM-Storage“ (z. B. mehrere Hypervisor greifen auf ein gemeinsames NFS-Share zu).

### Szenario: VM-Storage auf NFSv4

**Ausgangslage**
- Mehrere **Hypervisor-Hosts** (z. B. KVM/Proxmox)
- Gemeinsames **NFSv4-Share**
- VM-Disks liegen als Dateien (vm-100-disk-0.qcow2) auf NFS
- **Nur ein Host darf eine VM gleichzeitig schreiben**

➡️ Locking ist kritisch, sonst droht Dateisystem-Korruption
___

### Schritt-für-Schritt: Locking beim VM-Start
**1️⃣ VM wird gestartet (Host A)**

**Ablauf:**

1. Hypervisor öffnet VM-Disk
2. NFSv4:
  - OPEN → Open State
  - LOCK → Exclusive Byte-Range Lock (0–EOF)
3. Server vergibt Lock
4. VM startet

📌 Ergebnis:
- Host A besitzt exklusiven Schreibzugriff
- Host B wird blockiert
___

**2️⃣ Zweiter Startversuch (Host B)**

- Host B versucht:
  - OPEN + LOCK
- Server antwortet:
  - ❌ Lock denied (EAGAIN / EWOULDBLOCK)

➡️ VM kann **nicht doppelt gestartet** werden
**➡️ Datenschutz & Konsistenz gewährleistet**
___

### Lease-Mechanismus im VM-Betrieb
**Während die VM läuft:**
- Host A erneuert regelmäßig seine Lease
- Erfolgt automatisch über normale IOs
- Lease-Typisch: **30–90 Sekunden**

### Vorteil:
- Kein permanenter Heartbeat nötig
- Geringe Netzwerk-Last
___

### Crash-Szenario: Hypervisor stürzt ab
**Problem**
- Host A ist weg
- Lock ist noch aktiv

### Lösung durch NFSv4
1. Lease von Host A läuft ab
2. Server verwirft:
  - Open State
  - Lock State
3. Host B kann VM starten

⏱️ Downtime = Lease-Zeit

📌 Kein manuelles Unlock nötig
___

### Server-Neustart (Storage-Reboot)
**Was passiert?**
- Server verliert Lock-State im RAM

**NFSv4-Recovery:**
1. Server signalisiert „Grace Period“
2. Clients:
  - Reclaim Locks
3. Laufende VMs:
  - IO kurz pausiert
  - Danach Weiterbetrieb

⚠️ Voraussetzung:

- **Stabile Client-IDs**
- **Zeit-Synchronisation (NTP!)**

___

### Warum NFSv4 gut für VM-Storage ist
|Vorteil|	Bedeutung|
|-------|------------|
|Stateful Locking|	Schutz vor Doppelstarts|
|Lease-basiert	|Automatisches Recovery|
|Port 2049|	Firewall-freundlich|
|Kein lockd/statd|	Weniger Fehlerquellen|
|Byte-Range Locks|	Ganze Disk exklusiv sperrbar|

### Typische Fehler & ihre Ursachen
|Symptom|	Ursache|
|-------|----------|
|VM „hängt“ beim Start|	Lock-Contention|
|VM startet zu früh nach Crash|	Lease zu kurz|
|IO-Freezes	|Server in Grace Period|
|„stale stateid“|	Storage-Reboot|

### Best Practices für VM-Storage auf NFSv4

**✅ NFSv4.1 oder 4.2 verwenden**
**✅ Lease nicht zu kurz konfigurieren**
**✅ NTP auf allen Hosts**
**✅ Keine gleichzeitigen Mounts als rw außerhalb des Clusters**
**✅ Für DB-intensive VMs ggf. lokale Disks bevorzugen**

### Kurzfassung

NFSv4-Locking ist der Sicherheitsgurt für VM-Storage
Ohne Locking → Datenverlust
Mit NFSv4 → kontrollierter, clusterfähiger Betrieb

