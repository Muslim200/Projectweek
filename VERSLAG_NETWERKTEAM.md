# Verslag Netwerkteam – PingFin26
**International Week 2026 | Odisee**

---

## Inleiding

### Beschrijving van het project

PingFin26 is een schoolproject waarbij studenten een vereenvoudigd versie van het SEPA-betalingssysteem simuleren. Verschillende teams bouwen bankapplicaties die met elkaar moeten communiceren via een centrale Clearing Bank. Het project loopt over 4 dagen (27–30 april 2026) en vereist samenwerking tussen een netwerkteam en meerdere bankteams.

### Rol en verantwoordelijkheden van het netwerkteam

Wij als netwerkteam zijn de infrastructuurploeg van het hele project. Terwijl alle andere teams druk bezig zijn met het bouwen van hun bankapplicaties, zorgen wij ervoor dat die applicaties effectief met elkaar kunnen communiceren.

Concreet zetten wij een fysiek bekabeld netwerk op met een switch, de nodige bekabeling, en beheren we een centrale server waarop alle bankapplicaties van de andere teams draaien. Die applicaties worden aangeleverd als Docker-containers, en het is aan ons om die te deployen, draaiende te houden en bereikbaar te maken voor iedereen op het netwerk.

Wij zijn ook het aanspreekpunt voor alle teams als er iets fout gaat met connectiviteit, deployment of netwerkcommunicatie. Wij bepalen zelf welke poorten elk team gebruikt en vertalen dat naar een stabiele, werkende omgeving.

Kortom: zonder ons werk kunnen de banken niet met elkaar communiceren en faalt het hele systeem. Wij zijn de stille motor achter het project.

### Doelstellingen

- Een stabiel lokaal bedraad netwerk opzetten via een switch
- Een centrale server configureren met Docker
- Alle bankapplicaties deployen als Docker-containers op de server
- Inter-container communicatie garanderen via een gedeeld Docker-netwerk
- Duidelijke deployment guidelines opstellen voor de bankteams
- Beschikbaarheid en connectiviteit van alle services monitoren en troubleshooten

---

## Netwerk- en systeemarchitectuur

### Overzicht van de netwerkopstelling

Onze netwerkopstelling bestaat uit een centrale server, een switch, UTP-kabels en een consolekabel. De server fungeert als het hart van de infrastructuur: alle Docker-containers van de bankteams draaien hierop en zijn via het lokale netwerk bereikbaar voor alle deelnemende teams.

| Component | Model |
|-----------|-------|
| Server | Dell OptiPlex-7040 |
| Switch | HP V1910 24G (JE006A) |
| Besturingssysteem server | Ubuntu |
| Server IP | 192.168.137.232 |

Voor de internetverbinding van de server hebben wij gekozen voor **network sharing**: via een UTP-kabel delen wij de wifi-verbinding van onze laptops met de server via Windows ICS (Internet Connection Sharing). Dit was een bewuste en praktische keuze gezien de beschikbare middelen, al maken wij er een gekend risico van dat de internetverbinding afhankelijk is van één laptop. Voor de scope van dit project is dit echter geen probleem.

Alle apparaten bevinden zich in hetzelfde lokale subnet, waardoor de bankteams de server en de daarop draaiende services kunnen bereiken via een vast IP-adres op het lokale netwerk.

**Netwerktopologie:**

```
[Laptop X] ──── (WiFi) ──── School internet
[Laptop X] ──┐
[Laptop 2] ──┤──── [HP Switch V1910] ──── [Server 192.168.137.232]
[Laptop 3] ──┘
     │
[Bank team laptops]
```

### Diagram van de architectuur

```
                    SERVER (192.168.137.232)
          ┌─────────────────────────────────────┐
          │           Docker: pingfin_net        │
          │  ┌──────────┐  ┌──────────────────┐ │
          │  │    CB    │  │     Bank IUS     │ │
          │  │  :8080   │  │      :8081       │ │
          │  └────┬─────┘  └────────┬─────────┘ │
          │       │    pingfin_net  │            │
          │  ┌────┴──────────────────┴────────┐  │
          │  │         Docker DNS             │  │
          │  └────────────────────────────────┘  │
          │  ┌──────────────────┐               │
          │  │     Bank KBC     │               │
          │  │      :8082       │               │
          │  └──────────────────┘               │
          └─────────────────────────────────────┘
                        │
                   [HP Switch]
                        │
        ┌───────────────┼───────────────┐
   [CB laptop]   [Bank IUS laptop]  [Bank KBC laptop]
```

### Beschrijving van de Docker setup

Alle containers draaien op het Docker-netwerk `pingfin_net`. Dit netwerk laat containers toe met elkaar te communiceren via de **containernaam als hostname**, zonder dat IP-adressen nodig zijn. Docker zorgt automatisch voor DNS-resolutie binnen hetzelfde netwerk.

**Docker netwerk aanmaken:**
```bash
docker network create pingfin_net
```

**Poortindeling:**

| Team | Container naam | Poort |
|------|---------------|-------|
| Centrale Bank (CB) | cb_app | 8080 |
| Bank IUS | bankius_app | 8081 |
| Bank KBC | bankkbc_app | 8082 |

---

## Deployment setup

### Hoe applicaties gedeployed worden

Alle bankapplicaties worden gedeployed op onze centrale server via Docker. De bankteams leveren hun applicatie aan als een publieke GitHub-repository met een werkende Dockerfile in de root. Wij als netwerkteam nemen het volledige deploymentproces van daaraf over, zodat de bankteams zich enkel hoeven te focussen op hun eigen applicatie.

### Workflow: van code tot draaiende container

Het deploymentproces verloopt steeds op dezelfde manier. Zodra een team ons hun GitHub-link bezorgt, clonen wij de repository op de server. Vervolgens bouwen wij het Docker-image op basis van hun Dockerfile en starten we de container op in het gedeelde Docker-netwerk `pingfin_net`.

```bash
# 1. Repository klonen
git clone https://github.com/bankteam/their-repo.git
cd their-repo

# 2. Docker image bouwen
docker build -t teamname:latest .

# 3. Container starten op pingfin_net
docker run -d --name teamname_app --network pingfin_net -p PORT:80 teamname:latest

# 4. Verifiëren
curl http://192.168.137.232:PORT/api/info/
```

Op deze manier draaien alle containers op hetzelfde netwerk en kunnen ze onderling met elkaar communiceren.

### Afspraken rond naamgeving, poorten en routing

Om verwarring te vermijden en alles overzichtelijk te houden, hebben wij duidelijke afspraken gemaakt rond naamgeving en poortindeling. Docker-images en containers worden consequent benoemd naar de teamnaam, bijvoorbeeld `teamname:latest` en `teamname_app`. Dit maakt het beheer en het troubleshooten een stuk eenvoudiger wanneer meerdere containers tegelijk draaien.

### Reverse proxy

Momenteel geen reverse proxy geconfigureerd. Elke service is rechtstreeks bereikbaar via het server-IP en de toegewezen poort.

---

## Overzicht van services

### Gedeployde services (Dag 2 – testfase)

| Applicatie | Container naam | Poort | URL |
|------------|---------------|-------|-----|
| dummy_api (test) | dummy_app | 8080 | http://192.168.137.232:8080 |
| caller_api (test) | caller_app | 8081 | http://192.168.137.232:8081 |

*De echte bankapplicaties worden gedeployed op Dag 3.*

### Hoe services met elkaar communiceren

Containers op `pingfin_net` communiceren via Docker DNS. Een container kan een andere aanroepen via de containernaam:

```python
# caller_app roept dummy_app aan via containernaam
requests.get('http://dummy_app/api/info/')
```

Dit werkt zonder IP-adres te kennen. Docker lost de naam automatisch op naar het interne IP van de container.

---

## Testing & connectiviteit

### Aanpak

Om de infrastructuur te testen voordat de echte bankapplicaties beschikbaar waren, hebben wij zelf twee dummy-containers gebouwd en gedeployed. Op die manier konden we verifiëren of onze deployment-workflow correct werkte en of containers onderling met elkaar konden communiceren via het Docker-netwerk.

### Dummy API's

We bouwden twee aparte containers in Python:

- **dummy_app** — een eenvoudige API die data teruggeeft via een JSON-endpoint, bereikbaar op poort 8080
- **caller_app** — een tweede container die de dummy API aanroept, specifiek om inter-container communicatie te testen, bereikbaar op poort 8081

Beide containers draaiden succesvol op de server met status **Up**. Dit bevestigt dat onze deployment-workflow correct werkt.

### Testresultaten

**Test 1 — Health check:**
```bash
curl http://192.168.137.232:8080/api/info/
```
Response:
```json
{"code": 2000, "data": [], "message": "OK", "ok": true, "status": 200}
```

**Test 2 — Inter-container communicatie:**
```bash
curl http://192.168.137.232:8081/api/call/
```
Response:
```json
{
  "dummy_app_response": {"code": 2000, "data": [], "message": "OK", "ok": true, "status": 200},
  "message": "Successfully called dummy_app",
  "ok": true
}
```

Beide endpoints antwoorden met statuscode 200 en code 2000, wat exact overeenkomt met het vereiste formaat uit de deployment guidelines. Dit bevestigt dat de containers correct draaien en bereikbaar zijn via het netwerk, en dat inter-container communicatie via Docker DNS werkt.

### Gebruikte tools

- **Browser** — JSON-responses visueel inspecteren
- **curl** — endpoints testen vanuit de terminal
- **docker ps** — status van draaiende containers opvolgen
- **docker logs** — container logs bekijken bij problemen
- **docker inspect** — netwerkconfiguratie van containers controleren

---

## Beschrijving van het proces

### Plan van aanpak

**Dag 1 – Setup & voorbereiding**
- Hardware ophalen (server, switch, kabels)
- Fysiek netwerk opzetten (switch, kabels)
- Server OS controleren en opstarten
- Docker installeren en testen
- Docker netwerk `pingfin_net` aanmaken
- Deployment guidelines opstellen en publiceren op GitHub
- Communicatie met bankteams over vereisten

**Dag 2 – Integratie & deployment**
- Dummy API's bouwen en deployen
- Inter-container communicatie testen en bevestigen
- Deployment flow verfijnen
- Guidelines aanvullen op basis van bevindingen

**Dag 3 – Stabilisatie & optimalisatie** *(nog te doen)*
- Echte bankapplicaties ontvangen en deployen
- Connectiviteit tussen alle teams testen
- Problemen troubleshooten

**Dag 4 – Presentatie** *(nog te doen)*
- Finale presentatie voorbereiden en geven

### Werkverdeling binnen het team

*(In te vullen door het team)*

### Communicatie en samenwerking met bankteams

De deployment guidelines werden gepubliceerd op GitHub zodat alle teams ze kunnen raadplegen: https://github.com/Muslim200/Projectweek

De bankteams werden gevraagd om:
- Een publieke GitHub-repo aan te maken met hun code + Dockerfile
- Ons de repo-link te bezorgen
- Een `/api/info/` endpoint te voorzien als health check

---

## Probleemafhandeling & betrouwbaarheid

### Overzicht van problemen en oplossingen

**Probleem 1 — Server vast in initramfs bij opstarten**
- *Symptoom:* Server toonde `(initramfs)` prompt en startte niet op.
- *Oorzaak:* Bestaand OS-probleem op de server.
- *Oplossing:* Meerdere keren herstart; server startte uiteindelijk op in de bestaande Ubuntu-installatie.

**Probleem 2 — USB-boot mislukt**
- *Symptoom:* "Selected boot device failed" bij USB-boot.
- *Oorzaak:* De USB was niet correct geflasht.
- *Oplossing:* Beslissing om niet te herinstalleren — server startte op in bestaande Ubuntu.

**Probleem 3 — Server heeft geen WiFi**
- *Symptoom:* Server kon niet draadloos verbinden met internet.
- *Oplossing:* Windows ICS (Internet Connection Sharing) via Laptop X. Laptop X deelt zijn WiFi via ethernet-kabel door de switch naar de server.

**Probleem 4 — Statisch IP conflicteert met netplan**
- *Symptoom:* IP via Ubuntu GUI instellen werkte niet correct.
- *Oorzaak:* Zowel netplan als de GUI probeerden dezelfde interface te beheren.
- *Oplossing:* Netplan-configuratiebestand rechtstreeks bewerkt en `sudo netplan apply` uitgevoerd.

**Probleem 5 — DNS werkt niet na statisch IP**
- *Symptoom:* `ping 8.8.8.8` werkte maar `ping google.com` niet.
- *Oorzaak:* `/etc/resolv.conf` verwees naar `127.0.0.53` (niet functioneel).
- *Oplossing:*
```bash
sudo rm /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

**Probleem 6 — Gateway niet bereikbaar**
- *Symptoom:* Server kon `192.168.137.1` (Laptop X) niet bereiken na herstart.
- *Oorzaak:* ICS op Laptop X was gereset na herstart.
- *Oplossing:* ICS opnieuw activeren op Laptop X via Netwerkinstellingen → Delen.

**Probleem 7 — `curl` niet beschikbaar in container**
- *Symptoom:* `docker exec caller_app curl ...` faalde.
- *Oorzaak:* `python:3.11-slim` bevat geen extra tools.
- *Oplossing:* Python's ingebouwde `urllib` gebruikt voor de test:
```bash
docker exec caller_app python -c "import urllib.request; print(urllib.request.urlopen('http://dummy_app/api/info/').read())"
```

**Probleem 8 — caller_api miste `/api/info/` endpoint**
- *Symptoom:* `curl .../api/info/` gaf 404 terug op poort 8081.
- *Oorzaak:* Endpoint ontbrak in de code, terwijl onze eigen guidelines dit verplicht stellen.
- *Oplossing:* Endpoint toegevoegd aan `app.py`, container herbouwd en herstart.

**Probleem 9 — curl faalde direct na `docker run`**
- *Symptoom:* Verbinding geweigerd direct na starten van container.
- *Oorzaak:* Container was nog niet volledig opgestart (timing).
- *Oplossing:* Enkele seconden gewacht en opnieuw getest.

### Maatregelen om stabiliteit te verbeteren

- Statisch IP ingesteld op de server zodat het IP niet verandert na herstart
- Alle containers worden gestart met `--restart unless-stopped` *(aanbevolen voor productie)*
- Health check via `/api/info/` na elke deployment
- Containers enkel gestopt en verwijderd als dat nodig is, nooit zomaar

---

## Handleiding / Deployment guide

### Stappen om een nieuwe applicatie te deployen

```bash
# 1. Repository klonen
git clone https://github.com/bankteam/their-repo.git
cd their-repo

# 2. Image bouwen
docker build -t teamname:latest .

# 3. Container starten
docker run -d --name teamname_app --network pingfin_net -p PORT:80 teamname:latest

# 4. Controleren
curl http://192.168.137.232:PORT/api/info/
```

### Vereisten voor bankteams

1. Publieke GitHub-repository met code + `Dockerfile` in de root
2. De applicatie moet draaien op poort 80 binnen de container
3. Verplicht endpoint: `GET /api/info/` met response:
```json
{
  "ok": true,
  "status": 200,
  "code": 2000,
  "message": "OK",
  "data": {
    "team": "Teamnaam",
    "bic": "BICCODE1",
    "members": ["naam1", "naam2"]
  }
}
```

### Hoe controleren of een service correct werkt

```bash
# Container draait?
docker ps

# Health check
curl http://192.168.137.232:PORT/api/info/

# Logs bekijken bij problemen
docker logs container_naam
```

---

## Reflectie

*(Wordt ingevuld op Dag 4)*

### Terugblik op het project

*(In te vullen)*

### Geleerde lessen

*(In te vullen)*

### Wat zou je anders aanpakken?

*(In te vullen)*

### Suggesties om de workshop te verbeteren

*(In te vullen)*

---

## Bijlagen

### Bijlage A – Docker commando's referentie

```bash
# Alle draaiende containers
docker ps

# Alle containers (ook gestopte)
docker ps -a

# Container stoppen en verwijderen
docker stop naam && docker rm naam

# Logs bekijken
docker logs naam

# Netwerk van container controleren
docker inspect naam | grep -A5 '"Networks"'

# Container verbinden met netwerk
docker network connect pingfin_net naam
```

### Bijlage B – Server netwerkconfiguratie

```
Interface: enp0s31f6
IP: 192.168.137.232/24
Gateway: 192.168.137.1 (Laptop X via ICS)
DNS: 8.8.8.8
```

### Bijlage C – GitHub repository

Deployment guidelines en documentatie:
https://github.com/Muslim200/Projectweek

---

*Netwerkteam – PingFin26 – International Week 2026 – Odisee*
