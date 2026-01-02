### 📘 Einführung in NFSv4

**NFSv4 (Network File System Version 4)** ist die moderne Weiterentwicklung des Network File Systems und stellt ein **zustandsbehaftetes, sicheres und leistungsfähiges Netzwerk-Dateisystem** dar.

Im Gegensatz zu älteren NFS-Versionen wurde NFSv4 **grundlegend neu konzipiert,** um heutigen Anforderungen an Sicherheit, Skalierbarkeit und Internet-Tauglichkeit gerecht zu werden.
___


### 🎯 Ziel von NFSv4

NFSv4 verfolgt insbesondere folgende Ziele:

- einheitliches, standardisiertes Protokoll
- verbesserte Sicherheit durch integrierte Authentifizierung
- reduzierter Verwaltungsaufwand
- bessere Unterstützung verteilter und globaler Umgebungen
- konsistentes Locking und Zustandsverwaltung

NFSv4 ist heute der **empfohlene Standard** für neue NFS-Installationen.
___

### 🧱 Zentrale Eigenschaften von NFSv4

#### Zustandsbehaftetes Protokoll

NFSv4 verwaltet Server-Zustände (Sessions, Locks, Delegationen).
Dadurch werden **konsistente Datei- und Sperrmechanismen** ermöglicht.

#### Einheitlicher Port

- Kommunikation erfolgt standardmäßig über TCP Port 2049
- Kein separater Portmapper notwendig
- Firewall- und WAN-freundlich

#### Integrierte Sicherheit

- Unterstützung von RPCSEC_GSS
- Nutzung von Kerberos möglich
- Authentifizierung, Integrität und Verschlüsselung auf Protokollebene

#### Verbesserte Locking-Mechanismen

- Datei- und Record-Locks sind Teil des Protokolls
- Kein separates Lock-Daemon-Konzept mehr
___

### 🌐 NFSv4 im Vergleich zu älteren Versionen

| Merkmal |	NFSv3 |	NFSv4 |
|---------|:-----:|:-----:|
| Zustandsbehaftet      | 	❌ |	✅ |
| Sicherheit integriert |	❌ |	✅ |
| Fester Port           |	❌ |	✅ |
| Locking integriert    |	❌ |	✅ |
| WAN-tauglich	        | eingeschränkt |	✅ |


### 🧠 Merksätze

> NFSv4 ist mehr als ein Update – es ist ein neues Protokolldesign.
>
> Sicherheit und Locking sind fester Bestandteil von NFSv4.
>
> NFSv4 ist für moderne, verteilte Netzwerke ausgelegt.

---
⬅ [Zur Übersicht](../README.md)
➡ [Weiter: Grundlagen von NFSv4](../ch02-grundlagen/README.md)

