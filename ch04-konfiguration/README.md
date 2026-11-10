## NFSv4 Praxis-Setup unter Linux (Server + Client)

Hier ist ein **praxisnahes, sauberes NFSv4-Setup unter Linux** (Server + Client), so wie man es **real im Alltag** betrereibt – inkl. Sicherheit, Performance-Defaults und typischen Stolperfallen.

### 🎯 Ziel des Setups

- **Zentraler Linux-Server** stellt Verzeichnisse per NFSv4
- **Mehrere Linux-Clients** mounten diese stabil
- Einheitliche **UID/GID**
- Saubere **Rechte, Autostart, Firewall**
- Ohne exotische Zusatzdienste (kein LDAP nötig)

### 1️⃣ Grundannahmen (Best Practice)
|Rolle|	Beispiel|
|-----|---------|
|Server|	server01 – IP 192.168.10.10|
|Clients|	client01, client02|
|Netzwerk|	Internes LAN|
|Freigabe|	/srv/nfs/praxis|
|NFS-Version|	NFSv4.2|
|Sicherheit|	IP-basiert + Root-Squash|

⚠️ **Wichtig**: Konsistente Benutzer- und Gruppenkennungen

Bei NFSv4 mit AUTH_SYS (sec=sys) sollten Benutzer und Gruppen auf NFS-Server und NFS-Clients konsistente numerische UID- und GID-Werte besitzen.
Beispiel: Der Benutzer praxis verwendet auf allen beteiligten Systemen UID 1001 und GID 1001.
Unterschiedliche Kennungen können zu Zugriffsproblemen führen oder dazu, dass ein anderer Benutzer unbeabsichtigt Zugriff auf Dateien erhält.
Bei NFSv4 mit Kerberos ist dagegen eine namensbasierte Authentifizierung mit serverseitiger Identitätszuordnung möglich. Identische numerische UIDs auf allen Systemen sind dann nicht grundsätzlich erforderlich.

**Merksatz**: Bei NFSv4 mit sec=sys sind konsistente UID/GID-Werte entscheidend. Bei NFSv4 mit Kerberos steht dagegen die authentifizierte Benutzeridentität im Vordergrund.
___

### 2️⃣ NFS-Server einrichten
**Pakete installieren**

```bash
apt install nfs-kernel-server
```

**Verzeichnisstruktur (NFSv4-Standard)**

```bash
mkdir -p /srv/nfs/praxis
chown praxis:praxis /srv/nfs/praxis
chmod 2770 /srv/nfs/praxis
```

2 = **setgid**, damit Gruppenrechte erhalten bleiben

___

### 3️⃣ /etc/exports (KERNSTÜCK)

```bash
/srv/nfs        192.168.10.0/24(ro,fsid=0,no_subtree_check)
/srv/nfs/praxis 192.168.10.0/24(rw,sync,root_squash,no_subtree_check)
```

**📌 Erklärung**
- fsid=0 → NFSv4 Root (Pflicht!)
- root_squash → Client-root wird niemals Server-root
- sync → Datensicherheit > Performance

Danach:

```bash
exportfs -ra
systemctl restart nfs-server
```
___

### 4️⃣ Firewall (Server)

```bash
ufw allow from 192.168.10.0/24 to any port 2049 proto tcp
```

NFSv4 nutzt **nur Port 2049** → deutlich sicherer als NFSv3
___

### 5️⃣ Client-Konfiguration
**Pakete installieren**

```bash
apt install nfs-common
```

**Mountpunkt**

```bash
mkdir -p /praxis
```

**Manuell testen**

```bash
mount -t nfs4 server01:/praxis /praxis
```

Wenn das funktioniert → **Automount**
___

### 6️⃣ /etc/fstab **(Client – stabil & sauber)**

```bash
server01:/praxis  /praxis  nfs4  _netdev,noatime,hard,timeo=600,retrans=5  0  0
```

✔️  _netdev → bootet auch ohne Server
✔️  hard → keine stillen Datenverluste
✔️  noatime → Performance

Test:

```bash
mount -a
```
___

### 7️⃣ Benutzer & Rechte (Praxis-tauglich)
**Beispiel:**
```bash
groupadd praxis
useradd -u 1001 -g praxis praxis
```
Auf **allen Systemen identisch!**

**Typische Struktur**

```bash
/srv/nfs/praxis
├── daten
├── scans
└── archive
```

Rechte:
```bash
chmod -R 2770 /srv/nfs/praxis
```
___

### 8️⃣ Performance & Stabilität (empfohlen)
**Server**

```bash
echo "options nfsd nfsdvers=4.2" > /etc/modprobe.d/nfsd.conf
systemctl restart nfs-server
```
Das ist ein klassischer, sinnvoller NFS-Client-Tuning-Parameter, der oft unkommentiert übernommen wird. Ich erkläre ihn technisch sauber, inkl. Wirkung, Grenzen, Risiken und Praxisempfehlung.

### 📄 Kontext

In der Regel taucht das hier im /etc/fstab **Client-Mount** auf, z. B.:

```bash
server:/praxis /praxis nfs4 \
rsize=1048576,wsize=1048576,_netdev,hard 0 0
```

oder beim manuellen Mount:

```bash
mount -t nfs4 -o rsize=1048576,wsize=1048576 server:/praxis /praxis
```

🔧 rsize=1048576

**🔍Was bedeutet das?**

rsize = **Read Size**

→ Maximale Datenmenge (in Bytes), die der Client **pro NFS-Read-Request** vom Server anfordert.

```bash
rsize=1048576   # = 1 MiB
```

**🧠 Technischer Hintergrund**

- NFS arbeitet mit **RPC-Requests**
- Jeder Read = Request + Response
- Größere rsize ⇒
  - weniger Requests
  - weniger Kontextwechsel
  - weniger Protokoll-Overhead

**🧨 Ohne Tuning (Default)**

Typische Defaults:
- 64 KiB oder 128 KiB
- historisch klein wegen alten Netzwerken

➡️ Bei großen Dateien:
- **viele tausend RPCs**
- unnötige CPU-Last
- geringerer Durchsatz

**🛡 Wirkung mit 1 MiB**

✔ Deutlich höherer Durchsatz
✔ Weniger CPU-Last
✔ Besser für:
  - Scans (PDF, TIFF)
  - Images
  - Datenbanken (lesend)
  - Backups
___

🔧 wsize=1048576
**🔍 Was bedeutet das?**

wsize = **Write Size**
→ Maximale Datenmenge pro **Schreib-Request** zum Server.

```bash
wsize=1048576   # = 1 MiB
```

**🧠 Technischer Hintergrund**

- Große Writes → weniger Roundtrips
- Besonders relevant bei:
  - vielen kleinen Dateien
  - sequenziellen Writes
  - VM-Images
  - Office-Dokumenten

**🛡 Wirkung**

✔ Weniger RPCs
✔ Höhere Schreibrate
✔ Bessere Latenz unter Last
___

### ⚙️ Warum genau 1 MiB (1048576)?
**Technische Gründe**
- Linux Kernel ≥ 5.x unterstützt 1 MiB stabil
- Passt gut zu:
  - Jumbo Frames
  - modernen NICs
  - SSD / NVMe
- Sehr guter Sweet Spot zwischen:
  - Performance
  - Fehleranfälligkeit
  - Latenz

**Historie**
|Wert	Bemerkung
|-----|----------|
|32–64 KiB|	Alt / Legacy|
|128–256 KiB|	Default vieler Distros|
|**1 MiB**|**Best Practice (2024+)**|
|>1 MiB|	Meist kein Gewinn|
___

### ⚠️ Nebenwirkungen & Grenzen
**Netzwerk**

- Schlechte Netzwerke → größere Pakete → mehr Retransmits

WLAN ❌

WAN / VPN ⚠️

**Empfehlung nach Umfeld**

|Umgebung|	Empfehlung|
|--------|------------|          
|Gigabit-LAN|	✅ 1 MiB|
|10 GbE|	✅ 1 MiB<
|WLAN|	❌ kleiner|
|WAN / VPN|	⚠️ testen|

___

### 🔐 Sicherheit & Datenintegrität

❗ Wichtig: rsize / wsize haben **keinen Einfluss auf Sicherheit**
Sie ändern **nur die Chunk-Größe**, nicht:
- Authentifizierung
- Kerberos
- Rechte
- Konsistenz

Kombination mit:
- hard
- sync (Server)
- krb5i / krb5p

ist **voll sicher**.

___

### 🧪 Prüfen & Validieren
**Aktive Werte anzeigen**

```bash
nfsstat -m
```

Beispiel:

```bash
rsize=1048576,wsize=1048576
```
**Benchmark (optional)**

```bash
dd if=/praxis/test.img of=/dev/null bs=1M count=1024
```

### 🏛️ BSI-Einordnung

- **Nicht sicherheitsrelevant**
- **Performance-Optimierung**
- Kein Risiko für Compliance
- Explizit erlaubt im BSI-Grundschutz
___

### 🧠 Kurzfazit
|Aspekt|	Bewertung|
|------|-------------|
|Performance|⭐⭐⭐⭐⭐|
|Stabilität|	⭐⭐⭐⭐|
|Sicherheit|	⭐⭐⭐⭐⭐|
|Praxisnutzen|	⭐⭐⭐⭐⭐|

**👉 Empfohlene Einstellung für produktive NFS-Clients im LAN**

**Client (optional Tuning)**

```bash
rsize=1048576,wsize=1048576
```
___


### 9️⃣ Typische Fehler & Lösungen

|Problem|	Ursache|	Lösung|
|-------|----------|----------|
|permission denied|	UID mismatch|	gleiche UID/GID|
|Hängt beim Boot|	kein netdev|	fstab anpassen|
|Root darf alles|	|no_root_squash|	nicht verwenden|
|Langsam|	kleine rsize|	1M setzen|

___

### 🔐 Sicherheit (Praxis-Minimum)
- NFS nur im internen Netz
- Firewall aktiv
- root_squash immer
- Regelmäßige **Backups auf dem Server**, nicht vom Client

___

✅ Ergebnis

✔ Zentraler, stabiler Fileserver

✔ Saubere Linux-Integration

✔ Minimaler Wartungsaufwand

✔ Praxis- & produktionsgeeignet
___

Hier ist ein **praxistaugliches Setup für NFSv4 + Kerberos unter Linux** – so wie es **in sicheren Umgebungen (Praxis, Behörde, Klinik, Forschung)** eingesetzt wird.

👉 Ziel: **verschlüsselter, authentifizierter Zugriff auf NFS, keine UID-Spoofing-Risiken, kein Root-Zugriff vom Client.**

### 🎯 Was bringt NFS + Kerberos?

|Problem ohne Kerberos|	Lösung mit Kerberos|
|---------------------|--------------------|
|UID/GID manipulierbar|	Starke Authentifizierung|
|Klartext-Zugriff|	Verschlüsselung möglich|
|Root-Spoofing|	kryptografisch verhindert|
|Kein User-Tracking|	Pro Benutzer authentifiziert|

___

### 🧠 Architektur (Minimal & bewährt)

|Rolle	|Dienst|
|-------|-------|
|KDC	|Kerberos Key Distribution Center|
|NFS-Server|	nfs-kernel-server|
|Clients|	|nfs-common + Kerberos|
|Realm|	PRAXIS.LOCAL|
|Security Flavor|	krb5i oder krb5p|

> 💡 Für kleine Umgebungen reicht ein integrierter KDC auf dem NFS-Server

___

### 1️⃣ Kerberos installieren (Server & Clients)

```bash
apt install krb5-kdc krb5-admin-server krb5-user
```

Realm-Abfrage:

```bash
Realm: PRAXIS.LOCAL
KDC: server01.praxis.local
Admin Server: server01.praxis.local
```

### 2️⃣ /etc/krb5.conf **(ALLE Systeme)**

```bash
[libdefaults]
 default_realm = PRAXIS.LOCAL
 dns_lookup_kdc = false
 dns_lookup_realm = false
 ticket_lifetime = 24h
 forwardable = true

[realms]
 PRAXIS.LOCAL = {
  kdc = server01.praxis.local
  admin_server = server01.praxis.local
 }

[domain_realm]
 .praxis.local = PRAXIS.LOCAL
 praxis.local = PRAXIS.LOCAL
```

**⚠️ DNS & Zeit müssen stimmen!**
→ chrony oder systemd-timesyncd
___

### 3️⃣ Kerberos initialisieren (nur Server)

```bash
krb5_newrealm
```

Admin anlegen:

```bash
kadmin.local
addprinc admin/admin
quit
```
___

### 4️⃣ NFS-Service-Principals anlegen

```bash
kadmin.local
addprinc -randkey nfs/server01.praxis.local
ktadd nfs/server01.praxis.local
quit
```
✔️ Keytab landet in /etc/krb5.keytab
___

### 5️⃣ NFS-Server Kerberos-fähig machen
/etc/default/nfs-kernel-server

```bash
RPCNFSDOPTS="--nfs-version 4.2"
```

**Kerberos aktivieren**

```bash
systemctl restart nfs-server rpc-gssd rpc-svcgssd
```
___

### 6️⃣ /etc/exports (mit Kerberos!)

```bash
/srv/nfs  *(ro,fsid=0,sec=krb5i)
/srv/nfs/praxis *(rw,sync,sec=krb5i)
```
**🔐 Security-Optionen**
- krb5 → Authentifiziert
- krb5i → + Integrität (empfohlen)
- krb5p → + Verschlüsselung (langsamer)

Aktivieren:

```bash
exportfs -ra
```
___

### 7️⃣ Client konfigurieren
**Dienste starten**

```bash
systemctl enable rpc-gssd --now
```

**Kerberos-Ticket holen**

```bash
kinit praxis
klist
```
___

### 8️⃣ Mount (Client)

```bash
mount -t nfs4 -o sec=krb5i server01.praxis.local:/praxis /praxis
```

/etc/fstab

```bash
server01.praxis.local:/praxis /praxis nfs4 sec=krb5i,_netdev,hard 0 0
```
___

### 9️⃣ WICHTIGE Praxisregeln
**🔒 Sicherheit**
- root_squash ist implizit
- UID/GID müssen trotzdem identisch sein
- Kerberos ersetzt keine Unix-Rechte

**⏱ Zeit**

```bash
apt install chrony
```

Zeitabweichung > 5 Minuten = **Login kaputt**

### 10️⃣ Typische Fehler & Ursachen

|Fehler|	Ursache|
|-------|----------|
|access denied by server|	kein Ticket (kinit)|
|Permission denied|	UID/GID| mismatch|
|No credentials found|	rpc-gssd läuft nicht|
|Mount hängt|	DNS / Reverse DNS falsch|

🏁 Ergebnis

✔ Kryptografisch gesicherter NFS-Zugriff

✔ Keine IP- oder UID-Fälschung

✔ Praxis- & revisionssicher

✔ Produktionsstandard
