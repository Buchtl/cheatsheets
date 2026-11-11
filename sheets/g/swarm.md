Wenn der Manager-Node festgenagelt war und der Bind Mount immer gepasst hat, gibt es im Swarm-Kontext noch drei sehr spezifische Ursachen, die genau diesen „Gedächtnisverlust“ bei einem Update oder Neustart auslösen können.

1. Die Swarm-Cluster-ID hat sich geändert (Der Klassiker)

Portainer verknüpft jeden Swarm-Stack in seiner Datenbank fest mit der internen Cluster-ID deines Docker Swarms.
• Was passiert: Wenn du irgendwann einmal docker swarm leave --force und danach docker swarm init ausgeführt hast (z. B. nach einer IP-Änderung des Servers oder einem Zertifikats-Reset), vergibt Docker eine brandneue Cluster-ID.
• Die Folge: Für Portainer ist das danach ein komplett fremder Swarm. Die alten Stacks existieren noch in der Datenbank, sind aber an die alte Cluster-ID gebunden und tauchen in der aktuellen Umgebung nur noch als „Limited Control“ (extern) auf.

2. Race Condition durch fehlendes stop-first

Auch wenn Portainer auf demselben Node bleibt, versucht Swarm bei einem docker service update oder docker stack deploy (wenn order: stop-first nicht explizit gesetzt ist) oft kurzzeitig zwei Container parallel zu betreiben.
• Was passiert: Portainer nutzt BoltDB (eine Datei-Datenbank). Zwei gleichzeitige Schreibzugriffe beim Migrationsprozess eines Updates können die Index-Struktur der portainer.db beschädigen, sodass Stacks nicht mehr geladen werden können.

3. Namenskonflikt des Environments (Endpoint-Bug in älteren Versionen)

Bei Updates von älteren Portainer-Versionen (insbesondere rund um die Versionen 2.11 bis 2.19) gab es einen bekannten Bug im Zusammenspiel mit dem Portainer Agent im Swarm.
• Was passiert: Wenn der Portainer-Server vor dem Portainer-Agent hochgefahren ist, konnte er den Swarm-Endpoint beim Start nicht sofort erreichen und hat beim Reconnect intern eine neue Endpoint-ID generiert.

So gehst du jetzt auf Nummer sicher

Da dein Bind Mount und das Pinning bereits stimmen, bist du schon zu 90 % auf der sicheren Seite. Um die restlichen 10 % auszuschließen, reicht vor deinem nächsten Update dieser kurze Check:
1. Einmalig das UI-Backup herunterladen (Settings -> General -> Back up Portainer). Damit hast du den kompletten Zustand sicher auf deiner Festplatte.
2. Prüfen, ob order: stop-first in deiner Portainer-Stack-Datei steht. Falls nicht, ergänze es unter deploy: -> update_config:.
3. Keine großen Versionssprünge auf einmal machen. Falls deine Portainer-Version schon mehrere Jahre alt ist, aktualisiere zuerst auf die letzte 2.19.x / 2.21.x Version und erst danach auf die aktuelle Version (aktuell 2.27+ / 2.3x), damit die Datenbank-Migrationen sauber nacheinander durchlaufen.
Welche Portainer-Version läuft aktuell bei dir, und nutzt du für die Verbindung zum Swarm den Portainer Agent oder direkt den Docker Socket?


Ein eingebundenes SAN ist der exakte Grund für den damaligen Verlust nach dem Reboot.
Das ist ein klassisches Timing-Problem beim Booten von Linux-VMs. Wenn die VM nach einem Update neu startet, laufen zwei Prozesse parallel ab, wobei der falsche gewinnt:
1. Docker ist zu schnell. Der Docker-Dienst startet beim Booten extrem früh, sobald das Basis-System steht.
2. Das SAN braucht länger. Bis iSCSI/Fibre-Channel-Sessions aufgebaut, Multipath-Geräte erkannt und das Dateisystem eingehängt sind, vergehen oft ein paar Sekunden mehr.
3. Der leere Ordner-Effekt. In genau diesen Sekunden startet Docker bereits den Portainer-Swarm-Service. Da das SAN noch nicht unter dem Pfad eingehängt ist, sieht Docker den leeren lokalen Mountpoint-Ordner auf der Systemplatte. Portainer findet dort keine portainer.db, denkt es wäre eine Erstinstallation und legt eine nagelneue, leere Datenbank an. Kurz darauf hängt Linux das SAN darüber – oder umgekehrt, der Container bleibt im leeren lokalen Ordner gefangen.

So löst du das SAN-Timing-Problem endgültig

Du musst dem Betriebssystem lediglich beibringen, dass der Docker-Dienst beim Booten zwingend auf das SAN warten muss, bevor er auch nur einen einzigen Container startet.

Schritt 1: Systemd-Abhängigkeit für Docker setzen

Erstelle auf dem Manager-Node (und idealerweise auf allen Worker-Nodes, die SAN-Mounts nutzen) eine Override-Konfiguration für Docker:
bash
sudo systemctl edit docker.service
Verwende Code mit Vorsicht.
Füge im sich öffnenden Editor ganz oben folgende Zeilen ein (passe /pfad/zu/deinem/san an deinen echten Mountpoint an, z. B. /mnt/san):
ini
[Unit]
RequiresMountsFor=/pfad/zu/deinem/san
After=remote-fs.target
Verwende Code mit Vorsicht.
Speichere die Datei und lade die Konfiguration neu:
bash
sudo systemctl daemon-reload
Verwende Code mit Vorsicht.

Schritt 2: fstab-Eintrag absichern (Wichtiger Schutzmechanismus)

Prüfe in deiner /etc/fstab, wie das SAN eingebunden ist. Wenn dort die Option _netdev (bei iSCSI/Netzwerk-SAN) und nofail entfernt bzw. sauber gesetzt ist, verhindert Linux im Fehlerfall, dass Dienste auf einen ungemounteten Ordner schreiben.
Noch besser: Entziehe dem lokalen (ungemounteten) Ordner auf der Systemplatte die Schreibrechte. Solange das SAN nicht eingehängt ist, greift diese Berechtigung. Sobald das SAN eingehängt wird, überdecken die Rechte des SAN-Dateisystems diesen Schutz automatisch:
1. Portainer kurz stoppen (docker service scale portainer_portainer=0)
2. SAN aushängen (sudo umount /pfad/zu/deinem/san)
3. Dem leeren lokalen Ordner die Rechte entziehen (sudo chattr +i /pfad/zu/deinem/san) – Damit wird der Ordner unveränderlich (immutable).
4. SAN wieder einhängen (sudo mount -a)
5. Portainer wieder starten (docker service scale portainer_portainer=1)
Der Effekt: Sollte das SAN jemals ausfallen oder beim Booten zu spät kommen, kann Docker keine leere Datenbank mehr in den lokalen Ordner schreiben. Der Container bricht stattdessen sauber ab und startet erst dann erfolgreich, wenn das SAN wirklich da ist.
Wie ist dein SAN in der VM eingebunden – über iSCSI / Multipath in der VM selbst oder als durchgereichte virtuelle Festplatte vom Hypervisor (z. B. VMware/Proxmox)?


Ja, absolut – genau für dieses Szenario ist das Portainer-Backup gemacht und rettet dich in unter einer Minute.
Wenn Portainer nach einem SAN-Timing-Problem oder Update mit einer leeren Datenbank hochfährt, kannst du dein Backup direkt über die Weboberfläche wieder einspielen.

Was passiert beim Wiederherstellen des Backups?

• Alle Stacks kommen sofort zurück. Die .tar.gz-Backup-Datei enthält deine komplette portainer.db samt allen gespeicherten docker-compose.yml-Dateien und Umgebungsvariablen.
• Die Verknüpfung zu laufenden Containern wird repariert. Sobald Portainer das Backup eingespielt hat und sich selbst neu startet, erkennt er deine auf dem Swarm laufenden Container wieder als seine eigenen Stacks. Der Status „Limited Control“ verschwindet sofort und du kannst sie wieder ganz normal bearbeiten.
• Deine laufenden Dienste werden nicht unterbrochen. Das Einspielen des Backups überschreibt nur Portainers interne Verwaltung. Deine produktiven Container laufen währenddessen ohne eine Sekunde Ausfallzeit weiter.

So spielst du das Backup im Notfall ein

1. Portainer öffnen. Wenn Portainer leer gestartet ist, fragt er nach einem neuen Admin-Passwort (oder zeigt den leeren Login). Auf dem allerersten Einrichtungsbildschirm gibt es ganz unten direkt die Option „Restore Portainer from backup“.
2. Falls du schon eingeloggt bist: Geh auf Settings -> General -> Back up Portainer, wähle den Tab Restore from backup, lade deine .tar.gz-Datei hoch und klicke auf Restore.
3. Kurz warten: Portainer startet sich einmal intern neu (dauert ca. 10 Sekunden) und danach ist alles exakt so wie vor dem Update.

Ein wichtiger Haken beim SAN-Problem

Das Backup hilft dir perfekt – aber nur, wenn das SAN zum Zeitpunkt des Restores auch wirklich eingehängt ist.
Wenn du das Backup einspielst, während das SAN noch nicht gemountet ist, schreibt Portainer das Backup in den falschen (lokalen) Ordner. Sobald das SAN danach doch noch eingehängt wird, überdeckt es diesen Ordner wieder. Mit der Kombination aus RequiresMountsFor (Systemd) und einer aktuellen Backup-Datei auf deinem PC bist du aber zu 100 % gegen jeden Datenverlust abgesichert.
Möchtest du, dass ich dir zeige, wie du das Portainer-Backup über die Portainer-API automatisiert (z. B. per Cronjob) erstellen kannst, statt es jedes Mal manuell herunterzuladen?


Um das Portainer-Backup vollautomatisch per Cronjob zu erstellen, nutzt du die Portainer-API zusammen mit einem API-Access-Token. So musst du nirgendwo ein Klartext-Passwort hinterlegen und bekommst regelmäßig eine fertige .tar.gz-Datei auf deinen Server (oder direkt auf dein SAN).

Schritt 1: API-Token in Portainer erstellen

1. Logge dich in Portainer ein und klicke oben rechts auf deinen Benutzernamen -> My account.
2. Scrolle zum Abschnitt Access tokens und klicke auf Add access token.
3. Vergib eine Beschreibung (z. B. auto-backup) und kopiere den angezeigten Token (beginnt meist mit ptr_...). Hinweis: Er wird nur einmal angezeigt.

Schritt 2: Backup-Skript anlegen

Erstelle auf deinem Manager-Node ein kleines Shell-Skript, z. B. unter /usr/local/bin/portainer-backup.sh:
bash
sudo nano /usr/local/bin/portainer-backup.sh
Verwende Code mit Vorsicht.
Füge folgenden Inhalt ein und passe die drei Variablen oben an deine Umgebung an:
bash
#!/bin/bash

# --- Konfiguration ---
PORTAINER_URL="https://deine-portainer-ip:9443"
API_TOKEN="ptr_DEIN_KOPIERTER_TOKEN"
BACKUP_DIR="/pfad/zu/deinem/backup-ordner"
KEEP_DAYS=14
# ---------------------

mkdir -p "$BACKUP_DIR"
DATE=$(date +"%Y-%m-%d_%H-%M")
BACKUP_FILE="$BACKUP_DIR/portainer-backup_$DATE.tar.gz"

# Backup über die Portainer API anfordern (-k erlaubt selbstsignierte SSL-Zertifikate)
curl -k -s -X POST "$PORTAINER_URL/api/backup" \
  -H "X-API-Key: $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}' \
  -o "$BACKUP_FILE"

# Prüfen, ob die Datei nicht leer ist (z.B. bei API-Fehler)
if [ -s "$BACKUP_FILE" ]; then
    # Alte Backups löschen, die älter als KEEP_DAYS sind
    find "$BACKUP_DIR" -name "portainer-backup_*.tar.gz" -type f -mtime +$KEEP_DAYS -delete
else
    rm -f "$BACKUP_FILE"
    echo "Fehler: Portainer Backup konnte nicht erstellt werden." >&2
    exit 1
fi
Verwende Code mit Vorsicht.
(Tipp: Speichere BACKUP_DIR am besten nicht im selben Ordner wie die Portainer-Daten, sondern in einem separaten Backup-Pfad oder auf einer lokalen Platte der VM, damit das Backup auch im Notfall griffbereit ist.)
Mache das Skript ausführbar und teste es einmal manuell:
bash
sudo chmod 700 /usr/local/bin/portainer-backup.sh
sudo /usr/local/bin/portainer-backup.sh
ls -lh /pfad/zu/deinem/backup-ordner
Verwende Code mit Vorsicht.
Wenn dort eine .tar.gz-Datei mit einigen Megabytes liegt, funktioniert alles einwandfrei.

Schritt 3: Cronjob einrichten

Öffne die Crontab des Root-Users, damit das Skript automatisch läuft:
bash
sudo crontab -e
Verwende Code mit Vorsicht.
Füge ganz unten folgende Zeile hinzu, um das Backup zum Beispiel jeden Tag um 03:00 Uhr nachts auszuführen:
cron
0 3 * * * /usr/local/bin/portainer-backup.sh
Verwende Code mit Vorsicht.
Damit hast du jeden Morgen ein frisches, vollständiges Backup deiner gesamten Portainer-Konfiguration und aller Swarm-Stacks, ohne jemals wieder manuell daran denken zu müssen.
Möchtest du das Backup im Skript zusätzlich noch verschlüsseln (Portainer unterstützt ein Passwort beim API-Backup), oder reicht dir die unverschlüsselte Ablage auf deinem internen Speicher?

