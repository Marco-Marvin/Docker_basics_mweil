# Docker Basics (ct3003)

Diese README basiert auf dem c't 3003 YouTube-Video **"So einfach ist Docker"** und enthält alle im Video erwähnten Befehle sowie die zugehörigen Docker-Compose-Dateien für Pi-hole, Portainer und Watchtower. Ergänzt wurde eine eigene Compose-Datei für einen nginx-Webserver.

## Hier alle im Video erwähnten Befehle

### Docker installieren

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh ./get-docker.sh
```

Falls kein `curl` installiert ist: `apt install curl`

Wenn man nicht mit dem Nutzer `root` arbeitet, sollte man den aktuellen Benutzer berechtigen:

```bash
sudo usermod -aG docker $USER
```

### Mit Docker arbeiten

* Laufende Container auflisten: `docker ps`
* Alle Container auflisten (auch gestoppte): `docker ps -a`
* Einen Container anhalten: `docker stop <Containername>` (den Namen findet man mit `docker ps` heraus)
* Einen gestoppten Container endgültig löschen: `docker rm <Containername>`

### Einen simplen Webserver starten

Der Container aus dem Image nginx fährt mit folgendem Befehl hoch:

```bash
docker run -p 80:80 nginx
```

Die eigene IP-Adresse erhält man mit `ip a`.

### Arbeiten mit Docker-Compose

Legt euch am besten einen eigenen Ordner für das Docker-Projekt an, um Ordnung zu halten. Die Datei `docker-compose.yml` (bzw. hier je Ordner `*.yml`) enthält die Definition der Container.

Bearbeitet wird die Datei mit:

```bash
nano docker-compose.yml
```

Den Texteditor Nano beendet man mit: `Strg+X`, dann `Y`.

Die Compose-Zusammenstellung hochfahren:

```bash
docker compose -f <datei>.yml up -d
```

Will man die Container updaten, lädt man die neuen Images mit:

```bash
docker compose -f <datei>.yml pull
```

### Eine oder mehrere Docker-Compose-Dateien?

Das ist definitiv Geschmackssache und hängt von der Umgebung ab. Wenn man mehr als ein Projekt (zum Beispiel einen Blog und ein Pihole) auf einem Server betreibt, sollte man für jedes einen Ordner anlegen und darin eine Docker-Compose-Datei ablegen. Die nützlichen Helfer wie Portainer und Watchtower kommen zusammen in eine weitere Datei. Dann kann man mit `docker compose down` gezielt Teile der Umgebung herunterfahren.

---

## Ordnerstruktur dieses Repos

```
Docker_basics_mweil/
|-Readme.md
|-pihole/
  |-pihole.yml
|-portainer/
  |-portainer.yml
|-watchtower/
  |-watchtower.yml
|-nginx/
  |-nginx.yml
```

---

## Erläuterung der Serveranwendungen

### 🕳️ Pi-hole (`pihole/pihole.yml`)

[Pi-hole](https://pi-hole.net/) ist ein netzwerkweiter Werbe- und Tracker-Blocker, der als DNS-Server fungiert. Statt Werbung erst im Browser zu blockieren, filtert Pi-hole bereits die DNS-Anfragen: Anfragen an bekannte Werbe- und Tracking-Domains werden gar nicht erst aufgelöst, sodass alle Geräte im Netzwerk (PC, Smartphone, Smart-TV, …) automatisch werbefrei sind, sobald sie Pi-hole als DNS-Server verwenden.

* **Ports:** 53/tcp+udp (DNS), 67/udp (DHCP, optional), 80/tcp (Web-Oberfläche)
* **Volumes:** `etc-pihole` und `etc-dnsmasq.d` speichern Konfiguration und Blocklisten persistent außerhalb des Containers
* **Web-Login:** `http://<Server-IP>` bzw. `http://<Server-IP>/admin` – das Passwort wird beim ersten Start zufällig generiert und steht im Container-Log (`docker logs pihole`), sofern nicht über `WEBPASSWORD` selbst gesetzt
* **Zweck im Setup:** zeigt, wie ein Container mit eigenen Ports, Capabilities (`NET_ADMIN`) und persistenten Volumes betrieben wird

### 🐳 Portainer (`portainer/portainer.yml`)

[Portainer](https://www.portainer.io/) ist eine grafische Weboberfläche zur Verwaltung von Docker-Umgebungen. Statt alle Container, Images, Volumes und Netzwerke über die Kommandozeile zu verwalten, bietet Portainer ein übersichtliches Dashboard, über das man Container starten/stoppen, Logs einsehen, Compose-Dateien deployen und Ressourcen überwachen kann.

* **Port:** 9000 (Web-Oberfläche)
* **Volume:** `/var/run/docker.sock` wird in den Container gemountet, damit Portainer mit dem Docker-Daemon des Hosts kommunizieren kann; `portainer_data` speichert die eigene Konfiguration persistent
* **Web-Login:** `http://<Server-IP>:9000` – beim ersten Aufruf wird ein Admin-Benutzer mit Passwort angelegt
* **Zweck im Setup:** dient als zentrales "Helfer"-Tool, um alle anderen laufenden Container (Pi-hole, Watchtower, nginx) grafisch zu überwachen und zu steuern – als Ergänzung zu `docker ps` und Docker Desktop

### 🔄 Watchtower (`watchtower/watchtower.yml`)

[Watchtower](https://containrrr.dev/watchtower/) überwacht laufende Docker-Container und aktualisiert sie automatisch, sobald ein neues Image auf Docker Hub verfügbar ist. Der veraltete Container wird gestoppt, das neue Image gezogen und ein neuer Container mit identischer Konfiguration gestartet – ganz ohne manuelles Eingreifen.

* **Kein eigener Port / keine Web-Oberfläche:** Watchtower läuft im Hintergrund und wird nur über Logs (`docker logs watchtower`) beobachtet
* **Volume:** `/var/run/docker.sock`, damit Watchtower Zugriff auf alle anderen Container des Hosts hat und diese neu starten kann
* **Zweck im Setup:** hält Pi-hole, Portainer und nginx automatisch aktuell, ohne dass man selbst regelmäßig `docker compose pull` ausführen muss

### 🌐 nginx (`nginx/nginx.yml`)

[nginx](https://nginx.org/) ist ein schneller, leichtgewichtiger Webserver, der im Video als einfachstes Beispiel für einen Docker-Container gezeigt wird (`docker run -p 80:80 nginx`). Hier wurde daraus eine eigene Compose-Datei erstellt, um das Prinzip Ports/Volumes/Restart-Policy auch für den einfachsten Anwendungsfall zu üben.

* **Port:** 8080 auf dem Host, gemappt auf Port 80 im Container (8080 statt 80, da Port 80 bereits von Pi-hole belegt ist)
* **Volume:** `./html` wird read-only als Webroot (`/usr/share/nginx/html`) eingebunden, sodass eigene HTML-Dateien ohne Neubau des Images angezeigt werden können
* **Web-Login:** `http://<Server-IP>:8080` zeigt die eigene `index.html`
* **Zweck im Setup:** einfachstes Beispiel eines Webservers, der über Docker Compose statt über den `docker run`-Befehl aus dem Video gestartet wird

---

## Vorgehen beim Starten der Container

```bash
git clone <repo-url>
cd Docker_basics_mweil

docker compose -f pihole/pihole.yml up -d
docker ps

docker compose -f portainer/portainer.yml up -d
docker ps

docker compose -f watchtower/watchtower.yml up -d
docker ps

docker compose -f nginx/nginx.yml up -d
docker ps
```

Anschließend wurden die Web-Oberflächen von Pi-hole (`:80`/`admin`), Portainer (`:9000`) und nginx (`:8080`) im Browser aufgerufen und die laufenden Container zusätzlich über **Docker Desktop** gestartet/gestoppt und beobachtet.
