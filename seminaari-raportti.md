Multi-Tenant Architecture & JWT Authentication – Seminaarityö
Sisällysluettelo

Aihe

Tavoite

Tutkimuskysymykset

Johdanto ja tavoite

Teoreettinen tausta

Multi-tenancy-mallit

Päätökset ja hyväksymiskriteerit

Datan malli: käyttäjä useammassa tenantissa

Direct Tenant Ownership

User-Centric Membership-malli

Autentikaatiovirta ja tenantin valinta

JWT-autentikoinnin kriittiset huomioitavat asiat

Tenantin tunnistaminen

Datan eristäminen kyselytasolla

Käyttäjä useassa tenantissa

LoginTicket–väliaikainen JWT

Access Token

Yhteenveto

Aihe

Multi-tenant-arkkitehtuurin tietokantarakenne ja sen vaikutus JWT-autentikaatioon.

Tavoite

Tavoitteena on selvittää, miten moniasiakasympäristön (multi-tenant) tietokantarakenne muodostetaan ja miten se vaikuttaa JWT-autentikaatioon.
Lopuksi toteutan pienen testin, joka osoittaa autentikoinnin ja eristyksen toimivuuden eri tenanttien välillä — myös skenaarion, jossa käyttäjä kuuluu useampaan tenanttiin.

Tutkimuskysymykset

Miten multi-tenant-tietokanta eroaa single-tenant-ratkaisusta?

Miten tenantin tunnistus toteutetaan JWT-autentikaatiossa?

Miten eri tenantit eristetään tietoturvallisesti?

Miten toteutetaan testaus, jossa käyttäjä kuuluu useampaan tenanttiin?

Johdanto ja tavoite

Tässä seminaarityössä tutkin, miten moniasiakasarkkitehtuuri (multi-tenancy) toteutetaan ja miten se vaikuttaa autentikaatioon. Käytän esimerkkinä omaa SaaS-sovellustani, jossa toteutan JWT-pohjaisen autentikoinnin ja tenant-kohtaisen tietokantaeristyksen.
Lopuksi rakennan yksinkertaiset testit, jotka todentavat ratkaisun toimivuuden.
Aihe on ollut minulle teknisesti haastava ja ajankohtainen SaaS-kehityksen näkökulmasta.

Teoreettinen tausta
Multi-tenancy-mallit
1. Shared DB, shared schema

Kaikki tenantit jakavat samat taulut

Eristys tehdään tenant_id-sarakkeen avulla

Nopein rakentaa, mutta tietoturvariski on suurin, jos eristys pettää

2. Shared DB, separate schemas

Yhteinen tietokanta, mutta jokaisella tenantilla oma schema

Parempi eristys

Hallinta vaikeampaa etenkin suurissa ympäristöissä

3. Separate DB per tenant

Jokaisella tenantilla oma tietokanta

Kallein mutta turvallisin malli

Soveltuu suurille asiakasorganisaatioille

Tässä työssä käytän mallia shared schema, koska se on MVP-vaiheessa kustannustehokas ja selkeä tapa havainnollistaa JWT-autentikaation ja tenant-eristyksen välistä vuorovaikutusta.

Päätökset ja hyväksymiskriteerit
Valinnat:

NestJS + Prisma + PostgreSQL + JWT (shared schema)

Hyväksymiskriteerit:

JWT sisältää tenantId-arvon

Käyttäjä, jolla on useampi membership, valitsee aktiivisen tenantin kirjautumisen yhteydessä

Kaikki endpointit palauttavat vain valitun tenantin dataa

Testit todentavat tenant-eristyksen ja monitenanttijäsenyyden

Datan malli: käyttäjä useammassa tenantissa
1. Direct Tenant Ownership

Suora viittaus tenantId → data “omistaa” yhden yrityksen

Sopii entiteeteille kuten: Ride, Invoice, Report

onDelete: Cascade poistaa kaiken tenantin datan sen poistuessa

Kuva →

2. User-Centric Membership-malli

Mahdollistaa tilanteet, joissa käyttäjä kuuluu useisiin yrityksiin.

Keskeiset hyödyt:

Many-to-Many User ↔ Tenant

Selkeä roolien hallinta per tenant

Käyttäjän oikeudet riippuvat membershipistä, ei käyttäjästä itsestään

Skaalautuu hyvin

Kuva →

Autentikaatiovirta ja tenantin valinta
Taustaa

JWT oli minulle aluksi uusi ja haastava konsepti. Opin sen ensin Spring Bootilla ja jälkeenpäin NestJS:llä, mikä antoi minulle syvemmän ymmärryksen.

Kun sovellus toteutetaan multi-tenant shared schema -mallilla, JWT ei ole enää pelkkä kirjautumistunniste.
Sen tulee varmistaa:

käyttäjän oikeus dataan

oikea tenant-konteksti

datan eristäminen

manipulointiyritysten estäminen

JWT-autentikoinnin kriittiset huomioitavat asiat
1. Tenantin tunnistaminen

Multi-tenant sovelluksessa JWT:ssä täytyy olla tieto siitä, mihin tenanttiin käyttäjä on kirjautunut.

Tokeniin lisätään:

tenantId

Payload-esimerkki:

Kuva →

Backend käsittelee jokaisen requestin tämän tiedon perusteella.

2. Datan eristäminen kyselytasolla

Shared schema -mallissa eristys on toteutettava ohjelmallisesti:

SELECT * FROM rides WHERE tenantId = :tokenTenantId


Keskeistä:

tenantId tulee aina tokenista, ei requestista

Jos yksikin endpoint unohtaa suodattaa tenantId:n, seurauksena voi olla tietovuoto

Kuva →

3. Käyttäjä useassa tenantissa

Monissa SaaS-sovelluksissa käyttäjä voi kuulua useampaan tenanttiin.

Tämä aiheuttaa haasteen:

Miten valitaan “aktiivinen tenant”?

Valitaanko se ennen kirjautumista vai kirjautumisen jälkeen?

Huono ensimmäinen idea (hylätty):

Lisätä emailiin tenant-suffix
john@gmail.com → john@gmail.com#TenantABC

Ongelmia:

Kirjautumislogiikka monimutkainen

Vaatii suuria migraatioita

Virheherkkä ja huonosti skaalautuva

Lopullinen toteutettu ratkaisu

Käyttäjä lähettää POST /auth/login

Jos käyttäjällä on useampi membership:

Backend palauttaa tenant-listan

Samalla luodaan loginTicket, joka on lyhytkestoinen JWT

Frontend näyttää tenantit → käyttäjä valitsee

POST /auth/tenant-selection → backend varmistaa valinnan

Luodaan lopullinen Access Token, jossa on:

sub

email

tenantId

role

Kuva →

4. LoginTicket – väliaikainen JWT

Väliaikainen JWT, joka sisältää vain tenant-tiedot.
Se on voimassa lyhyen aikaa (esim. 5 min).

Hyödyt:

Ei voi kutsua API-päätepisteitä

Ei sisällä arkaluontoisia tietoja

Tenantia ei voi vaihtaa kesken session

Turvallinen ja selkeä flow

Kuva →

5. Access Token

Kun tenantti on valittu, backend luo varsinaisen pääsynhallintatokenin.

Token sisältää:

sub
email
tenantId
role


Kaikki API-kutsut tapahtuvat tämän tenantin kontekstissa.

Kuva →

Yhteenveto

Multi-tenant shared schema -arkkitehtuurissa:

TenantId täytyy sisällyttää JWT-tokeniin

Kaikki kyselyt suodatetaan tokenin tenantId:n perusteella

Käyttäjä voi kuulua useaan tenanttiin → aktiivinen tenant on valittava kirjautumisen yhteydessä

LoginTicket tekee multi-tenant-login-flow’sta turvallisen

Access Token varmistaa oikean tenant-kontekstin kaikissa pyynnöissä

Tämä työ auttoi minua ymmärtämään syvällisesti JWT-autentikaation, tenant-eristyksen sekä niiden yhteentoimivuuden modernissa SaaS-ympäristössä.
