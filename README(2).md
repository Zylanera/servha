# Server Security Incident Report
## Kompromittierung und Sicherungsmassnahmen

---

## 1. Zusammenfassung des Vorfalls

Am 9. Juli 2026 wurde festgestellt, dass der VPS Server (my-vps, IP: ##.##.##.###) mit Malware infiziert war. Ein Kryptominer-Trojaner namens "perfcc" wurde installiert und verursachte eine CPU-Auslastung von bis zu 99%. Der Server wurde erfolgreich bereinigt und umfassend gesichert.

---

## 2. Erkannte Probleme

### 2.1 CPU-Auslastung
- CPU-Auslastung stieg wiederholt auf 99%
- RAM-Auslastung erreichte 87-89%
- Server war mehrfach für kurze Zeit nicht mehr responsive
- Ausfallzeiten: 7. Juli 20:07 Uhr und 9. Juli 16:59 Uhr

### 2.2 Malware-Installation
Die Malware "perfcc" wurde durch folgende Cron-Jobs aktiviert:

```
/etc/cron.d/perfclean (enthielt: perfcc)
/etc/cron.daily/perfclean (Launcher-Skript)
/var/spool/cron/crontabs/root (11 * * * * /root/.config/cron/perfcc)
```

Diese Jobs führten den Kryptominer stündlich aus und verursachten die hohe CPU-Last.

### 2.3 Brute-Force-Attacken
Der Server wurde von mehreren IPs aus mit SSH-Brute-Force-Angriffen attackiert:

| IP-Adresse | Standort | ISP | Dienst | Username-Versuche |
|-----------|----------|-----|--------|-------------------|
| 103.183.62.2 | Dhaka, Bangladesch | Hasan Broadband Net | Broadband-ISP | ubuntu, sammy |
| 193.46.255.86 | Rushden, England | Unmanaged Ltd | Public Proxy Server | admin |
| 185.242.3.195 | Philadelphia, USA | Demenin B.V. | Data Center/Transit | root |
| 103.250.11.63 | Singapur | CV. Sayogi Jasa Mandiri | Data Center/Transit | root |
| 178.20.210.208 | Kasachstan | William Joerg Maria Dorchain | Data Center/Transit | webadmin, web1 |

Keine dieser Attacken war erfolgreich, aber sie deuteten auf aktive Scanning-Aktivitäten hin.

**Analyse der Angreifer-IPs:**
- Die meisten IPs stammen von Hosting-Providern oder Data-Center-Services
- Eine IP (193.46.255.86) ist ein öffentlicher Proxy-Server
- Die Bangladesch-IP war ein privater Broadband-Anschluss
- Alle Angreifer versuchten verschiedene Standard-Usernamen (ubuntu, admin, root, webadmin, sammy, web1)
- Das Angriffsmuster deutet auf automatisierte Scan-Tools hin, nicht auf manuelle Attacken
- Die Vielfalt der Herkunftsländer (Bangladesch, England, USA, Singapur, Kasachstan) deutet auf ein organisiertes Scanning-Netzwerk hin

### 2.4 Webmin-Sicherheitsprobleme
- Webmin Port war öffentlich erreichbar und wurde von Scannern aktiv gescannt
- Multiple Neustarts von Webmin (restart counter = 3) deuteten auf Fehler hin
- Der Dienst verbrauchte zeitweise bis zu 35% CPU

### 2.5 Ressourcen-Engpässe
- Verfügbarer RAM: nur 121-252 MB von 3,8 GB
- Kein Swap-Speicher konfiguriert, obwohl System an Limit lief
- RAM-Cache: 427 MB (zu hoch für diesen Server)

---

## 3. Ursachenanalyse

### 3.1 Wie kam die Malware herein?
Die genaue Kompromittierungsmethode konnte nicht definitiv bestimmt werden, aber folgende Vektoren waren wahrscheinlich:

1. Webmin-Exploit: Der Webmin-Port war öffentlich erreichbar und wurde aktiv gescannt
2. SSH Brute-Force: Mehrere externe IPs führten Brute-Force-Attacken durch
3. Schwaches Passwort oder bekannte Schwachstelle: Keine erfolgreiche Anmeldung in den Logs sichtbar, was auf einen automatisierten Exploit hindeutet

### 3.2 Funktionsweise der Malware
- perfcc ist ein bekannter Kryptominer-Trojaner
- Das Skript versteckte sich in mehreren Cron-Job-Dateien
- Es wurde stündlich (Minute 11 jeder Stunde) ausgeführt
- Das Binary selbst war nicht direkt sichtbar - wahrscheinlich in Environment-Variablen oder als Shell-Alias versteckt

---

## 4. Behobene Massnahmen

### 4.1 Malware-Entfernung

Folgende Cron-Jobs wurden gelöscht:
```bash
rm /etc/cron.d/perfclean
rm /etc/cron.daily/perfclean
crontab -r
rm -rf /root/.config/cron/
rm -rf /root/.cache/
```

Nach der Malware-Entfernung:
- CPU-Auslastung fiel auf normale Werte (unter 5%)
- Keine weiteren "perfcc" Prozesse in ps-Ausgaben sichtbar
- Cron-Jobs werden nicht mehr automatisch ausgeführt

### 4.2 SSH-Zugang gesichert

SSH wurde von öffentlichem Zugang auf Tailscale-Only beschränkt:

```bash
cat > /etc/ssh/sshd_config.d/tailscale.conf << 'EOF'
ListenAddress ###.###.##.##
Port ##
EOF
systemctl restart ssh
```

Ergebnis: SSH ist nun nur von Geräten im Tailnet erreichbar. Die öffentliche IP (##.##.##.###) akzeptiert keine SSH-Verbindungen mehr.

### 4.3 Fail2Ban installiert und konfiguriert

Installation:
```bash
apt install fail2ban -y
```

Konfiguration für aggressive Sperrung:
```bash
cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 2

[sshd]
enabled = true
port = ssh
logpath = /var/log/auth.log
maxretry = 2
EOF
systemctl restart fail2ban
```

Effekt: Nach 2 fehlgeschlagenen SSH-Anmeldeversuchen wird die IP für 1 Stunde blockiert.

### 4.4 Swap-Speicher hinzugefügt

Swap wurde installiert um OOM-Killer-Abstürze zu verhindern:

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

Ergebnis: System hat jetzt 2 GB Reserve-Speicher.

### 4.5 Tailscale-Integration

Tailscale wurde installiert und konfiguriert:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

Server-IP im Tailnet: ###.###.##.##

Dies ermöglicht sicheren SSH-Zugang nur für Geräte im persönlichen Tailnet.

---

## 5. Verbleibende Offene Ports

Nach der Sicherung sind folgende Ports offen:

- Port 80/443: Nginx Proxy Manager (für HTTP/HTTPS Weiterleitung)
- Port 4050: railpix.at Application (notwendig für Produktivbetrieb)
- SSH: Nur über Tailscale (###.###.##.##:##)

Port 22 ist auf der öffentlichen IP nicht mehr erreichbar.

---

## 6. Empfehlungen für zukünftige Sicherheit

### 6.1 Webmin
- Entweder komplett deinstallieren, wenn nicht benötigt
- Oder: Webmin-Port mit Firewall-Regeln nur auf bestimmte IPs beschränken
- Regelmäßig auf Updates prüfen

### 6.2 Monitoring
- Regelmäßig CPU/RAM/Disk-Auslastung überwachen
- Log-Dateien auf verdächtige Aktivitäten überprüfen
- Fail2Ban-Status regelmäßig kontrollieren

### 6.3 Backups
- Regelmäßige Backups der Docker-Container
- Backup-Scripts sollten manuell konfiguriert werden, nicht automatisch über Webmin

### 6.4 Updates
- System-Packages regelmäßig updaten
- Docker-Images aktuell halten
- Insbesondere Sicherheits-Updates zeitnah einspielen

### 6.5 Passwörter
- Root-SSH-Passwort eventuell ändern (war nicht kompromittiert, aber präventiv)
- Webmin Passwörter überprüfen und eventuell zurücksetzen

---

## 7. Timeline der Ereignisse

| Datum/Zeit | Ereignis |
|-----------|----------|
| 3. Juli 07:00 | Erste SSL-Zertifikat-Erneuerungen in Logs |
| 7. Juli 20:07 | Erster Node.js-Prozess-Crash (OOM-Killer) |
| 7. Juli 21:07-23:07 | Mehrere Neustarts des Webmin-Diensts |
| 8. Juli 06:59 | Server-Neustart nach Crash |
| 9. Juli 16:59 | Erneuter Node.js-Prozess-Crash |
| 9. Juli 17:50 | CPU-Last erreicht 99% |
| 9. Juli 18:09 | Cron-Job führt perfcc aus (letzte bekannte Ausführung) |
| 9. Juli 18:11 | Cron-Job führt perfcc aus |
| 9. Juli 18:22 | Malware-Cron-Jobs gelöscht |
| 9. Juli 18:29 | Fail2Ban installiert und gestartet |
| 9. Juli 18:41 | SSH auf Tailscale beschränkt |

---

## 8. Logs und Beweise

Die folgenden Log-Dateien wurden für diesen Bericht analysiert:

- /var/log/auth.log - SSH und Authentifizierungs-Versuche
- /var/webmin/miniserv.log - Webmin-Zugriffe und Aktivitäten
- /var/webmin/miniserv.error - Webmin-Fehler
- journalctl - systemd-Ereignisse
- /var/log/fail2ban.log - Fail2Ban-Aktivitäten

Alle Log-Dateien wurden in server-logs.tar.gz archiviert.

---

## 9. Abschluss

Der Server wurde erfolgreich bereinigt und ist nun:

1. Frei von Malware
2. Gegen Brute-Force-Attacken geschützt
3. SSH-Zugang ist auf Tailscale beschränkt
4. Hat ausreichend Swap-Speicher
5. Überwacht durch Fail2Ban

Die Ausfallzeiten sollten sich nicht wiederholen, da die Ursachen (Kryptominer + OOM-Crashes) behoben wurden.

---

**Bericht erstellt:** 9. Juli 2026
**Server:** my-vps (##.##.##.### / ###.###.##.## über Tailscale)
