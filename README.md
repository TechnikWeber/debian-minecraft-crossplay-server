# 🎮 Minecraft Crossplay-Server (Java + Bedrock)

**Einrichtungsanleitung für Proxmox LXC auf Debian 13**

PaperMC 1.21.11 + Geyser + Floodgate + ViaVersion | Stand: April 2026

---

## Voraussetzungen

- Proxmox mit einem Debian 13 LXC-Template
- LXC RAM: mindestens 4 GB (empfohlen 6 GB)
- LXC Disk: mindestens 8 GB
- LXC CPU: mindestens 2 Kerne
- „Start at boot" im LXC aktiviert

> **Wichtig zur Versionswahl:** Wir nutzen Paper **1.21.11** (= 26.1.1 unter dem neuen Mojang-Versionsschema), nicht die allerneueste 26.1.2. Grund: Geyser ist mit 26.1.2 (Stand April 2026) noch nicht kompatibel und stürzt mit `ExceptionInInitializerError` ab. Sobald Geyser nachzieht, kann man auf 26.1.2 hochziehen. Das ist der normale Crossplay-Server-Workflow: **Geyser ist immer der limitierende Faktor**, Paper-Updates folgen erst nach Geyser-Updates.

> **Info:** Java-Clients in einer neueren Version (z.B. 26.1.2) können sich trotzdem verbinden – dafür installieren wir am Schluss ViaVersion + ViaBackwards.

---

## Schritt 0 – Uhrzeit & Zeitzone prüfen

```bash
timedatectl
timedatectl set-timezone Europe/Berlin
```

Falls NTP nicht synchronisiert:

```bash
apt install -y systemd-timesyncd
timedatectl set-ntp true
```

---

## Schritt 1 – System aktualisieren & Abhängigkeiten installieren

Als root im LXC:

```bash
apt update && apt upgrade -y
apt install -y curl wget unzip screen ufw ca-certificates gnupg
```

---

## Schritt 2 – Java 25 installieren

Paper 1.21.11 benötigt **Java 25**. Debian 13 hat das nicht im Standard-Repository, daher über das Adoptium-Repository (Eclipse Temurin):

```bash
wget -qO- https://packages.adoptium.net/artifactory/api/gpg/key/public | \
  gpg --dearmor -o /usr/share/keyrings/adoptium.gpg

echo "deb [signed-by=/usr/share/keyrings/adoptium.gpg] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | \
  tee /etc/apt/sources.list.d/adoptium.list

apt update
apt install -y temurin-25-jre

java -version
```

Ausgabe sollte etwa `openjdk version "25"` zeigen.

---

## Schritt 3 – Benutzer anlegen

```bash
useradd -m -r -s /bin/bash minecraft
```

> **Info:** Der `-r`-Flag macht das einen System-User ohne Passwort. Du kannst dich nicht direkt als minecraft einloggen – Wechsel geht nur **von root aus** mit `su - minecraft`. Wenn du schon als minecraft eingeloggt bist und nochmal `su - minecraft` versuchst, kommt „Authentication failure" – das ist normal.

---

## Schritt 4 – PaperMC 1.21.11 herunterladen

Direkt vom PaperMC-CDN als root auf dem LXC:

```bash
mkdir -p /home/minecraft/paper
cd /home/minecraft/paper

wget -O paper.jar \
  "https://fill-data.papermc.io/v1/objects/25eb85bd8415195ce4bc188e1939e0c7cef77fb51d26d4e766407ee922561097/paper-1.21.11-130.jar"

chown -R minecraft:minecraft /home/minecraft/paper
ls -lh paper.jar
```

Die Datei sollte ca. 50–60 MB groß sein.

> **Info:** Diese URL zeigt auf Build 130 von Paper 1.21.11 und ist stabil. Falls du eine neuere Version brauchst: auf [papermc.io/downloads/paper](https://papermc.io/downloads/paper) im Versions-Dropdown **1.21.11** wählen, Rechtsklick auf den Download-Button → "Link-Adresse kopieren".

---

## Schritt 5 – EULA akzeptieren & Erster Teststart

Als root in den minecraft-User wechseln und Server starten:

```bash
su - minecraft
cd ~/paper
java -Xms2G -Xmx4G -jar paper.jar --nogui
```

Beim ersten Start wird die `eula.txt` erstellt und der Server stoppt mit einem Hinweis. Falls der Server nicht von alleine zurück zum Prompt kommt: `Ctrl+C` drücken.

EULA bearbeiten:

```bash
nano eula.txt
```

`eula=false` → `eula=true`. Speichern mit `Ctrl+X` → `Y` → `Enter`.

Erneut starten:

```bash
java -Xms2G -Xmx4G -jar paper.jar --nogui
```

Bei `Done (XX.XXXs)! For help, type "help"` läuft alles. In der jetzt erscheinenden Server-Konsole `>` eingeben:

```
stop
```

---

## Schritt 6 – Geyser & Floodgate installieren

Noch als minecraft-User:

```bash
cd ~/paper/plugins

curl -Lo Geyser-Spigot.jar \
  "https://download.geysermc.org/v2/projects/geyser/versions/latest/builds/latest/downloads/spigot"

curl -Lo floodgate-spigot.jar \
  "https://download.geysermc.org/v2/projects/floodgate/versions/latest/builds/latest/downloads/spigot"

ls -lh
```

Beide Dateien sollten ein paar MB groß sein. Dann zurück zu root:

```bash
exit
```

---

## Schritt 7 – ViaVersion & ViaBackwards installieren

Damit Java-Spieler mit neueren Client-Versionen (z.B. 26.1.2) trotzdem auf den 1.21.11-Server können:

```bash
su - minecraft
cd ~/paper/plugins

# ViaVersion 5.9.0 Release vom Hangar-CDN (offizielle Quelle)
wget -O ViaVersion.jar \
  "https://hangarcdn.papermc.io/plugins/ViaVersion/ViaVersion/versions/5.9.0/PAPER/ViaVersion-5.9.0.jar"

# ViaBackwards 5.9.0 Release
wget -O ViaBackwards.jar \
  "https://hangarcdn.papermc.io/plugins/ViaVersion/ViaBackwards/versions/5.9.0/PAPER/ViaBackwards-5.9.0.jar"

# Prüfen
ls -lh Via*.jar
file Via*.jar

exit
```

Beide Dateien sollten **5–10 MB groß** sein und `Java archive data (JAR)` als Typ haben.

> **Wichtig:** Direkter Hangar-CDN-Link mit fester Versionsnummer ist die einzig zuverlässige Methode. Die GitHub-`latest/download`-URLs liefern für ViaVersion eine 9-Byte-Fehlerseite statt der JAR. Auch die Hangar-API-URL ist nicht stabil.

> **Info:** Beide Plugins **immer aus demselben Channel** (Release oder Snapshot) ziehen, nie mischen. Release ist die richtige Wahl für den normalen Betrieb.

> **Update-Workflow:** Bei neuen Versionen einfach `5.9.0` in den URLs durch die aktuelle Release-Version ersetzen. Aktuelle Versionsnummer auf [hangar.papermc.io/ViaVersion/ViaVersion/versions](https://hangar.papermc.io/ViaVersion/ViaVersion/versions) checken.

---

## Schritt 8 – server.properties anpassen

Als root:

```bash
nano /home/minecraft/paper/server.properties
```

Wichtige Einstellungen:

| Einstellung | Wert |
|---|---|
| `motd=SternpunktMC` | Servername in der Java-Serverliste |
| `gamemode=survival` | survival / creative / adventure |
| `difficulty=easy` | peaceful / easy / normal / hard |
| `max-players=10` | Maximale Spieleranzahl |
| `online-mode=true` | **Wichtig: true lassen!** |
| `level-name=world` | Weltname |
| `server-port=25565` | Java-Port (Standard) |
| `enforce-secure-profile=false` | **Wichtig für Crossplay!** Sonst werden Bedrock-Spieler beim Chatten gekickt |

Speichern mit `Ctrl+X` → `Y` → `Enter`.

> **Wichtig zu `enforce-secure-profile`:** Diese Einstellung verlangt von Java-Clients kryptographisch signierte Chat-Nachrichten von Mojangs Servern. Bedrock-Clients (über Floodgate) können das technisch nicht liefern, weil sie über Microsoft/Xbox authentifiziert werden, nicht über Mojangs Java-Auth. Wenn die Einstellung auf `true` steht, werden Bedrock-Spieler beim ersten Chatversuch mit dem Fehler **„Der Chat wurde aufgrund eines fehlenden öffentlichen Profilschlüssels deaktiviert"** rausgeworfen. Auf privaten Crossplay-Servern ist `false` der Standard-Workaround und offizielle Empfehlung von Floodgate.

---

## Schritt 9 – Plugins-Configs generieren lassen

Server einmal kurz starten damit Geyser/Floodgate/ViaVersion ihre Config-Dateien erstellen:

```bash
su - minecraft
cd ~/paper
java -Xms2G -Xmx4G -jar paper.jar --nogui
```

Warten bis `Done` erscheint. In den Logs sollte stehen:

```
[INFO] [floodgate] Took XXXms to boot Floodgate
[INFO] [Geyser-Spigot] Started Geyser on UDP port 19132
[INFO] [ViaVersion] Loading ViaVersion v5.9.0
[INFO] [ViaBackwards] Loading ViaBackwards v5.9.0
[INFO] Done (XX.XXXs)! For help, type "help"
```

In der Server-Konsole:

```
stop
```

Zurück zu root:

```bash
exit
```

> **Wichtig:** Wenn Geyser hier mit `ExceptionInInitializerError` und `Couldn't find createItemStack method` abstürzt, ist die Paper-Version zu neu für Geyser. Dann ist Paper auf 1.21.11 fixiert bis Geyser nachzieht.

> **Hinweis bei Locale-Download:** Geyser zeigt beim ersten Start `Missing MC locale file: en_us` und lädt sich Übersetzungen aus dem Vanilla-JAR. Das ist normal und passiert nur einmal.

---

## Schritt 10 – Geyser konfigurieren

Die Geyser-Config-Struktur hat sich geändert – nicht mehr `remote:` sondern `java:`:

```bash
nano /home/minecraft/paper/plugins/Geyser-Spigot/config.yml
```

**Im `bedrock:`-Block** den `motd:`-Unterblock anpassen (das was Bedrock-Spieler in der Serverliste sehen):

```yaml
bedrock:
  motd:
    primary-motd: SternpunktMC
    secondary-motd: Java + Bedrock Crossplay
```

**Im `gameplay:`-Block** den Servernamen setzen (was Bedrock-Spieler ingame im Pause-Menü sehen):

```yaml
gameplay:
  server-name: SternpunktMC
```

**Im `java:`-Block** – das ist der wichtigste Punkt – `auth-type` auf `floodgate` ändern:

```yaml
java:
  auth-type: floodgate
```

Speichern mit `Ctrl+X` → `Y` → `Enter`.

> **Info:** Du hast jetzt drei MOTD/Namen-Felder die alle den gleichen Server beschreiben:
> - `motd=` in server.properties → Java-Spieler in der Serverliste
> - `bedrock.motd.primary-motd` in Geyser config.yml → Bedrock-Spieler in der Serverliste
> - `gameplay.server-name` in Geyser config.yml → Bedrock-Spieler ingame
>
> Am saubersten überall den gleichen Namen setzen.

---

## Schritt 11 – Floodgate Account-Linking aktivieren (optional, empfohlen)

Wenn du sowohl mit Java- als auch Bedrock-Client mit deinem Microsoft-Account spielen willst, ist Account-Linking sehr praktisch – sonst musst du dich zweimal opsen (einmal als `MaxPower3309`, einmal als `.MaxPower3309`).

```bash
nano /home/minecraft/paper/plugins/floodgate/config.yml
```

Im `player-link:`-Block:

```yaml
player-link:
  enabled: true
  require-link: false
  allowed: true
```

Speichern. Verlinkung im Spiel geht später so:

1. Mit Java-Client einloggen
2. Mit Bedrock-Client einloggen (erstmal als `.MaxPower3309`)
3. Im Java-Chat: `/linkaccount` → 4-stelliger Code wird angezeigt
4. Im Bedrock-Chat: `/linkaccount MaxPower3309 <code>`

Nach dem nächsten Bedrock-Login bist du als `MaxPower3309` (ohne Punkt) drin – mit allen Rechten und Inventar deines Java-Accounts.

---

## Schritt 12 – systemd-Service einrichten

```bash
nano /etc/systemd/system/minecraft-paper.service
```

Inhalt:

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

Aktivieren und starten:

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

Erfolg sieht so aus:

```
[INFO] Done (X.XXXs)! For help, type "help"
[INFO] [Geyser-Spigot] Started Geyser on UDP port 19132
[INFO] [floodgate] Floodgate has been enabled
[INFO] [ViaVersion] Loading ViaVersion v5.9.0
```

---

## Schritt 13 – Firewall einrichten

```bash
ufw allow 22/tcp        # SSH
ufw allow 25565/tcp     # Java
ufw allow 19132/udp     # Bedrock via Geyser
ufw enable
ufw status
```

---

## Schritt 14 – Port-Forwarding am Router

| Protokoll | Externer Port | Interner Port | Ziel |
|---|---|---|---|
| TCP | 25565 | 25565 | LXC-IP |
| UDP | 19132 | 19132 | LXC-IP |

LXC-IP ermitteln:

```bash
ip a | grep inet
```

---

## Schritt 15 – Backup einrichten

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
  plugins/floodgate/config.yml 2>/dev/null
systemctl start minecraft-paper

find "$BACKUP_DIR" -name "paper-backup-*.tar.gz" -mtime +$KEEP_DAYS -delete
echo "Backup erstellt: $BACKUP_FILE"
```

```bash
chmod +x /usr/local/bin/paper-backup.sh
crontab -e
```

Hinzufügen:

```
30 3 * * * /usr/local/bin/paper-backup.sh >> /var/log/paper-backup.log 2>&1
```

Test:

```bash
/usr/local/bin/paper-backup.sh
ls -lh /home/minecraft/backups/
```

---

## ✅ Fertig

**Verbindung:**
- **Java:** `<LXC-IP>` (Port 25565 ist Standard, muss nicht angegeben werden)
- **Bedrock:** IP eingeben, Port 19132 (im Bedrock-Dialog meist schon vorausgefüllt)

**Spielbar mit:**
- Java-Clients in fast jeder Version (1.21.11 nativ, andere via ViaVersion/ViaBackwards)
- Bedrock-Clients (Handy, Konsole, Win10/11) via Geyser

---

## 🔧 Wartung – die wichtigsten Befehle

```bash
systemctl start minecraft-paper      # starten
systemctl stop minecraft-paper       # stoppen
systemctl restart minecraft-paper    # neu starten
systemctl status minecraft-paper     # Status
journalctl -u minecraft-paper -n 50  # letzte 50 Logs
journalctl -u minecraft-paper -f     # Live-Logs (Ctrl+C zum Beenden)
/usr/local/bin/paper-backup.sh       # manuelles Backup
```

## OP-Rechte vergeben

**Wichtig zum Verständnis:** Java- und Bedrock-Account sind aus Server-Sicht **zwei verschiedene Spieler**, auch wenn beide unter demselben Microsoft-Account hängen. Du musst entweder beide einzeln opsen oder Account-Linking nutzen (siehe Schritt 11).

**Mit Account-Linking** (einfacher Weg): Nach dem Verlinken nur den Java-Namen opsen.

**Ohne Account-Linking:** Beide separat opsen, der Bedrock-Spieler braucht den Floodgate-Präfix:

```
op MaxPower3309
op .MaxPower3309
```

Beide Versionen müssen den Server **mindestens einmal betreten haben** bevor die OP-Befehle funktionieren.

**Wenn du selbst noch kein OP bist** – über die interaktive Konsole:

```bash
systemctl stop minecraft-paper
su - minecraft
cd ~/paper
java -Xms2G -Xmx4G -jar paper.jar --nogui
```

Nach `Done` direkt eintippen (ohne `/`):

```
op DeinName
stop
```

```bash
exit
systemctl start minecraft-paper
```

## Plugins aktualisieren

**Geyser & Floodgate** – die `latest`-URLs holen automatisch die neueste Version:

```bash
systemctl stop minecraft-paper
su - minecraft
cd ~/paper/plugins

curl -Lo Geyser-Spigot.jar \
  "https://download.geysermc.org/v2/projects/geyser/versions/latest/builds/latest/downloads/spigot"
curl -Lo floodgate-spigot.jar \
  "https://download.geysermc.org/v2/projects/floodgate/versions/latest/builds/latest/downloads/spigot"

exit
systemctl start minecraft-paper
journalctl -u minecraft-paper -n 30
```

**ViaVersion & ViaBackwards** – versionsspezifische URL anpassen. Aktuelle Version auf [hangar.papermc.io/ViaVersion/ViaVersion/versions](https://hangar.papermc.io/ViaVersion/ViaVersion/versions) prüfen, dann:

```bash
systemctl stop minecraft-paper
su - minecraft
cd ~/paper/plugins

# Versionsnummer in beiden URLs anpassen!
wget -O ViaVersion.jar \
  "https://hangarcdn.papermc.io/plugins/ViaVersion/ViaVersion/versions/5.9.0/PAPER/ViaVersion-5.9.0.jar"
wget -O ViaBackwards.jar \
  "https://hangarcdn.papermc.io/plugins/ViaVersion/ViaBackwards/versions/5.9.0/PAPER/ViaBackwards-5.9.0.jar"

ls -lh Via*.jar
file Via*.jar

exit
systemctl start minecraft-paper
```

Beide Dateien müssen weiterhin mehrere MB groß sein und als `Java archive data (JAR)` erkannt werden – sonst ist der Download fehlgeschlagen.

## Paper auf neue Version updaten

**Wichtige Reihenfolge:**

1. **Backup machen**
2. **Geyser/Floodgate updaten**
3. Geyser kurz starten und prüfen ob es mit der angestrebten Paper-Version kompatibel ist (entweder testen oder im Geyser-Discord/GitHub schauen)
4. **Erst dann** Paper hochziehen

```bash
/usr/local/bin/paper-backup.sh
systemctl stop minecraft-paper

su - minecraft
cd ~/paper
mv paper.jar paper.jar.alt

# Aktuellen Download-Link von papermc.io/downloads/paper holen
wget -O paper.jar "URL_HIER_EINSETZEN"

exit
systemctl start minecraft-paper
journalctl -u minecraft-paper -n 50
```

Wenn der Server sauber startet → `paper.jar.alt` kann gelöscht werden. Falls Probleme: Server stoppen, `mv paper.jar.alt paper.jar`, wieder starten.

## Typische Probleme

**Bedrock-Spieler findet den Server nicht:**
- Firewall: `ufw status` – ist UDP 19132 offen?
- Router: ist UDP 19132 weitergeleitet?
- Geyser läuft? `journalctl -u minecraft-paper | grep -i geyser`

**Geyser stürzt mit `ExceptionInInitializerError` und `Couldn't find createItemStack method` ab:**
- Paper-Version ist neuer als Geyser unterstützt
- Lösung: Paper eine Minor-Version zurück bis Geyser nachzieht
- Bei Update: immer Geyser zuerst, dann Paper

**Java-Client meldet „outdated server" oder „outdated client":**
- Versions-Mismatch zwischen Client und Server
- Mit ViaVersion/ViaBackwards installiert sollte das nicht passieren – wenn doch: Logs prüfen ob die Plugins korrekt geladen wurden
- `file plugins/Via*.jar` – beide müssen `Java archive data (JAR)` sein, nicht `ASCII text` (= kaputter Download)

**Plugin-Datei ist nur ein paar Bytes groß / `ZipException: zip END header not found`:**
- Download ist fehlgeschlagen (HTML-Fehlerseite statt JAR)
- Datei löschen, mit `wget` von der Hangar-CDN-URL erneut holen (siehe Schritt 7)

**Bedrock-Spieler kann nicht einloggen („Authentifizierung fehlgeschlagen"):**
- `auth-type: floodgate` im `java:`-Block der Geyser-config.yml prüfen
- `online-mode=true` in server.properties
- Floodgate-Plugin tatsächlich im `plugins/` Ordner?

**Bedrock-Spieler wird beim Chatten gekickt („Der Chat wurde aufgrund eines fehlenden öffentlichen Profilschlüssels deaktiviert"):**
- In `server.properties` muss `enforce-secure-profile=false` gesetzt sein
- Server neu starten: `systemctl restart minecraft-paper`
- Hintergrund: Java verlangt sonst kryptographisch signierte Chat-Profile, die Bedrock-Clients technisch nicht haben können

**Server startet nicht, „Address already in use":**
- Noch ein Java-Prozess läuft: `pkill -f paper.jar`, dann neu starten

**`su - minecraft` fragt nach Passwort und scheitert:**
- Du bist schon als minecraft eingeloggt (Prompt zeigt `minecraft@...`). Einfach direkt weitermachen, kein erneutes `su` nötig.
- Oder du bist als anderer User außer root – dann erst `exit` zu root, dann `su - minecraft`.
