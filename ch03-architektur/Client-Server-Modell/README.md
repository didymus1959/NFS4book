Hier ist eine **übersichtliche Erklärung der Architektur von NFSv4 im Client-Server-Modell**, von den Grundkomponenten bis zu den Besonderheiten von NFSv4.
___

+--------------------+           TCP / Port 2049           +--------------------+
|     NFSv4 Client   | <--------------------------------> |     NFSv4 Server   |
|--------------------|                                     |--------------------|
|  VFS / POSIX API   |  OPEN / READ / WRITE / LOCK (RPC)  |  NFSv4 Daemon      |
|--------------------|------------------------------------>|  (nfsd)           |
|  NFSv4 Client      |                                     |--------------------|
|  - File Handles    | <----- Responses / Status -------- |  - File Handles   |
|  - Caching         |                                     |  - Locks          |
|  - Client State    |                                     |  - Sessions       |
|--------------------|                                     |--------------------|
|  Auth (AUTH_SYS /  | ===== RPCSEC_GSS (Kerberos) =====> |  Auth & ACLs      |
|  Kerberos)         |                                     |--------------------|
+--------------------+                                     +--------------------+
          |                                                           |
          | mount server:/                                           | export /
          v                                                           v
   +-------------------+                                   +-------------------+
   | Local Mountpoint  |                                   | Server Filesystem |
   | /mnt/nfs          |                                   | (Global Namespace)|
   +-------------------+                                   +-------------------+
___

### 🧩 Grundidee des NFSv4 Client-Server-Modells

**NFSv4 (Network File System Version 4)** erlaubt es Clients, Dateien so zu nutzen, als lägen sie lokal – tatsächlich befinden sie sich auf einem entfernten Server.


- **Client:** fordert Dateioperationen an (lesen, schreiben, sperren)
- **Server:** stellt Dateisysteme bereit und verwaltet Zugriffe
- **Kommunikation:** zustandsbehaftet (stateful) über TCP, meist Port 2049



### 🏗️ Architekturüberblick


#### 1. NFSv4 Client
Typischerweise ein Linux-, UNIX- oder BSD-System.

**Aufgaben:**

- Mountet das entfernte Dateisystem
- Führt POSIX-Dateioperationen aus (open, read, write, lock)
- Hält Client-State (z. B. Datei-Handles, Locks, Sessions)
- Caching (Attribut- & Datencache)
___

#### 2. NFSv4 Server
Stellt die Dateien und Verzeichnisse bereit.

**Aufgaben:**

- Exportiert ein einheitliches Namespace-Root (kein /etc/exports pro Mount wie bei NFSv3)

- Verwalten von:
  - Datei-Locks
  - Client-Sessions
  - Benutzeridentitäten
- Setzt Zugriffsrechte durch (ACLs)
___

#### 3. Protokoll & Transport

|Aspekt|NFSv4|
|:-----|:-----|
|Transport|**TCP only**|
|Port|**2049**|
|Zustand|**Stateful**|
|RPC|ONC RPC integriert|
|Firewalls|Einfach (ein Port)|

👉 Kein rpcbind, kein mountd, kein lockd mehr nötig (alles integriert).
___

### 🔐 Sicherheit & Authentifizierung

#### Sicherheitsmechanismen:

- **AUTH_SYS** (klassisch, UID/GID-basiert)
- **Kerberos (RPCSEC_GSS):**
  - krb5 → Authentifizierung
  - krb5i → Authentifizierung + Integrität
  - krb5p → zusätzlich Verschlüsselung

#### Zugriffskontrolle:

POSIX-Rechte
NFSv4 ACLs (Windows-ähnlich, sehr granular)
___

### 🌳 Namespace-Konzept (wichtig!)
Ein zentraler Unterschied zu NFSv3:

- Ein globaler Namespace
- Client mountet nur ein Root-Export, z. B.:

```bash
mount -t nfs4 server:/ /mnt/nfs
```
Unterhalb dieses Roots liegen alle Exporte logisch eingebunden.
___

### 🔁 Sitzungen, Locks & Recovery

#### Stateful Design:

- Server merkt sich:
  - geöffnete Dateien
  - Locks
  - Client-IDs

#### Vorteile:

Sauberes File-Locking
Bessere POSIX-Semantik

#### Herausforderung:

- **Server-Reboot → Client muss State neu aufbauen**
- Gelöst durch:
  - Lease-Zeiten
   Grace-Period nach Neustart
___

📊 Vergleich zu älteren NFS-Versionen

|Merkmal|NFSv3|NFSv4|
|:------|:---:|----:|
|Zustand| Stateless| Stateful|
|Ports| Viele| Nur 2049|
|Sicherheit| Schwach| Kerberos, ACLs|
|Locks |Extern (lockd)| Integriert|
|Namespace| Einzelne Exporte| Globaler Namespace|
___

### 🧠 Zusammenfassung
**NFSv4 Client-Server-Architektur** zeichnet sich aus durch:
- Zustandsbehaftete Kommunikation
- Einheitlichen Namespace
- Integrierte Sicherheit & Locks
- Firewall-freundliches Design
- Bessere POSIX-Semantik

