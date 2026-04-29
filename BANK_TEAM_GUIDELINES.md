# PingFin26 – Bank Team Guidelines
**International Week 2026 | Netwerkteam**

---

## Hoe het werkt

Jij levert een complete `docker-compose.yml` met jouw app én database erin.  
Wij voeren op de server gewoon `docker-compose up` uit — niets meer.

**Server IP:** `192.168.137.232`
**Docker netwerk:** `pingfin_net`

---

## Stap 1 – Repo structuur

Zorg dat jouw repo er zo uitziet:

```
jouw-repo/
├── docker-compose.yml  ← verplicht | start app + database containers
├── Dockerfile          ← verplicht | bouwt de app container
├── .dockerignore       ← verplicht | voorkomt dat node_modules in container komen
├── .gitignore          ← verplicht | voorkomt dat .env naar GitHub gepusht wordt
├── .env.example        ← verplicht | template met alle variabelen
├── init.sql            ← verplicht | maakt jouw database tabellen aan
├── package.json
└── src/
    └── app.js
```

Repo moet **publiek** zijn op GitHub.

---

## Stap 2 – Dockerfile en .dockerignore aanmaken

Maak een bestand `Dockerfile` in de root. Dit is alleen voor jouw app — geen database hierin.

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 80
CMD ["node", "src/app.js"]
```

> Pas `src/app.js` aan naar jouw eigen entry point (bv. `app.js` of `index.js`).  
> Poort **80** is verplicht binnenin de container.

Maak ook een `.dockerignore` bestand aan in de root:

```
node_modules
.git
.env
*.log
```

> Zonder `.dockerignore` worden jouw lokale `node_modules` gekopieerd in de container, wat buildsfouten veroorzaakt.

Maak ook een `.gitignore` bestand aan in de root:

```
node_modules
.env
*.log
```

> `.gitignore` voorkomt dat `.env` naar GitHub gepusht wordt. `.dockerignore` en `.gitignore` zijn **niet** hetzelfde.

---

## Stap 3 – .env.example aanmaken

Maak een bestand `.env.example` in de root. Dit is een template met alle variabelen die jouw app nodig heeft.

```env
DB_HOST=db
DB_USER=root
DB_PASSWORD=password
DB_NAME=bankdb
DB_PORT=3306

TEAM_NAME=Jouw Banknaam
TEAM_BIC=JOUBBIC1
TEAM_MEMBERS=naam1,naam2,naam3

APP_CONTAINER_NAME=JOUW_APP_NAAM
DB_CONTAINER_NAME=JOUW_DB_NAAM
APP_PORT=JOUW_POORT
```

**Lokaal testen:**  
Kopieer `.env.example` naar `.env`, maak het netwerk aan en start docker-compose:

```bash
cp .env.example .env
docker network create pingfin_net
docker compose up
```

> `pingfin_net` bestaat al op de server — lokaal moet je het eenmalig zelf aanmaken.

> `.env` nooit pushen naar GitHub — het staat in `.gitignore`.  
> `.env.example` wél pushen — dat is enkel een template zonder geheimen.

---

## Stap 4 – init.sql aanmaken

Maak een bestand `init.sql` in de root met jouw tabellen. Dit wordt automatisch uitgevoerd bij de eerste opstart van de database.

```sql
CREATE DATABASE IF NOT EXISTS bankdb;
USE bankdb;

CREATE TABLE IF NOT EXISTS transactions (
  id INT AUTO_INCREMENT PRIMARY KEY,
  from_bic VARCHAR(20),
  to_bic VARCHAR(20),
  amount DECIMAL(10,2),
  currency VARCHAR(3) DEFAULT 'EUR',
  status VARCHAR(20) DEFAULT 'pending'
);
```

Pas de tabellen aan naar jouw eigen noden.

---

## Stap 5 – docker-compose.yml aanmaken

Maak een bestand `docker-compose.yml` in de root.

**Gebruik jouw eigen waarden uit deze tabel:**

| Team | container_name app | container_name db | Poort |
|------|--------------------|-------------------|-------|
| Clearing Bank | `cb_app` | `cb_db` | `8080` |
| Bank IUS | `bankius_app` | `bankius_db` | `8081` |
| Bank KBC | `bankkbc_app` | `bankkbc_db` | `8082` |

```yaml
version: '3.8'

services:
  app:
    build: .
    container_name: ${APP_CONTAINER_NAME}
    ports:
      - "${APP_PORT}:80"
    environment:
      DB_HOST: ${DB_HOST}
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
      DB_NAME: ${DB_NAME}
      DB_PORT: ${DB_PORT}
      TEAM_NAME: ${TEAM_NAME}
      TEAM_BIC: ${TEAM_BIC}
      TEAM_MEMBERS: ${TEAM_MEMBERS}
    depends_on:
      db:
        condition: service_healthy
    networks:
      - pingfin_net
      - internal

  db:
    image: mysql:8.0
    container_name: ${DB_CONTAINER_NAME}
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - internal
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${DB_PASSWORD}"]
      interval: 5s
      timeout: 5s
      retries: 10

networks:
  pingfin_net:
    external: true                    # dit netwerk bestaat al op de server
  internal:
    driver: bridge
```

**Wat hier belangrijk is:**
- `container_name` voor app en db **moet uniek zijn** op de hele server — gebruik exact de namen uit de tabel
- `${VARIABELE}` — Docker Compose leest de waarden automatisch uit jouw `.env` bestand
- `pingfin_net` is `external: true` — dit netwerk bestaat al op de server, jij maakt het niet aan
- De DB staat alleen op `internal` — andere teams kunnen jouw database niet bereiken
- `condition: service_healthy` — de app start pas als MySQL volledig klaar is

---

## Stap 6 – DB connectie in jouw app

Zorg eerst dat `mysql2` in jouw `package.json` staat:

```bash
npm install mysql2
```

Gebruik **altijd** environment variables. Nooit hardcoded waardes.

```javascript
const mysql = require('mysql2');

const db = mysql.createConnection({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  port: process.env.DB_PORT || 3306
});
```

> `DB_HOST` is `db` — de naam van de database service in docker-compose.yml.

---

## Stap 7 – /api/info/ endpoint toevoegen

Dit endpoint is **verplicht**. Wij gebruiken het om te controleren of jouw app correct draait.

```
GET /api/info/
```

Verwacht antwoord:

```json
{
  "ok": true,
  "status": 200,
  "code": 2000,
  "message": "OK",
  "data": {
    "team": "Jouw Banknaam",
    "bic": "JOUBBIC1",
    "members": ["naam1", "naam2"]
  }
}
```

Gebruik de environment variables uit `.env`:

```javascript
app.get('/api/info/', (req, res) => {
  res.json({
    ok: true,
    status: 200,
    code: 2000,
    message: 'OK',
    data: {
      team: process.env.TEAM_NAME,
      bic: process.env.TEAM_BIC,
      members: process.env.TEAM_MEMBERS.split(',')
    }
  });
});
```

---

## Stap 8 – Pushen naar GitHub en link sturen

```bash
git add .
git commit -m "docker setup klaar"
git push
```

Stuur daarna naar het netwerkteam:
- GitHub link
- Jouw teamnaam en BIC
- Naam contactpersoon

---

## Communicatie met andere banken

Vanuit jouw app bereik je andere banken via hun containernaam:

| Service | URL |
|---------|-----|
| Clearing Bank | `http://cb_app/api/...` |
| Bank IUS | `http://bankius_app/api/...` |
| Bank KBC | `http://bankkbc_app/api/...` |

---

## Checklist

```
[ ] Stap 1 – Repo structuur correct
[ ] Stap 2 – Dockerfile aangemaakt + CMD aangepast naar jouw entry point
[ ] Stap 2 – .dockerignore aangemaakt
[ ] Stap 2 – .gitignore aangemaakt
[ ] Stap 3 – .env.example aangemaakt
[ ] Stap 4 – init.sql aangemaakt met jouw tabellen
[ ] Stap 5 – docker-compose.yml aangemaakt + container namen + poort aangepast
[ ] Stap 6 – mysql2 geïnstalleerd en in package.json
[ ] Stap 6 – DB connectie via environment variables
[ ] Stap 7 – /api/info/ endpoint aanwezig en correct
[ ] Stap 8 – Gepusht en link gestuurd naar netwerkteam
```

---

*Vragen? Contacteer het netwerkteam.*
*PingFin26 – International Week 2026 – Odisee*
