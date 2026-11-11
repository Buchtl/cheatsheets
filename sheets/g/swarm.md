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
