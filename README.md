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

- Minst 8 GB RAM (projektet använder totalt ~2 GB med alla fyra VMs igång (2,5 GB vid körning av web3 också.))
- Minst 20 GB ledigt diskutrymme

**Secrets-fil:**

Skapa filen `secrets.env` i projektets rotkatalog innan du kör `vagrant up`. Se avsnittet [Secrets](#secrets).


## Kom igång

```bash
# 1. Klona repot
git clone https://github.com/Virtualiseringsteknik-och-automation/J-H-VIR-projekt.git
cd J-H-VIR-projekt

# 2. Skapa secrets-filen (se avsnittet Secrets nedan)


# 3. Starta alla VMs 
vagrant up

# 4. SSH in på kontrollnoden (db-maskinen)
vagrant ssh db

# 5. Kör playbooken
cd ~/ansible/ansible
ansible-playbook site.yml

# 6. Öppna gästboken i webbläsaren
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



## Säkerhetsanalys



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