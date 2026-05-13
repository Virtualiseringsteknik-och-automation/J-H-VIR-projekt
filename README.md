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



## Komponenter

### Vagrantfile

Definierar fyra virtuella maskiner i VirtualBox med ett gemensamt host-only-nätverk (`192.168.56.0/24`). Port forwarding på lb-VM:en (`80 → 8080`) gör att webbapplikationen är nåbar från Windows-hosten. Databasservern har medvetet ingen port forwarding.

`db`-VM:en fungerar även som Ansible-kontrollnod. Vid uppstart klonar den automatiskt repot från GitHub och genererar ett SSH-nyckelpar. Den publika nyckeln sparas i den delade Vagrant-mappen och kopieras automatiskt till övriga VMs vid deras bootstrap — det är så Ansible kan nå dem utan lösenord.

### ansible.cfg

Konfigurationsfilen innehåller två sektioner. Under `[defaults]` inaktiveras `host_key_checking` vilket gör att Ansible inte ställer en kontrollfråga första gången den ansluter till en ny server — praktiskt i en labbmiljö där VMs återkapas ofta. Inventory pekas ut till `./inventory.ini` så att man inte behöver ange den manuellt vid varje körning. Under `[ssh_connection]` aktiveras `allow_world_readable_tmpfiles` vilket tillåter Ansible att läsa temporära filer från mappar som är tillgängliga för alla användare, något som krävs när Ansible ansluter som en annan användare än den som äger filerna.


## Krav och förutsättningar



## Kom igång



## Secrets



## Säkerhetsåtgärder



## Säkerhetsanalys



## Verifiering



## Designval och motivering