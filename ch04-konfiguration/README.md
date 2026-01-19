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

**⚠️ Wichtig:**
Alle Systeme müssen **identische UID/GID** für Benutzer haben
(z. B. User praxis = UID 1001 überall)
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


