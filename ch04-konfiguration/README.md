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


