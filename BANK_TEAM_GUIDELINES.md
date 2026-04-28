# PingFin26 – Bank Team Guidelines
**International Week 2026 | Netwerkteam**

Ctrl + Shift + V 

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
├── docker-compose.yml  ← verplicht
├── Dockerfile          ← verplicht
├── .dockerignore       ← verplicht
├── init.sql            ← verplicht
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

---

## Stap 3 – init.sql aanmaken

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

## Stap 4 – docker-compose.yml aanmaken

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
    container_name: JOUW_APP_NAAM     # ← uit de tabel hierboven
    ports:
      - "JOUW_POORT:80"               # ← uit de tabel hierboven
    environment:
      DB_HOST: db
      DB_USER: root
      DB_PASSWORD: password
      DB_NAME: bankdb
    depends_on:
      db:
        condition: service_healthy    # ← wacht tot MySQL echt klaar is
    networks:
      - pingfin_net
      - internal

  db:
    image: mysql:8.0
    container_name: JOUW_DB_NAAM      # ← uit de tabel hierboven
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: bankdb
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - internal
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-ppassword"]
      interval: 5s
      timeout: 5s
      retries: 10                     # ← probeert 10x, elke 5s

networks:
  pingfin_net:
    external: true                    # dit netwerk bestaat al op de server
  internal:
    driver: bridge
```

**Wat hier belangrijk is:**
- `container_name` voor app en db **moet uniek zijn** op de hele server — gebruik exact de namen uit de poortindeling tabel
- `pingfin_net` is `external: true` — dit netwerk bestaat al op de server, jij maakt het niet aan
- De DB staat alleen op `internal` — andere teams kunnen jouw database niet bereiken
- `DB_HOST: db` — de app bereikt de database via de service naam `db`, niet via `localhost`
- `condition: service_healthy` — de app start pas als MySQL volledig klaar is om verbindingen te accepteren

---

## Stap 5 – DB connectie in jouw app

Zorg eerst dat `mysql2` in jouw `package.json` staat:

```bash
npm install mysql2
```

Gebruik **altijd** environment variables. Nooit hardcoded waardes.

```javascript
const mysql = require('mysql2');

const db = mysql.createConnection({
  host: process.env.DB_HOST,         // waarde: 'db'
  user: process.env.DB_USER,         // waarde: 'root'
  password: process.env.DB_PASSWORD, // waarde: 'password'
  database: process.env.DB_NAME,     // waarde: 'bankdb'
  port: process.env.DB_PORT || 3306
});
```

> `DB_HOST` is `db` omdat dat de naam is van de database service in docker-compose.yml. Docker lost die naam automatisch op naar het juiste IP.

---

## Stap 6 – /api/info/ endpoint toevoegen

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

---

## Stap 7 – Pushen naar GitHub en link sturen

```bash
git add .
git commit -m "docker setup klaar"
git push
```

Stuur daarna naar het netwerkteam:
- GitHub link
- Jouw teamnaam en BIC
- Naam contactpersoon
- Extra environment variables als je die nodig hebt

---

## Poortindeling

| Team | Container naam | Poort |
|------|---------------|-------|
| Clearing Bank | `cb_app` | 8080 |
| Bank IUS | `bankius_app` | 8081 |
| Bank KBC | `bankkbc_app` | 8082 |

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
[ ] Stap 3 – init.sql aangemaakt met jouw tabellen
[ ] Stap 4 – docker-compose.yml aangemaakt + container namen + poort aangepast
[ ] Stap 5 – mysql2 geïnstalleerd en in package.json
[ ] Stap 5 – DB connectie via environment variables (geen hardcoded waardes)
[ ] Stap 6 – /api/info/ endpoint aanwezig en correct
[ ] Stap 7 – Gepusht en link gestuurd naar netwerkteam
```

---

*Vragen? Contacteer het netwerkteam.*
*PingFin26 – International Week 2026 – Odisee*
