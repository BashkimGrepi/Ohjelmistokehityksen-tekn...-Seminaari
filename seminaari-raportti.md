Seminaarityö: Multi-Tenant–arkkitehtuuri ja JWT-autentikaatio
Aihe

Multi-tenant-arkkitehtuurin tietokantarakenne ja sen vaikutus JWT-autentikaatioon

Tavoite

Tavoitteena on selvittää, miten moniasiakasympäristön (multi-tenant) tietokantarakenne muodostetaan ja miten se vaikuttaa JWT-autentikaatioon.
Lopuksi toteutan pienen testin, joka osoittaa autentikoinnin ja eristyksen toimivuuden eri tenanttien välillä — myös skenaarion, jossa käyttäjä kuuluu useampaan tenanttiin.

Tutkimuskysymykset

Miten multi-tenant-tietokanta eroaa single-tenant-ratkaisusta?

Miten tenantin tunnistus toteutetaan JWT-autentikaatiossa?

Miten eri tenantit eristetään tietoturvallisesti?

Miten toteutetaan testaus, jossa käyttäjä kuuluu useampaan tenanttiin?

Johdanto ja tavoite

Tässä seminaarityössä tutkin, miten moniasiakasarkkitehtuuri (multi-tenancy) toteutetaan ja miten se vaikuttaa autentikaatioon.
Käytän esimerkkinä omaa SaaS-sovellustani, jossa toteutan JWT-pohjaisen autentikoinnin ja tenant-kohtaisen tietokantaeristyksen.
Lopuksi rakennan yksinkertaiset testit, jotka todentavat ratkaisun toimivuuden.
Aihe on ollut minulle teknisesti haastava ja ajankohtainen SaaS-kehityksen näkökulmasta.

Teoreettinen tausta

Multi-tenancy tarkoittaa ohjelmistoarkkitehtuuria, jossa useampi asiakasorganisaatio käyttää samaa järjestelmää, mutta heidän datansa on eristetty. Tämä voidaan toteuttaa usealla eri tavalla.

Multi-tenancy-mallit
1. Shared Database, Shared Schema

Kaikki tenantit käyttävät samoja tauluja.

Eristys tehdään tenant_id-sarakkeen avulla.

Nopein toteuttaa, mutta suurin tietoturvariski, jos eristys pettää.

2. Shared Database, Separate Schemas

Yhteinen tietokanta, mutta jokaisella tenantilla oma schema.

Parempi eristys kuin shared schema -mallissa.

Hallinnointi ja migraatiot monimutkaisempia.

3. Separate Database per Tenant

Jokaisella tenantilla oma tietokanta.

Paras mahdollinen eristys ja tietoturva.

Kallein ja raskain ylläpitää.

Tässä työssä toteutan mallin Shared Database & Shared Schema.

Päätökset ja hyväksymiskriteerit
Valitut teknologiat

NestJS

Prisma ORM

PostgreSQL

JWT-autentikaatio (shared schema)

Hyväksymiskriteerit

JWT sisältää aina tenantId:n.

Käyttäjä, jolla on useampi jäsenyys, valitsee aktiivisen tenantin.

Endpointit palauttavat vain sen tenantin dataa, johon käyttäjä on kirjautunut.

Testit todentavat tenant-eristyksen ja monitenanttijäsenyyden.

Datan malli: käyttäjä useammassa tenantissa

Multi-tenant-järjestelmä vaatii kaksi erillistä datamallin näkökulmaa:

1. Direct Tenant Ownership

Suora viittaus tenantId.

Käytetään entiteeteissä, jotka kuuluvat nimenomaisesti yhdelle yritykselle (esim. Ride, Report, Driver).

onDelete: Cascade poistaa kaiken tenanttiin kuuluvan datan, jos tenant poistetaan.

Kuva →

2. User-Centric Membership -malli

Mahdollistaa käyttäjän kuulumisen useampaan yritykseen.

Keskeisiä ominaisuuksia:

Many-to-Many-suhde: User ↔ Tenant

Membership-taulu määrittää:

mihin tenantteihin käyttäjä kuuluu

mikä hänen roolinsa on kussakin tenantissa

Mahdollistaa roolipohjaisen ja tenant-pohjaisen oikeuksien hallinnan.

Kuva →

Autentikaatiovirta ja tenantin valinta
Taustaa

JWT oli minulle aluksi uusi ja haastava konsepti. Opin sen ensin Spring Bootilla ja toteutin myöhemmin uuden version NestJS:llä.
Multi-tenant-ympäristössä JWT ei voi olla pelkkä “kirjautumistunniste”, vaan sen on:

tunnistettava käyttäjän tenant

estettävä käyttöoikeudet muihin tenanteihin

estettävä datan vuotaminen väärälle organisaatiolle

käsiteltävä tilanteet, joissa käyttäjä kuuluu useampaan tenanttiin

JWT-autentikoinnin kriittiset huomioitavat asiat
1. Tenantin tunnistaminen

Single-tenant-sovelluksessa JWT voi sisältää vain sub, email, role.
Multi-tenant-mallissa tämä ei riitä.

JWT:n payloadiin täytyy lisätä:

tenantId

Esimerkki payloadista (demokäyttäjä):

Kuva →

TenantId:n avulla backend pystyy käsittelemään jokaisen pyynnön oikeassa kontekstissa.

2. Datan eristäminen kyselytasolla

Shared schema -mallissa kaikki tenantit jakavat samat taulut, joten eristys on suoritettava ohjelmallisesti.

Käytäntö:

Jokainen kysely kirjoitetaan muodossa:

WHERE tenantId = :tokenTenantId

Johtopäätös:

tenantId luetaan aina tokenista, ei request-bodysta.

Tietoturvavuoto syntyy, jos yksikin endpoint unohtaa suodattaa datan tenantId:n perusteella.

Esimerkki tenant-eristyksestä

Kuva →

3. Käyttäjä useassa tenantissa – aktiivinen tenant JWT:ssä

Nykyisissä SaaS-järjestelmissä on tavallista, että käyttäjä on jäsenenä useammassa yrityksessä (tenantissa).
Tämä aiheuttaa kaksi ongelmaa:

Miten käyttäjä valitsee aktiivisen tenantin?

Milloin tämä valinta tehdään — ennen vai jälkeen kirjautumisen?

Virheellinen idea (hylätty):

Lisätä tenant-suffix emailiin

john@gmail.com → john@gmail.com#TenantXYZ

Osoittautui erittäin huonoksi: altis virheille, ei skaalautuva, vaikea ylläpitää.

Lopullinen ja toimiva toteutus
Vaihe 1 — Login

Käyttäjä tekee POST /auth/login

Backend tarkistaa käyttäjän

Jos käyttäjällä on useampi membership → ei luoda vielä AccessTokenia

Vaihe 2 — LoginTicket

Backend luo väliaikaisen, hyvin lyhytkestoisen JWT-tokenin:

sisältää listan käyttäjän tenant-jäsenyyksistä

ei sisällä arkaluontoista tietoa

ei voi käyttää API-kutsuihin

exp: 5 minuuttia

Kuva →

Vaihe 3 — Tenant valitaan

Frontend näyttää listan tenanteista

Käyttäjä valitsee tenantin

POST /auth/tenant-selection

Vaihe 4 — Access Token

Backend verifioi LoginTicketin

Tarkistaa, että käyttäjällä on oikeus valittuun tenanttiin

Luo lopullisen Access Tokenin, jossa:

sub
email
tenantId
role


Kuva →

4. LoginTicket – väliaikainen JWT

Hyödyt:

Ei voi käyttää muihin API-päätepisteisiin

Ei sisällä henkilökohtaisia tietoja

Estää tenantin vaihtamisen ilman uutta autentikointia

Tekee multi-tenant-login-flow’sta selkeän ja turvallisen

Kuva →

5. Access Token

Kun tenant on valittu:

Luodaan varsinainen pääsytoken

Kaikki API-kutsut tapahtuvat tässä tenant-kontekstissa

Roolit (ADMIN/USER) määräytyvät membershipin mukaan

Kuva →

Yhteenveto

Multi-tenant shared schema -arkkitehtuuri yhdistettynä JWT-autentikaatioon vaatii tarkkaa ja harkittua toteutusta.

Keskeiset havainnot:

TenantId tulee tallentaa JWT-tokeniin.

Kaikki tietokantakyselyt suodatetaan tokenin tenantId:n perusteella.

Käyttäjä voi kuulua useaan tenanttiin → aktiivinen tenant täytyy valita kirjautumisen yhteydessä.

LoginTicket antaa turvallisen tavan toteuttaa monivaiheinen kirjautuminen.

Access Token varmistaa oikean tenant-kontekstin kaikissa pyynnöissä.

Tenant-eristys on ratkaisevan tärkeä osa SaaS-järjestelmän tietoturvaa.
