# 🎮 Minecraft Crossplay-Server (Java + Bedrock)

**Einrichtungsanleitung für Proxmox LXC auf Debian 13**

PaperMC 26.1.2 + Geyser + Floodgate | Stand: April 2026

---

## Voraussetzungen

- Proxmox mit einem Debian 13 LXC-Template
- LXC RAM: mindestens 4 GB (empfohlen 6 GB – Java + Welt + Plugins brauchen spürbar mehr als Bedrock)
- LXC Disk: mindestens 8 GB
- LXC CPU: mindestens 2 Kerne (PaperMC profitiert stark von mehreren Kernen)
- „Start at boot" im LXC aktiviert (für automatischen Neustart nach Stromausfall)

> **Info:** Falls du bereits den Bedrock-Server nach der anderen Anleitung eingerichtet hast, kann dieser Crossplay-Server parallel dazu laufen – jeder in einem eigenen LXC mit eigener IP. Am Router brauchst du dann nur verschiedene externe Ports.

> **Info:** Paper 26.1.2 benötigt **Java 25**. Ältere Versionen (Java 21, 17) funktionieren nicht mehr.

---

## Schritt 0 – Uhrzeit & Zeitzone prüfen

Wie beim Bedrock-Server sollte die Systemzeit des LXC überprüft werden, damit Backups zur richtigen Zeit laufen und Log-Einträge stimmen.

### Aktuelle Zeit & Zeitzone anzeigen

```bash
timedatectl
```

### Zeitzone ändern (z.B. auf Deutschland)

```bash
timedatectl set-timezone Europe/Berlin
```

### NTP-Synchronisation prüfen

Falls `System clock synchronized: no` erscheint:

```bash
apt install -y systemd-timesyncd
timedatectl set-ntp true
timedatectl
```

---

## Schritt 1 – System aktualisieren & Abhängigkeiten installieren

Als root im LXC ausführen:

```bash
apt update && apt upgrade -y
apt install -y curl wget unzip screen ufw ca-certificates gnupg
```

---

## Schritt 2 – Java 25 (Temurin/OpenJDK) installieren

Debian 13 hat standardmäßig noch kein Java 25 im Repository. Wir installieren es über das Adoptium-Repository (Eclipse Temurin):

```bash
# Adoptium GPG-Schlüssel hinzufügen
wget -qO- https://packages.adoptium.net/artifactory/api/gpg/key/public | \
  gpg --dearmor -o /usr/share/keyrings/adoptium.gpg

# Repository eintragen
echo "deb [signed-by=/usr/share/keyrings/adoptium.gpg] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | \
  tee /etc/apt/sources.list.d/adoptium.list

# Java 25 installieren
apt update
apt install -y temurin-25-jre
```

Installation prüfen:

```bash
java -version
```

Ausgabe sollte etwa so aussehen:

```
openjdk version "25" 2025-09-16
OpenJDK Runtime Environment Temurin-25+36 (build 25+36)
OpenJDK 64-Bit Server VM Temurin-25+36 (build 25+36, mixed mode)
```

> **Info:** `temurin-25-jre` reicht für den Serverbetrieb. Das volle JDK wird nicht benötigt.

---

## Schritt 3 – Benutzer anlegen

Aus Sicherheitsgründen wird der Server unter einem eigenen Benutzer betrieben:

```bash
useradd -m -r -s /bin/bash minecraft
su - minecraft
```

---

## Schritt 4 – PaperMC-Server herunterladen

PaperMC ist ein optimierter Minecraft-Java-Server mit Plugin-Unterstützung. Die aktuellen Builds sind unter [papermc.io/downloads/paper](https://papermc.io/downloads/paper) zu finden.

```bash
mkdir -p ~/paper && cd ~/paper

# Den aktuellen Paper-Build für 26.1.2 herunterladen
# Die URL ändert sich bei jedem Build – hier die API nutzen um immer den aktuellsten zu bekommen:

LATEST_BUILD=$(curl -s -H "User-Agent: Crossplay-Setup/1.0" \
  https://fill.papermc.io/v3/projects/paper/versions/26.1.2/builds | \
  grep -o '"url":"[^"]*paper-26.1.2[^"]*\.jar"' | head -n 1 | cut -d'"' -f4)

curl -Lo paper.jar "$LATEST_BUILD"
```

Prüfen ob die Datei heruntergeladen wurde:

```bash
ls -lh paper.jar
```

> **Info:** Falls der Download fehlschlägt: Datei manuell auf dem PC von [papermc.io/downloads/paper](https://papermc.io/downloads/paper) herunterladen und per scp übertragen: `scp paper.jar minecraft@<LXC-IP>:~/paper/paper.jar`

---

## Schritt 5 – EULA akzeptieren & Erster Teststart

Beim ersten Start generiert Paper die Dateien und bricht dann ab, weil die EULA noch nicht akzeptiert wurde:

```bash
cd ~/paper
java -Xms2G -Xmx4G -jar paper.jar --nogui
```

Nach dem Abbruch die `eula.txt` bearbeiten:

```bash
nano eula.txt
```

`eula=false` in `eula=true` ändern. Speichern mit `Ctrl+X` → `Y` → `Enter`.

Dann Server erneut starten:

```bash
java -Xms2G -Xmx4G -jar paper.jar --nogui
```

Wenn `Done (XX.XXXs)! For help, type "help"` erscheint läuft alles korrekt. Mit `stop` beenden (nicht Ctrl+C – sonst wird die Welt nicht sauber gespeichert).

> **Info:** `-Xms2G -Xmx4G` bedeutet: Minimum 2 GB, Maximum 4 GB RAM. Diese Werte an die Größe deines LXC anpassen – aber immer mindestens 1–2 GB unter dem LXC-Maximum lassen, damit das System selbst noch Luft hat.

---

## Schritt 6 – Geyser & Floodgate installieren

Diese beiden Plugins sind das Herzstück des Crossplay-Setups:

- **Geyser** übersetzt das Bedrock-Protokoll in das Java-Protokoll
- **Floodgate** erlaubt Bedrock-Spielern die Anmeldung am Online-Mode-Server ohne Java-Account

Als minecraft-User im Plugin-Ordner:

```bash
cd ~/paper/plugins

# Geyser (Spigot-Variante – funktioniert auch auf Paper)
curl -Lo Geyser-Spigot.jar \
  "https://download.geysermc.org/v2/projects/geyser/versions/latest/builds/latest/downloads/spigot"

# Floodgate (Spigot-Variante)
curl -Lo floodgate-spigot.jar \
  "https://download.geysermc.org/v2/projects/floodgate/versions/latest/builds/latest/downloads/spigot"

ls -lh
```

Beide `.jar`-Dateien sollten jetzt im `plugins/`-Ordner liegen.

> **Info:** Geyser und Floodgate müssen zur selben „Generation" passen. Wenn man immer die `latest`-Version von beiden nimmt passen sie zusammen.

---

## Schritt 7 – server.properties anpassen

Den minecraft-Benutzer verlassen (`exit`) und als root die server.properties bearbeiten:

```bash
nano /home/minecraft/paper/server.properties
```

Folgende Einstellungen anpassen:

| Einstellung | Beschreibung |
|---|---|
| `motd=Weber Crossplay` | Name/Beschreibung des Servers in der Serverliste |
| `gamemode=creative` | survival / creative / adventure |
| `difficulty=peaceful` | peaceful / easy / normal / hard |
| `max-players=10` | Maximale Spieleranzahl |
| `online-mode=true` | Xbox/Microsoft-Authentifizierung (WICHTIG für Sicherheit!) |
| `level-name=world` | Weltname (= Ordnername) |
| `server-port=25565` | Java-Port (Standard) |
| `enable-command-block=true` | Command Blocks erlauben |

> **Wichtig:** `online-mode=true` lassen! Floodgate kümmert sich darum dass Bedrock-Spieler trotzdem rein kommen – die werden über ihren Microsoft-Account authentifiziert.

Mit `Ctrl+X` → `Y` → `Enter` speichern.

---

## Schritt 8 – Plugins zum Generieren der Configs einmal starten

Damit Geyser und Floodgate ihre Konfigurationsdateien erstellen, den Server kurz starten und wieder stoppen:

```bash
su - minecraft
cd ~/paper
java -Xms2G -Xmx4G -jar paper.jar --nogui
```

Warten bis `Done` erscheint, dann:

```
stop
```

Zurück als root:

```bash
exit
```

---

## Schritt 9 – Geyser konfigurieren

```bash
nano /home/minecraft/paper/plugins/Geyser-Spigot/config.yml
```

Folgende Einstellungen prüfen/anpassen:

```yaml
bedrock:
  # Port auf dem Bedrock-Clients eingehen – Standard 19132
  port: 19132
  # Name der im Minecraft-Bedrock-Serverbrowser angezeigt wird
  motd1: "Weber Crossplay"
  motd2: "Java + Bedrock"

remote:
  # MUSS floodgate sein damit Bedrock-Spieler ohne Java-Account rein kommen
  auth-type: floodgate
```

Speichern mit `Ctrl+X` → `Y` → `Enter`.

> **Info:** Die `auth-type: floodgate` Zeile ist entscheidend! Ohne sie können Bedrock-Spieler den Server zwar sehen, sich aber nicht einloggen.

---

## Schritt 10 – Floodgate konfigurieren (optional)

Die Standardkonfiguration von Floodgate ist für die meisten Setups in Ordnung. Ein paar interessante Einstellungen:

```bash
nano /home/minecraft/paper/plugins/floodgate/config.yml
```

Relevante Optionen:

```yaml
# Präfix der vor Bedrock-Spielernamen gesetzt wird damit Java-Namen nicht kollidieren
# Standard ist "." – kann man auch leer lassen oder ein anderes Zeichen setzen
username-prefix: "."

# Leerzeichen in Bedrock-Namen durch Unterstrich ersetzen (sonst Fehler bei manchen Plugins)
replace-spaces: true

# Erlaubt Bedrock-Accounts mit einem Java-Account zu verlinken
player-link:
  enabled: true
  allowed: true
```

Speichern mit `Ctrl+X` → `Y` → `Enter`.

> **Info:** Der Präfix `.` wird auch in der Whitelist und bei OP-Rechten verwendet. Ein Bedrock-Spieler „Stefan" erscheint auf dem Server als „.Stefan".

---

## Schritt 11 – systemd-Service einrichten

Als root eine systemd-Service-Datei erstellen:

```bash
nano /etc/systemd/system/minecraft-paper.service
```

Inhalt der Datei:

```ini
[Unit]
Description=Minecraft Paper Server (Crossplay)
After=network.target

[Service]
User=minecraft
WorkingDirectory=/home/minecraft/paper
ExecStart=/usr/bin/java -Xms2G -Xmx4G -XX:+UseG1GC -jar paper.jar --nogui
ExecStop=/bin/bash -c 'echo "stop" > /proc/$MAINPID/fd/0'
Restart=on-failure
RestartSec=10
StandardInput=null

[Install]
WantedBy=multi-user.target
```

> **Info:** Die RAM-Werte (`-Xms2G -Xmx4G`) an die Größe deines LXC anpassen. Faustregel: `-Xmx` = LXC-RAM minus 1 GB für das System.

Service aktivieren und starten:

```bash
systemctl daemon-reload
systemctl enable minecraft-paper
systemctl start minecraft-paper
systemctl status minecraft-paper
```

Logs prüfen:

```bash
journalctl -u minecraft-paper -n 50
```

Wenn folgende Zeilen erscheinen läuft alles:

```
[INFO] Done (X.XXXs)! For help, type "help"
[INFO] [geyser-spigot] Started Geyser on 0.0.0.0:19132
[INFO] [floodgate] Floodgate has been enabled
```

---

## Schritt 12 – Firewall einrichten (ufw)

Für Crossplay werden **zwei Ports** benötigt:

```bash
# SSH zuerst absichern!
ufw allow 22/tcp

# Java-Port (TCP)
ufw allow 25565/tcp

# Bedrock-Port (UDP) für Geyser
ufw allow 19132/udp

# Firewall aktivieren
ufw enable

ufw status
```

---

## Schritt 13 – Port-Forwarding am Router

Damit der Server aus dem Internet erreichbar ist, müssen am Router **zwei Weiterleitungen** eingerichtet werden:

**Weiterleitung 1 – Java:**

| Einstellung | Wert |
|---|---|
| Protokoll | TCP |
| Externer Port | 25565 |
| Interner Port | 25565 |
| Ziel-IP | IP des LXC |

**Weiterleitung 2 – Bedrock:**

| Einstellung | Wert |
|---|---|
| Protokoll | UDP |
| Externer Port | 19132 |
| Interner Port | 19132 |
| Ziel-IP | IP des LXC |

LXC-IP ermitteln:

```bash
ip a | grep inet
```

> **Info:** Falls der Bedrock-Server (aus der anderen Anleitung) auch noch laufen soll, musst du für einen der beiden Server einen anderen externen Port wählen (z.B. extern 19133 → intern 19132 auf dem Crossplay-LXC). Bedrock-Spieler müssen dann beim Verbindungsaufbau den Port mit angeben.

---

## Schritt 14 – Automatisches Backup

Ein tägliches Backup-Script erstellen, das 7 Tage aufbewahrt:

```bash
nano /usr/local/bin/paper-backup.sh
```

Inhalt:

```bash
#!/bin/bash

BACKUP_DIR="/home/minecraft/backups"
KEEP_DAYS=7

mkdir -p "$BACKUP_DIR"
DATE=$(date +%Y-%m-%d_%H-%M)
BACKUP_FILE="$BACKUP_DIR/paper-backup-$DATE.tar.gz"

systemctl stop minecraft-paper
tar -czf "$BACKUP_FILE" -C "/home/minecraft/paper" \
  world world_nether world_the_end \
  server.properties whitelist.json ops.json \
  plugins/Geyser-Spigot/config.yml \
  plugins/floodgate/config.yml
systemctl start minecraft-paper

find "$BACKUP_DIR" -name "paper-backup-*.tar.gz" -mtime +$KEEP_DAYS -delete
echo "Backup erstellt: $BACKUP_FILE"
```

Script ausführbar machen:

```bash
chmod +x /usr/local/bin/paper-backup.sh
```

Cronjob einrichten (03:30 Uhr täglich – eine halbe Stunde nach dem Bedrock-Server falls beide laufen):

```bash
crontab -e
```

Folgende Zeile hinzufügen:

```
30 3 * * * /usr/local/bin/paper-backup.sh >> /var/log/paper-backup.log 2>&1
```

Script manuell testen:

```bash
/usr/local/bin/paper-backup.sh
ls -lh /home/minecraft/backups/
```

---

## ✅ Fertig – Zusammenfassung

- ✅ PaperMC läuft als systemd-Service
- ✅ Geyser übersetzt Bedrock ↔ Java
- ✅ Floodgate erlaubt Bedrock-Spieler-Login mit Microsoft-Account
- ✅ Online-Mode aktiv – sichere Authentifizierung für alle Spieler
- ✅ Startet automatisch beim Booten (LXC + systemd)
- ✅ Firewall mit ufw konfiguriert (Port 25565 TCP + 19132 UDP)
- ✅ Port-Forwarding für Internetzugang eingerichtet
- ✅ Tägliches Backup um 03:30 Uhr, 7 Tage Aufbewahrung
- ✅ Backup-Logs unter `/var/log/paper-backup.log`

### Mit dem Server verbinden

**Java Edition:** Multiplayer → Server hinzufügen → `<IP>:25565`

**Bedrock Edition** (Handy, Konsole, Windows 10/11):
Multiplayer → Server hinzufügen → IP eingeben, Port 19132

LXC-IP ermitteln:

```bash
ip a | grep inet
```

---

## 🔧 Server-Wartung

### Server starten, stoppen & neu starten

| Befehl | Beschreibung |
|---|---|
| `systemctl start minecraft-paper` | Server starten |
| `systemctl stop minecraft-paper` | Server stoppen |
| `systemctl restart minecraft-paper` | Server neu starten |
| `systemctl status minecraft-paper` | Status anzeigen |
| `journalctl -u minecraft-paper -n 50` | Letzte 50 Log-Einträge |
| `journalctl -u minecraft-paper -f` | Live-Logs mitlesen (Ctrl+C zum Beenden) |

### Backup manuell ausführen

```bash
/usr/local/bin/paper-backup.sh
```

Vorhandene Backups anzeigen:

```bash
ls -lh /home/minecraft/backups/
```

### Backup einspielen

```bash
# Server stoppen
systemctl stop minecraft-paper

# Aktuelle Welten sichern (empfohlen!)
cp -r /home/minecraft/paper/world /home/minecraft/world_vor_wiederherstellung

# Backup einspielen (Dateiname anpassen!)
tar -xzf /home/minecraft/backups/paper-backup-2026-04-17_03-30.tar.gz \
  -C /home/minecraft/paper

# Server wieder starten
systemctl start minecraft-paper
```

### Operator-Rechte (OP) vergeben

**Für Java-Spieler** – einfach per Konsole oder Befehl:

Option 1 – über systemd-Log mittels in-game Command (wenn du selbst schon OP bist):

```
/op DeinJavaName
```

Option 2 – direkt in der Datei `ops.json`:

```bash
systemctl stop minecraft-paper
nano /home/minecraft/paper/ops.json
```

```json
[
  {
    "uuid": "UUID-des-Spielers",
    "name": "DeinJavaName",
    "level": 4,
    "bypassesPlayerLimit": false
  }
]
```

```bash
systemctl start minecraft-paper
```

**Für Bedrock-Spieler** – den Präfix nicht vergessen!

Wenn dein Floodgate-Präfix `.` ist (Standard), musst du den Bedrock-Spieler mit Präfix opsen:

```
/op .DeinBedrockName
```

> **Info:** Befehle im Minecraft-Chat werden **MIT** `/` eingegeben (anders als bei Bedrock Dedicated Server!). In der Server-Konsole (journalctl hilft dir da nicht, dafür brauchst du eine interaktive Eingabe via screen oder tmux) werden Befehle ohne `/` eingegeben.

**Alternativer Weg mit interaktiver Konsole:**

```bash
# Service stoppen
systemctl stop minecraft-paper

# Als minecraft-User manuell starten
su - minecraft
cd ~/paper
java -Xms2G -Xmx4G -jar paper.jar --nogui
```

Nach `Done` direkt eintippen:

```
op DeinName
```

Dann sauber beenden:

```
stop
exit
```

```bash
systemctl start minecraft-paper
```

### Whitelist aktivieren

Wenn nur bestimmte Spieler auf den Server sollen:

```bash
nano /home/minecraft/paper/server.properties
```

```
white-list=true
enforce-whitelist=true
```

Server neu starten und Spieler hinzufügen:

```bash
# Für Java-Spieler
systemctl restart minecraft-paper
# Dann in-game als OP:
/whitelist add JavaName

# Für Bedrock-Spieler (Präfix beachten!):
/whitelist add .BedrockName
```

Alternativ direkt in `whitelist.json` eintragen.

### Plugin hinzufügen

PaperMC unterstützt die meisten Spigot/Bukkit-Plugins. Beliebte Quellen:

- [hangar.papermc.io](https://hangar.papermc.io) – offizielles Paper-Plugin-Portal
- [modrinth.com/plugins](https://modrinth.com/plugins)
- [spigotmc.org/resources](https://www.spigotmc.org/resources/)

Installation:

```bash
systemctl stop minecraft-paper
su - minecraft
cd ~/paper/plugins
# Plugin .jar herunterladen
curl -Lo NeuesPlugin.jar "https://download-url..."
exit
systemctl start minecraft-paper
```

Nach dem Start Logs prüfen ob das Plugin sauber lädt:

```bash
journalctl -u minecraft-paper -n 100 | grep -i neuesplugin
```

### Geyser & Floodgate aktualisieren

Beide Plugins haben in-game Befehle zum Check:

```
/geyser version
/floodgate version
```

Update-Vorgang:

```bash
systemctl stop minecraft-paper
su - minecraft
cd ~/paper/plugins

# Alte Versionen durch neue ersetzen
curl -Lo Geyser-Spigot.jar \
  "https://download.geysermc.org/v2/projects/geyser/versions/latest/builds/latest/downloads/spigot"
curl -Lo floodgate-spigot.jar \
  "https://download.geysermc.org/v2/projects/floodgate/versions/latest/builds/latest/downloads/spigot"

exit
systemctl start minecraft-paper
```

> **Info:** Die Config-Dateien unter `plugins/Geyser-Spigot/` und `plugins/floodgate/` bleiben beim Austausch des Plugin-Jars erhalten.

### Neue Welt erstellen (alte Welt behalten)

Weltnamen in der `server.properties` ändern:

```bash
nano /home/minecraft/paper/server.properties
```

```
level-name=Welt2026-04
```

```bash
systemctl restart minecraft-paper
```

Der Server erstellt automatisch eine neue Welt. Die alte liegt weiterhin als `world/` daneben.

### Neue Welt erstellen (alte Welt löschen)

```bash
# Server stoppen
systemctl stop minecraft-paper

# Backup machen falls noch nicht geschehen!
/usr/local/bin/paper-backup.sh

# Alle drei Dimensionen löschen
rm -rf /home/minecraft/paper/world
rm -rf /home/minecraft/paper/world_nether
rm -rf /home/minecraft/paper/world_the_end

# Wenn neuer Seed gewünscht, level-seed= leeren
nano /home/minecraft/paper/server.properties
# level-seed=

# Server starten – neue Welt wird generiert
systemctl start minecraft-paper
```

> **Vorsicht:** Unwiderruflich! Immer vorher Backup machen.

### Paper-Server-Update (neue Minecraft-Version)

Welten und Plugins bleiben erhalten. Nur die `paper.jar` wird ersetzt.

**Schritt 1** – Aktuelle Version auf [papermc.io/downloads/paper](https://papermc.io/downloads/paper) nachschauen.

**Schritt 2** – Backup und Stop:

```bash
/usr/local/bin/paper-backup.sh
systemctl stop minecraft-paper
```

**Schritt 3** – Neue Version herunterladen:

```bash
su - minecraft
cd ~/paper

# Alte jar zur Sicherheit umbenennen
mv paper.jar paper.jar.bak

# Neue Version holen (Versionsnummer in der URL anpassen!)
LATEST_BUILD=$(curl -s -H "User-Agent: Crossplay-Setup/1.0" \
  https://fill.papermc.io/v3/projects/paper/versions/NEUE_VERSION/builds | \
  grep -o '"url":"[^"]*paper-NEUE_VERSION[^"]*\.jar"' | head -n 1 | cut -d'"' -f4)

curl -Lo paper.jar "$LATEST_BUILD"
exit
```

**Schritt 4** – Kompatibilität prüfen! Bei größeren Versionssprüngen (z.B. 26.1 → 26.2):

- Geyser & Floodgate updaten (siehe oben)
- Andere Plugins auf Kompatibilität prüfen
- Evtl. neue Java-Version nötig (Paper-Release-Notes lesen!)

**Schritt 5** – Server starten und Logs prüfen:

```bash
systemctl start minecraft-paper
journalctl -u minecraft-paper -n 50
```

In den Logs sollte die neue Version erscheinen und alle Plugins sauber laden.

### Typische Probleme

**Bedrock-Spieler findet den Server nicht:**
- Firewall: Ist UDP 19132 offen? (`ufw status`)
- Router: Ist UDP 19132 weitergeleitet?
- Geyser-Log: Läuft Geyser? (`journalctl -u minecraft-paper | grep -i geyser`)

**Bedrock-Spieler kann sich nicht einloggen („Authentifizierung fehlgeschlagen"):**
- `auth-type: floodgate` in der Geyser-config.yml prüfen
- `online-mode=true` in server.properties
- Floodgate-Plugin tatsächlich im `plugins/` Ordner?

**„UnsupportedClassVersionError":**
- Java-Version zu alt. Paper 26.1 braucht Java 25.
- Prüfen mit `java -version`.

**Server startet nicht, „Address already in use":**
- Läuft evtl. noch ein anderer Java-Prozess: `pkill -f paper.jar`
- Dann neu starten.
