## Protokollablauf

### NFSv4 - Überblick

**1. Grundlegende Komponenten**

**NFSv4 ist ein zustandsbehaftetes Client-Server-Protokoll.**

**Server**

- Exportiert Dateisysteme (Exports)
- Verwaltet:
  - Filehandles
  - Locks & Opens (Zustand)
  - ACLs (POSIX + NFSv4 ACLs)
  - Benutzeridentitäten

**Client**

- Mountet ein Export
- Führt Dateioperationen aus (READ, WRITE, OPEN, LOCK)
- Hält lokalen Cache (Daten + Attribute)

**Transport**

- TCP (Port **2049**)
- Kein Portmapper (rpcbind) mehr nötig

___

**2. Protokoll-Stack**

```bash
+-----------------------------+
|   Anwendungen (ls, cp, …)   |
+-----------------------------+
|   VFS (Kernel-Abstraktion)  |
+-----------------------------+
|   NFSv4 Protokoll           |
|   (Compound RPCs)           |
+-----------------------------+
|   RPC over TCP              |
+-----------------------------+
|   IP                        |
```
___

🔐 Sicherheits- und Identitätsmodell
Authentifizierung (RPCSEC_GSS)

AUTH_SYS (klassisch, unsicher)

Kerberos (empfohlen):

krb5 → Auth

krb5i → Auth + Integrität

krb5p → Auth + Verschlüsselung

Identitäten

User/Group als Strings:

user@domain


Mapping via:

idmapd

LDAP / AD

🔁 Protokollablauf (typischer Zugriff)
1. Mount (Pseudo-Filesystem)
Client → Server: GETATTR /
Client → Server: LOOKUP export


⚠️ Kein echtes mountd mehr – alles über NFS selbst.

2. Datei öffnen
PUTFH (Filehandle setzen)
OPEN


✔ Server merkt sich:

Wer die Datei geöffnet hat

Mit welchem Modus (read/write)

3. Lesen / Schreiben
READ
WRITE


Zustandsabhängig

Caching erlaubt, aber serverkoordiniert

4. Locking
LOCK
LOCKU


✔ Locking ist integriert (kein NLM mehr!)

5. Schließen
CLOSE

📦 Compound RPCs (zentrales NFSv4-Merkmal)

Mehrere Operationen in einem Request:

PUTFH → LOOKUP → OPEN → READ → CLOSE


✅ Vorteile:

Weniger Netzwerk-Roundtrips

Atomare Abläufe

Bessere Performance über WAN

🔄 Recovery & Session-Handling
Leases

Server vergibt Leases (Zeitfenster)

Client muss regelmäßig bestätigen

Server-Neustart

Client erkennt „grace period“

Reclaim von:

Opens

Locks

🆚 Unterschiede zu NFSv3 (Kurz)
Feature	NFSv3	NFSv4
Zustand	Stateless	Stateful
Locking	Extra (NLM)	Integriert
Sicherheit	AUTH_SYS	Kerberos
Ports	Viele	Nur 2049
WAN-fähig	Eingeschränkt	Ja
🧠 Merksatz

NFSv4 ist ein sicheres, zustandsbehaftetes, session-orientiertes Dateiprotokoll mit integrierter Sicherheit, Locking und komplexen RPC-Operationen.
