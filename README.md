# J-H-VIR-projekt. Lastbalanserad webbinfrastruktur med gästbok/chatt

> En automatiserad infrastruktur med en lastbalanserad webbapplikation (chatt/gästbok) driftsatt på två identiskt konfigurerade webbservrar via Ansible, med en gemensam PostgreSQL-databas och en Nginx-lastbalanserare med automatisk feldetektering. Infrastrukturen är skalbar och fler webservrar kan läggas till med enkla medel.

---

## Innehållsförteckning

- [Arkitektur](#arkitektur)
- [Miljöer och IP-adresser](#miljöer-och-ip-adresser)
- [Mappstruktur](#mappstruktur)
- [Komponenter](#komponenter)
- [Krav och förutsättningar](#krav-och-förutsättningar)
- [Kom igång](#kom-igång)
- [Secrets](#secrets)
- [Säkerhetsåtgärder](#säkerhetsåtgärder)
- [Säkerhetsanalys](#sökerhetsanalys)
- [Verifiering](#verifiering)
- [Designval och motivering](#designval-och-motivering)

---

## Arkitektur

![Ansibles funktion](docs/Topologi%20Ansible.jpg)
![Webbappens funktion](docs/Topologi%20webbapp.jpg)

## Miljöer och IP-adresser
| VM | Roll | IP-adress | Port forwarding | Beskrivning |
|---|---|---|---|---|
| `lb` | Lastbalanserare | 192.168.56.10 | `:80 → host:8080` | Nginx tar emot all inkommande trafik och fördelar den med round-robin |
| `web1` | Webbserver | 192.168.56.11 | — | Apache + Python, konfigurerad identiskt med web2|
| `web2` | Webbserver | 192.168.56.12 | — | Apache + Python, konfigurerad identiskt med web1|
| `db` | Databas + kontrollnod | 192.168.56.13 | — | PostgreSQL-databas samt Ansible-kontrollnod. Ingen port forwarding — ej nåbar utifrån |
| *`web3`* | *Webbserver* | *192.168.56.14* | — | *Exempelwebbserver som visar skalbarheten. Apache + Python, konfigurerras identiskt med web1 och web2*|

## Mappstruktur
repo/
│
├── ansible/
│   ├── ansible.cfg          # Ansible-konfiguration (inventory, remote_user, osv)
│   ├── inventory.ini        # Vilka servrar Ansible hanterar och i vilka grupper
│   ├── site.yml             # Master playbook — kör alla roller i rätt ordning
│   ├── secrets_example.yml  # Mall för secrets.yml (inga riktiga värden)
│   │
│   ├── vars/
│   │   └── main.yml         # Delade variabler (IP-adresser, portar, sökvägar)
│   │
│   └───roles/
│         ├── common/           # Driftsätter Flask-applikationen
│         │     └── main.yml
│         ├── database/
│         │     ├──handlers/ 
│         │     │      └── main.yml 
│         │     └── tasks/
│         │            └── main.yml
│         ├── loadbanancer/
│         │     ├── handlers
│         │     │      └── main.yml
│         │     ├── tasks/
│         │     │      └── main.yml
│         │     └── templates
│         │            └── loadbalancer.conf.j2
│         └── webserver           # Installerar och konfigurerar nginx som lastbalanserare
│               ├── tasks/
│               │      └── main.yml
│               ├── handlers/
│               │      └── main.yml
│               └── templates/
│                      └── nginx.conf.j2
│
├── docs/
│   ├── Topologi Ansible.jpg
│   └── Topologi webbapp.jpg
│
├── Vagrantfile          # här definieras alla VMar och nätverksinställningar
├── secrets.yml          # GITIGNORERAD — lösenord och känsliga värden
├── .gitignore
└── README.md



## Komponenter

### Vagrantfile

Definierar fyra virtuella maskiner i VirtualBox med ett gemensamt host-only-nätverk (`192.168.56.0/24`). Port forwarding på lb-VM:en (`80 → 8080`) gör att webbapplikationen är nåbar från Windows-hosten. Databasservern har medvetet ingen port forwarding.

`db`-VM:en fungerar även som Ansible-kontrollnod. Vid uppstart klonar den automatiskt repot från GitHub och genererar ett SSH-nyckelpar. Den publika nyckeln sparas i den delade Vagrant-mappen och kopieras automatiskt till övriga VMs vid deras bootstrap — det är så Ansible kan nå dem utan lösenord.

### ansible.cfg

Konfigurationsfilen innehåller två sektioner. Under `[defaults]` inaktiveras `host_key_checking` vilket gör att Ansible inte ställer en kontrollfråga första gången den ansluter till en ny server — praktiskt i en labbmiljö där VMs återkapas ofta. Inventory pekas ut till `./inventory.ini` så att man inte behöver ange den manuellt vid varje körning. Under `[ssh_connection]` aktiveras `allow_world_readable_tmpfiles` vilket tillåter Ansible att läsa temporära filer från mappar som är tillgängliga för alla användare, något som krävs när Ansible ansluter som en annan användare än den som äger filerna.

### site.yml

Master playbook som innehåller 4 plays som exekveras i följande ordning: 1. 'common' exekveras på samtliga hostar och ser till att, uppdateringar görs och installerar "bra att ha verktyg", så som curl och htop. 2. 'databasen' som körs emot databas noden. Här installeras och konfigureras PostgreSQL, samt nödvändiga inställningar som säkrar komunikation från och till webbservrarna. 3. 'webserver' riktar sig emot webb noderna och installerar apache2, Flask och relaterade paket. Den kofigurerar även miljön så att apache2 kan exekvera pythonkod samt kopiera in app koden från templates till sin Flask miljö 4. 'loadbalancer' körs sist och installerar nginx som sedan konfigureras till att agera lastbalanserare eftersom att webbservrarna redan är igång så kan nginx direkt börja dirigera trafiken till dom.

### Rollen common
Försöker göra en uppdatering om installationen är äldre en 1 timma och ser till att 3 "bra att ha" felsöknings paket är installerade 

### Rollen webserver
Denna roll ansvarar för att driftsätta och konfigurera Flask-applikationen. I denna miljö driftas Flask via Apache2 och mod_wsgi. Rollen Installerar apache2, python3-flask, python3-psycopg2 och modulen libapache2-mod-wsgi-py3 för Python-exekvering. sedan skapar applokationskatalogen /var/www/chatt som inkluderar bland annat undermappar för HTLM med rätt ägenderättigheter för www-data. Rollen utnytjar Ansibles Jinja2 templates för att skicka in miljövariablerdirekt i koden. Som då inkluderar databasuppgifter, WSGI-konfiguration och dynamiska IP-adresser till HTML-bannern. Apaches VirtualHost konfigureras med dedikerade WSGI-processgrupper. Trafik begränsas via en "Require ip regel" som säkerställer att endast lastbalanceraren får ansluta till webbservern.

### Rollen loadbalancer

Installerar Nginx och renderar `loadbalancer.conf.j2` med ett `upstream`-block som itererar över alla servrar i `[webservers]`. Varje server konfigureras med `max_fails=3 fail_timeout=30s` vilket är Nginx:s passiva health check — om en server inte svarar på tre requests tas den automatiskt bort från rotationen i 30 sekunder. Handlers startar om och laddar om Nginx.

### Rollen databas
Denna roll ansvarar för att installera, konfigurera och säkra PostgreSQL-databasen. Till skillnad från en statisk konfiguration är denna roll utformad för att dynamiskt anpassa sig efter den skalbara webbserver-miljön. Här installeras postgresql, postgresql-contrib samt paketen acl och python3-psycopg2 för att Ansible ska kunna hantera rättigheter och kommunicera med databasen. Rollen skapar databasen och en dedikerad applikationsanvändare. Både databasnamn och inloggningsuppgifter hämtas säkert från group_vars. Automatiskt skapas tabellen messages med informations kolumner relevanta till vart och när medelandet kom ifrån och tilldelar applikationsanvändaren nödvändiga rättigheter till tabellen. PostgreSQL konfigureras till att lyssna på nätverket (listen_addresses = '*'). För att uppräthålla en hög säkerhetsnivå och full skalbarhet används en Ansible-loop som kollar över gruppen [webservers] i inventariet. Denna loop lägger dynamiskt in webbservrarnas aktuella IP-adresser i "gästlistan" (pg_hba.conf), vilket innebär att endast aktiva webbservrar tillåts ansluta. All autentisering säkras dessutom med krypteringsmetoden scram-sha-256. Ansible-handlers används för att automatiskt starta om eller ladda om PostgreSQL vid uppdateringar av konfigurationsfilerna.

## Krav och förutsättningar

**Programvara som måste vara installerad på Windows-hosten:**

- [VirtualBox](https://www.virtualbox.org/) — testat med version 7.2.6
- [Vagrant](https://www.vagrantup.com/) — testat med version 2.4.9
- [Git](https://git-scm.com/) - testat med version 2.52.0.windows.1

**Hårdvarukrav:**

- Minst 16 GB RAM (projektet använder totalt ~8 GB med alla fyra VMs igång (10 GB vid körning av web3 också.))
- Minst 4 CPU-kärnor (Rekommenderat är 6 CP-kärnor)
- Minst 50 GB ledigt diskutrymme
Mängden RAM och cpu kan även justeras i "Vagrantfile". Dessa krav är beräknade på 1st Lastbalancerare, 1st Databas och 2st webservrar. Vid uppskalning med fler webservrar ökar kraven. 


## Kom igång

```bash
# 1. Klona repot
git clone https://github.com/Virtualiseringsteknik-och-automation/J-H-VIR-projekt.git
cd J-H-VIR-projekt

# 2. Starta alla VMs 
vagrant up

# 3. SSH in på kontrollnoden (db-maskinen)
vagrant ssh db

# 4. Kör playbooken
cd ~/ansible/ansible
ansible-playbook site.yml

# 5. Öppna gästboken i webbläsaren
# http://localhost:8080/app.py
```

**Förväntat slutresultat:**

Öppna `http://localhost:8080/app.py` i webbläsaren. Du ska se gästboken med ett formulär. Skriv ett inlägg — det sparas i databasen och syns direkt. Bannern längst ned i webbläsarfönstret visar vilken webbserver (`web1` eller `web2`) som svarade på requesten.

### Lägga till en ny webbserver

Ny webbserver läggs till i **två filer** på hostmaskinen utan att ändra någon annan konfiguration:

**Vagrantfile** — (Webbservern "web3" finns redan inlagd för proof of concept ta enbart bort hashen "#" för att lägga till den. Det behövs göras på rad 10 och raderna 175 - 205)

**ansible/inventory.ini** — Tag bort hashen "#" på raden för "web3":

```bash
#Starta den nya webservern
vagrant up web3 

#SSH:a in i databasmaskinen 
vagrant ssh db

#Kör ansible playbook 
ansible-playbook site.yml

#Nginx-konfigurationen uppdateras automatiskt och börjar skicka trafik till den nya servern.
```
---

## Secrets

Filen `secrets.env` måste skapas lokalt och **ska aldrig committas till Git** (den finns i `.gitignore`).

Skapa filen i projektets rotkatalog med följande innehåll:

```
export DB_PASSWORD="DITT_LÖSENORD_HÄR"
```

Filen läses av group_vars/all.yml och innehåller lösenord som webbservrarna använder för att kunna prata med databasen.

---


## Säkerhetsåtgärder
**1. Förberedelse för secretsfilen**

Vagrantfilen kan läsa in secretsfilen via Ruby för att kunna anropa variabler i secretsfilen:

```ruby
secrets = {}
File.foreach("secrets.env") do |line|
  key, value = line.strip.split("=")
  secrets[key] = value
end
```
Inga secrets finns i filen idag men detta är förberett för framtida användande i samband med härdning. Secretsfilen användes även 
under utvecklingen av denna miljö då git-repot var privat och användandet av en token var nödvändigt. Git-token fick då bo i secretsfilen.

**2. Direkt åtkomst till webbservrar blockeras**

Apache-konfigurationen begränsar inkommande trafik till enbart
lastbalansererarens IP-adress via en `<Location>`-regel:

```apache
<Location />
    Require ip {{ hostvars['lb']['ansible_host'] }}
</Location>
```

Det innebär att det inte går att nå webbservrarnas IP-adresser direkt
utifrån — all trafik måste passera lastbalanseraren. Det skyddar mot
att en angripare kringgår lastbalanseraren och når applikationen direkt.

**3. Databasåtkomst begränsad till specifika IP-adresser**

PostgreSQL:s `pg_hba.conf` konfigureras automatiskt av Ansible med en
post per webbserver, begränsad till exakt deras IP-adress med `/32`:
host  db  vagrant  192.168.56.11/32  scram-sha-256
host  db  vagrant  192.168.56.12/32  scram-sha-256

Endast webbservrar registrerade i inventory kan ansluta till databasen.
En ny VM på nätverket kan inte ansluta utan att läggas till i inventory
och att playbooken körs om.

**4. Lösenordskryptering med scram-sha-256**

PostgreSQL-autentiseringen använder `scram-sha-256` vilket är den
starkaste lösenordsbaserade autentiseringsmetoden i PostgreSQL. Det
innebär att lösenordet aldrig skickas i klartext över nätverket —
istället utförs en kryptografisk handskakningsprocess.

**5. Skydd mot SQL-injektion**

Gästboksapplikationen använder parametriserade queries i alla
databasanrop:

```python
cur.execute('INSERT INTO messages (message) VALUES (%s)', (new_message,))
```

Användarstyrd input kombineras aldrig direkt med SQL-strängar. Det
förhindrar att en angripare kan manipulera databasfrågor genom att
skriva SQL-kod i meddelandefältet.

**6. Databasen saknar port forwarding**

Databasens VM har medvetet ingen port forwarding konfigurerad i
Vagrantfilen. Det innebär att PostgreSQL (port 5432) inte är nåbar
från värddatorn eller internet — enbart från det privata nätverket
`192.168.56.0/24`.

**7. SSH-nyckelbaserad autentisering**

Ansible ansluter till alla VMs med ett ED25519-nyckelpar som genereras
automatiskt vid bootstrap av db-maskinen. Lösenordsbaserad SSH-inloggning
används inte. ED25519 är en modern elliptisk kurva-algoritm som anses
säkrare än det äldre RSA.

---


## Säkerhetsanalys
**Brist 1: Databaslösenord i klartext i group_vars**

`db_password` i `group_vars/all.yml` innehåller lösenordet i klartext.
Den som har tillgång till repot kan läsa det.

*Produktionslösning:* Ansible Vault krypterar känsliga variabler:
`ansible-vault encrypt_string 'lösenord' --name db_password`
Lösenordet kan även läggas i "secrets.env" för att inte pushas till git.

*Accepterat i denna miljö eftersom:* Miljön är isolerad
och lösenordet skyddar en labbdatabas utan känslig data. Bristen är
dokumenterad i koden med en kommentar.

---

**Brist 2: PostgreSQL lyssnar på alla IP-adresser**

`listen_addresses = '*'` gör att PostgreSQL accepterar anslutningsförsök
från alla IP-adresser. Åtkomsten begränsas av `pg_hba.conf`, men det
vore bättre att begränsa `listen_addresses` till enbart det privata
nätverkets adresser.

*Produktionslösning:*
listen_addresses = '192.168.56.13'

*Accepterat i denna miljö eftersom:* `pg_hba.conf` nekar alla
anslutningar som inte kommer från kända IP-adresser, och databasen
saknar port forwarding vilket gör den inte nåbar utifrån.

---

**Brist 3: Okrypterad intern kommunikation**

Trafiken mellan Nginx och webbservrar (HTTP) samt mellan webbservrar
och databasen (PostgreSQL utan TLS) är okrypterad. En angripare med
tillgång till det interna nätverket kan läsa trafiken.

*Produktionslösning:* Konfigurera TLS i Nginx för HTTPS samt aktivera
SSL i PostgreSQL-anslutningen med certifikat.

*Accepterat i denna miljö eftersom:* Nätverket `192.168.56.0/24` är
ett isolerat host-only-nätverk i VirtualBox som enbart är tillgängligt
från värddatorn.

---

**Brist 4: Felsökningsverktyg installerade på alla VMs**

`curl`, `nano` och `htop` är installerade på samtliga VMs via
common-rollen. I en produktionsmiljö minimerar man antalet installerade
paket för att minska attackytan — varje installerat program är en
potentiell sårbarhet.

*Produktionslösning:* Ta bort paketen från common-rollen och installera
dem enbart vid behov med `ansible-playbook --tags debug`.

*Accepterat i denna miljö eftersom:* Verktygen används aktivt för
felsökning och verifiering under utvecklingen av projektet.

---

**Brist 5: Passiv health check utan aktiv övervakning**

Nginx:s health check är passiv — den upptäcker fel först när en request
faktiskt misslyckas (`max_fails=3 fail_timeout=30s`). En server som är
degraderad men fortfarande svarar långsamt tas inte bort från rotationen.

*Produktionslösning:* Nginx Plus eller HAProxy erbjuder aktiva health
checks som proaktivt testar servrar i bakgrunden utan att vänta på
misslyckade requests. Projektet inkluderar ett verifieringsskript 
(health.py) som manuellt kan köras för att kontrollera databas- och 
Apache-status på varje webbserver, men detta triggas inte automatiskt av 
systemet. En fullständig aktiv health check skulle kräva att skriptet 
anropas kontinuerligt av ett övervakningssystem som Nagios eller via cron.

*Accepterat i denna miljö eftersom:* Nginx Open Source saknar stöd för
aktiva health checks utan betaltillägg. Den passiva lösningen räcker
för att demonstrera automatisk felhantering i en labbmiljö.

## Verifiering

Från hostmaskinen
```Bash
#Anslut till någon av webservrarna via vagrant ssh:
vagrant ssh web1

#Kör följande kommando:
python3 /usr/local/bin/health.py

#Nu körs en healtcheck för att kontrollera anslutningen från webservern till både lastbalanseraren och databasen. Förväntad output är:
--- Health Check för Webbserver ---
[ + ] Databasanslutning: OK
[ + ] Apache2-tjänst: OK (Körs)
-----------------------------------
Status: HEALTHY
```

## Designval och motivering