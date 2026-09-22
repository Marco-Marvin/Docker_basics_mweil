# Netzwerke mit Docker Compose

Grundlage: [Networking in Docker Compose](https://docs.docker.com/compose/how-tos/networking/)

## 1. Grundlagen

### Standardnetzwerk von Docker Compose

Wenn in der Compose-Datei kein eigenes Netzwerk angegeben ist, erstellt Docker Compose automatisch **ein einziges Standardnetzwerk** vom Typ `bridge` für das ganze Projekt. Der Name setzt sich zusammen aus dem **Projektnamen** (standardmäßig der Ordnername, in dem die Compose-Datei liegt) und dem Zusatz `default`, z. B. `docker_basics_mweil_default`. Alle Services, die in derselben Compose-Datei stehen und dort kein anderes Netzwerk zugewiesen bekommen, werden automatisch in dieses Standardnetzwerk gehängt.

### Kommunikation über Servicenamen

Innerhalb des Standardnetzwerks (und jedes anderen von Compose erstellten Netzwerks) besitzt jeder Container einen **DNS-Eintrag, der genau dem Servicenamen aus der Compose-Datei entspricht**. Ein Container kann also z. B. einfach `http://pihole` statt einer IP-Adresse ansprechen, um den Pi-hole-Container zu erreichen.

Der Servicename sollte statt der IP-Adresse verwendet werden, weil:

- Docker den Containern beim Start **dynamisch** IP-Adressen aus dem internen Subnetz vergibt,
- sich diese IP-Adresse bei jedem Neustart des Containers (oder des ganzen Stacks) **ändern kann**,
- der Servicename dagegen **stabil bleibt**, solange sich die Compose-Datei nicht ändert.

Wer stattdessen die IP fest im Code oder in einer Konfigurationsdatei einträgt, riskiert, dass die Verbindung nach dem nächsten `docker compose up` nicht mehr funktioniert.

### Benutzerdefinierte Netzwerke

Ein eigenes Netzwerk wird auf oberster Ebene der Compose-Datei unter dem Schlüssel `networks:` definiert, z. B.:

```yaml
networks:
  lab_net:
    driver: bridge
```

Damit ein Service diesem Netzwerk zugeordnet wird, trägt man beim jeweiligen Service ebenfalls einen `networks:`-Block mit dem Netzwerknamen ein:

```yaml
services:
  pihole:
    image: pihole/pihole:latest
    networks:
      - lab_net
```

Ein Service kann dabei auch mehreren Netzwerken gleichzeitig zugeordnet werden, wenn er z. B. sowohl mit einer Datenbank als auch mit einem Frontend-Netzwerk kommunizieren soll.

### `network_mode`

Über `network_mode` kann bei einem Service festgelegt werden, wie er ins Netzwerk eingebunden wird. Mögliche Werte sind u. a.:

- `bridge` – Standardverhalten, Container bekommt eine eigene virtuelle Netzwerkschnittstelle im Bridge-Netz,
- `host` – der Container teilt sich den Netzwerk-Stack direkt mit dem Host,
- `none` – der Container bekommt **keine** Netzwerkschnittstelle (komplett isoliert),
- `service:<name>` bzw. `container:<name>` – der Container nutzt den Netzwerk-Stack eines anderen Containers.

`host` eignet sich, wenn ein Container z. B. sehr viele Ports braucht, eine möglichst geringe Netzwerk-Latenz wichtig ist oder ein Dienst (etwa ein DHCP-Server) auf Protokolle angewiesen ist, die mit klassischem Port-Mapping schlecht funktionieren. Einschränkungen:

- Port-Mappings (`ports:`) werden im `host`-Modus **ignoriert**, der Container belegt die Ports direkt auf dem Host.
- Der Modus funktioniert **nur unter Linux** zuverlässig; unter Docker Desktop (Windows/Mac) läuft der Docker-Daemon in einer VM, sodass `host` dort nicht das native Verhalten liefert.
- Es kann zu **Portkonflikten** mit anderen Diensten auf dem Host kommen, da keine Isolation mehr stattfindet.
- Mehrere Container im `host`-Modus können sich nicht mehr sauber über Servicenamen im selben eigenen Netzwerk finden, da sie kein eigenes Compose-Netzwerk mehr nutzen.

### `docker compose stop` vs. `docker compose down`

| | `docker compose stop` | `docker compose down` |
|---|---|---|
| Container | werden **angehalten**, bleiben aber vorhanden | werden **gestoppt und entfernt** |
| Netzwerke | bleiben bestehen | von Compose erstellte Netzwerke werden **entfernt** |
| Volumes | bleiben bestehen | bleiben standardmäßig bestehen (nur mit `-v` werden auch Volumes entfernt) |
| Erneuter Start | `docker compose start` reaktiviert die vorhandenen Container | `docker compose up` **erstellt Container neu** |

Kurz gesagt: `stop` ist ein reines Anhalten (wie ein Pause-Knopf), `down` räumt den kompletten Stack inklusive der von Compose verwalteten Netzwerke wieder auf.

Ein Netzwerk, das in der Compose-Datei als `external: true` markiert ist, wurde **nicht von Compose erstellt**, sondern existiert bereits unabhängig davon (z. B. manuell mit `docker network create` angelegt). Solche externen Netzwerke werden von `docker compose down` **nicht entfernt**, weil Compose sie auch nicht selbst erzeugt hat – Compose verwaltet nur, was es selbst angelegt hat.

### Kommunikation zwischen verschiedenen Compose-Projekten

Standardmäßig bekommt jedes Compose-Projekt sein **eigenes, isoliertes** Standardnetzwerk – Container aus unterschiedlichen Projekten können sich also zunächst **nicht** über ihren Servicenamen erreichen. Damit Container aus verschiedenen Projekten trotzdem kommunizieren können, gibt es zwei gängige Wege:

1. Man legt ein Netzwerk **manuell** an (`docker network create shared_net`) und bindet es in beiden Compose-Dateien jeweils als `external: true` ein.
2. Man deklariert in einem Projekt ein Netzwerk, das im anderen Projekt ebenfalls als `external` referenziert wird.

In beiden Fällen hängen dann Container aus unterschiedlichen Projekten im selben Docker-Netzwerk und können sich über ihre Servicenamen (bzw. Netzwerk-Aliase) erreichen.

### Netzwerk-Aliase

Ein Netzwerk-Alias ist ein **zusätzlicher Hostname**, unter dem ein Container innerhalb eines bestimmten Netzwerks erreichbar ist – zusätzlich zu seinem Servicenamen. Das ist z. B. nützlich, wenn mehrere Compose-Projekte denselben Dienst unter einem einheitlichen, projektunabhängigen Namen ansprechen sollen. Definiert wird ein Alias beim jeweiligen Service innerhalb des Netzwerk-Blocks:

```yaml
services:
  pihole:
    networks:
      lab_net:
        aliases:
          - dns.local
```

Andere Container im selben Netzwerk können den Dienst dann sowohl über `pihole` als auch über `dns.local` erreichen.

### Dynamische und statische IP-Adressen

Standardmäßig vergibt Docker die IP-Adressen der Container **dynamisch** (per integriertem DHCP-ähnlichen Mechanismus) aus dem Subnetz des jeweiligen Netzwerks. Diese Adressen können sich bei einem Neustart ändern.

Möchte man stattdessen feste IP-Adressen nutzen, konfiguriert man beim Netzwerk einen `ipam`-Block (IP Address Management) mit einem `subnet`, und weist dem Service dann über `ipv4_address` eine konkrete Adresse aus diesem Subnetz zu:

```yaml
networks:
  lab_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24

services:
  pihole:
    networks:
      lab_net:
        ipv4_address: 172.20.0.10
```

Für die vorliegende Aufgabe werden bewusst **keine** statischen IP-Adressen verwendet, da die Container ohnehin zuverlässig über ihre Servicenamen erreichbar sind.

### Host-Port vs. Container-Port

Bei einer Portangabe wie `"8080:80"` gilt: `HOST_PORT:CONTAINER_PORT`.

- Der **Container-Port** ist der Port, auf dem der Dienst **innerhalb** des Containers lauscht (z. B. Port 80 bei nginx).
- Der **Host-Port** ist der Port, unter dem dieser Dienst von **außerhalb** – also vom Host-System bzw. aus dem restlichen Netzwerk – erreichbar gemacht wird.

Für die Kommunikation **zwischen Containern im selben Docker-Netzwerk** wird immer der **Container-Port** verwendet, da die Container sich direkt über das interne Docker-Netzwerk erreichen und dabei nicht über die auf dem Host veröffentlichten Ports gehen. Der Host-Port wird nur benötigt, wenn von außerhalb des Docker-Netzwerks (z. B. vom eigenen Browser aus) auf einen Dienst zugegriffen werden soll.

## Praktischer Test

Die zusammengeführte Datei `compose.yml` wurde im Hauptverzeichnis des Repositorys getestet:

```bash
docker compose -f compose.yml config
docker compose -f compose.yml up -d
docker compose -f compose.yml ps
docker network inspect lab_net
docker compose -f compose.yml down
```

**Ergebnis:**

- `docker compose -f compose.yml config` lief **ohne Fehler** durch, die YAML-Struktur war also gültig.
- Alle **vier Container** (`pihole`, `portainer`, `watchtower`, `nginx`) konnten erfolgreich gestartet werden (`docker compose ps` zeigte alle vier als `Up`).
- Über `docker network inspect lab_net` konnte bestätigt werden, dass **alle vier Container** mit dem Netzwerk `lab_net` verbunden waren, jeweils mit einer **dynamisch vergebenen** IP-Adresse aus dem Subnetz `172.18.0.0/16` (z. B. `pihole` → `172.18.0.2`, `nginx` → `172.18.0.4`).
- Die Namensauflösung über Servicenamen wurde mit `docker compose exec nginx getent hosts pihole` (sowie `portainer` und `watchtower`) erfolgreich getestet – alle drei Servicenamen wurden korrekt auf die jeweilige Container-IP aufgelöst.
- **Portkonflikte traten keine auf**, da nginx bereits von Anfang an auf Host-Port `8080` statt `80` gemappt war (Port `80` wird von Pi-hole belegt) und die restlichen Ports (`53`, `67`, `9000`) jeweils nur einmal vergeben sind.
- Die Weboberflächen wurden erfolgreich per `curl` geprüft: Nginx (`http://localhost:8080` → 200 OK), Pi-hole (`http://localhost/admin/` → 302 Redirect zum Login, wie erwartet) und Portainer (`http://localhost:9000` → 200 OK).
- **Nötige Anpassung:** Beim Zusammenführen mussten die relativen Volume-Pfade der einzelnen Services (z. B. `./etc-pihole` → `./pihole/etc-pihole`, `./portainer_data` → `./portainer/portainer_data`, `./html` → `./nginx/html`) an das neue Hauptverzeichnis angepasst werden, da sich relative Pfade in `compose.yml` nun auf das Repository-Root statt auf die jeweiligen Unterordner beziehen. Ohne diese Anpassung hätten die Container ihre persistenten Daten in falschen bzw. neuen, leeren Ordnern abgelegt.
- Nach `docker compose -f compose.yml down` wurden alle vier Container entfernt und `docker network ls` bestätigte, dass auch das von Compose verwaltete Netzwerk `lab_net` wieder entfernt wurde.
