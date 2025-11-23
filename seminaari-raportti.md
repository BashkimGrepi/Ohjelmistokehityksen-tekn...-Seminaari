# Seminaarityö: Multi-Tenant–arkkitehtuuri ja JWT-autentikaatio
## Aihe
**Multi-tenant-arkkitehtuurin tietokantarakenne ja sen vaikutus JWT-autentikaatioon**

**Tavoite**

Tavoitteena on selvittää, miten moniasiakasympäristön (multi-tenant) tietokantarakenne muodostetaan ja miten se vaikuttaa JWT-autentikaatioon.
Lopuksi toteutan pienen testin, joka osoittaa autentikoinnin ja eristyksen toimivuuden eri tenanttien välillä — myös skenaarion, jossa käyttäjä kuuluu useampaan tenanttiin.

### Tutkimuskysymykset?

Miten multi-tenant-tietokanta eroaa single-tenant-ratkaisusta?
Miten tenantin tunnistus toteutetaan JWT-autentikaatiossa?
Miten eri tenantit eristetään tietoturvallisesti?
Miten toteutetaan testaus, jossa käyttäjä kuuluu useampaan tenanttiin?

### Johdanto ja tavoite

Tässä seminaarityössä tutkin, miten moniasiakasarkkitehtuuri (multi-tenancy) toteutetaan ja miten se vaikuttaa autentikaatioon.
Käytän esimerkkinä omaa SaaS-sovellustani, jossa toteutan JWT-pohjaisen autentikoinnin ja tenant-kohtaisen tietokantaeristyksen.
Lopuksi rakennan yksinkertaiset testit, jotka todentavat ratkaisun toimivuuden.
Aihe on ollut minulle teknisesti haastava ja ajankohtainen SaaS-kehityksen näkökulmasta.

### Teoreettinen tausta

Multi-tenancy tarkoittaa ohjelmistoarkkitehtuuria, jossa useampi asiakasorganisaatio käyttää samaa järjestelmää, mutta heidän datansa on eristetty. Tämä voidaan toteuttaa usealla eri tavalla.

**_Multi-tenancy-mallit_**
**1. Shared Database, Shared Schema**
Kaikki tenantit käyttävät samoja tauluja.
Eristys tehdään tenant_id-sarakkeen avulla.
Nopein toteuttaa, mutta suurin tietoturvariski, jos eristys pettää.

**2. Shared Database, Separate Schemas**
Yhteinen tietokanta, mutta jokaisella tenantilla oma schema.
Parempi eristys kuin shared schema -mallissa.
Hallinnointi ja migraatiot monimutkaisempia.

**3. Separate Database per Tenant**
Jokaisella tenantilla oma tietokanta.
Paras mahdollinen eristys ja tietoturva.
Kallein ja raskain ylläpitää.
Minun työssäni toteutinn mallin "Shared Database & Shared Schema".

### Päätökset ja hyväksymiskriteerit
**Valitut teknologiat**- 

- NestJS
- Prisma ORM
- PostgreSQL
- JWT-autentikaatio (shared schema)

**Hyväksymiskriteerit**

- JWT sisältää aina tenantId:n.
- Käyttäjä, jolla on useampi jäsenyys, valitsee aktiivisen tenantin.
- Endpointit palauttavat vain sen tenantin dataa, johon käyttäjä on kirjautunut.
- Testit todentavat tenant-eristyksen ja monitenanttijäsenyyden.

### Datan malli: käyttäjä useammassa tenantissa

Multi-tenant-järjestelmä vaatii kaksi erillistä datamallin näkökulmaa:

**1. Direct Tenant Ownership**
- Suora viittaus tenantId.
- Käytetään entiteeteissä, jotka kuuluvat nimenomaisesti yhdelle yritykselle (esim. Ride, Report, Driver).
- onDelete: Cascade poistaa kaiken tenanttiin kuuluvan datan, jos tenant poistetaan.

<img width="569" height="393" alt="Näyttökuva 2025-11-17 162600" src="https://github.com/user-attachments/assets/b6cfae5a-f379-4911-ae09-4f70049b5fc5" />


 **2. User-Centric Membership -malli**

- Mahdollistaa käyttäjän kuulumisen useampaan yritykseen.
Keskeisiä ominaisuuksia:
- Many-to-Many-suhde: User ↔ Tenant
- Membership-taulu määrittää:
  - mihin tenantteihin käyttäjä kuuluu
  - mikä hänen roolinsa on kussakin tenantissa
  - Mahdollistaa roolipohjaisen ja tenant-pohjaisen oikeuksien hallinnan.

<img width="923" height="436" alt="Näyttökuva 2025-11-17 164107" src="https://github.com/user-attachments/assets/2877493d-bcd1-4b67-a79b-5f0dd8135d9c" />


## Autentikaatiovirta ja tenantin valinta
**_Taustaa_**

JWT oli minulle aluksi uusi ja haastava konsepti. Opin sen ensin Spring Bootilla ja toteutin myöhemmin uuden version NestJS:llä.
Multi-tenant-ympäristössä JWT ei voi olla pelkkä “kirjautumistunniste”, vaan sen on:
- tunnistettava käyttäjän tenant
- estettävä käyttöoikeudet muihin tenanteihin
- estettävä datan vuotamisen väärälle organisaatiolle
- käsiteltävä tilanteet, joissa käyttäjä kuuluu useampaan tenanttiin

### JWT-autentikoinnin kriittiset huomioitavat asiat
**_1. Tenantin tunnistaminen_**

Single-tenant-sovelluksessa JWT voi sisältää vain **sub, email, role**.
Multi-tenant-mallissa tämä ei riitä. JWT:n payloadiin täytyy lisätä **tenantId**. 
TenantId:n avulla backend pystyy käsittelemään jokaisen pyynnön oikeassa kontekstissa.
_Esimerkki payloadista (demokäyttäjä):_

<img width="757" height="429" alt="Näyttökuva 2025-11-19 134512" src="https://github.com/user-attachments/assets/449191ca-a3db-4623-8681-9a343446cb6b" />



**_2. Datan eristäminen kyselytasolla_**
Shared schema -mallissa kaikki tenantit jakavat samat taulut, joten eristys on suoritettava ohjelmallisesti.

**Käytäntö:**
Jokainen kysely kirjoitetaan muodossa: _WHERE tenantId = :tokenTenantId_

**Johtopäätös:**
tenantId luetaan aina tokenista, ei request-bodysta. Tietoturvavuoto syntyy,
jos yksikin endpoint unohtaa suodattaa datan tenantId:n perusteella.

_Esimerkki tenant-eristyksestä_

<img width="844" height="36" alt="Näyttökuva 2025-11-19 142633" src="https://github.com/user-attachments/assets/42a4e60d-d5f2-4d86-bdbd-bcf8a899ec87" />


**_3. Käyttäjä useassa tenantissa – aktiivinen tenant JWT:ssä_**

Nykyisissä SaaS-järjestelmissä on tavallista, että käyttäjä on jäsenenä useammassa yrityksessä (tenantissa).
Esimerkiksi konsultti voi toimia usean eri asiakkaan ympäristössä. Tällöin autentikointi ei voi olettaa, että käyttäjällä
on vain yksi tenantId - _Tämän suunnittelu aiheutti minulle haasteita._

- Miten käyttäjä valitsee aktiivisen tenantin?
- Milloin tämä valinta tehdään — ennen vai jälkeen kirjautumisen?
  
Ensimmäinen ideani (hylätty):

- Jos käyttäjällä on vain yksi jäsenyys (tenantId), tilanne on yksinkertainen:
  se laitetaan suoraan tokeniin
  Jos jäsenyyksiä on usemapi, mitä pitää tehdä?
  -> Alussa suunnittelin, että kun admin luo kuskin portaalissaan lisäämällä tiedot kuten s-posti,
    kuskin käyttäjänimen perään luodaan kyseisen tenantkohtaisen
    mukainen arvo (tenant-suffix). Eli jos kyseessä on "TXHELLS",
    niin kuskin käyttäjänimestä tulee email + tenant-suffix esim. "John@gmail.com|TXHELLS4168551...". Tämä arvo tallennettaisiin tietokantaan username kohtaan.
    Tällä tavalla käyttäjällä voisi olla sama s-posti eri yrityksissä.
    **mutta**
    _Tässä minulla tuli ongelmia vastaaan_
  - Kirjautumislogiikka erittäin monimutkaista -> täytyy parsia string arvoja ja verrata prefixejä
  - Jos tenant-suffix-formaatti muuttuu niin täytyy myös muuttaa sen tenantin kaikkien käyttäjien käyttäjänimet
  - Näitten lisäksi täytyy kuitenkin käsitellä tilannetta, jossa samalla emailillä löytyy useampi tenant.
    
  - Osoittautui erittäin huonoksi: altis virheille, ei skaalautuva, vaikea ylläpitää.

**_Lopullinen ja toimiva toteutus_**
**Vaihe 1 — Login**
-> 
Käyttäjä tekee **POST /auth/login**, jonka jälkeen **Backend** tarkistaa _(validoi)_ käyttäjän
_Tärkeää_ -> Jos käyttäjällä on useampi **membership** → ei luoda vielä **AccessTokenia**
<img width="663" height="449" alt="Näyttökuva 2025-11-19 150843" src="https://github.com/user-attachments/assets/39acfc03-0046-4300-aacc-6615a12ecb52" />
<img width="724" height="510" alt="Näyttökuva 2025-11-19 151251" src="https://github.com/user-attachments/assets/08d7f200-9352-4321-bb3e-162b0ebb40d7" />

**Vaihe 2 — LoginTicket**
-> 
Backend luo väliaikaisen, hyvin lyhytkestoisen JWT-ticketin: sisältää listan käyttäjän tenant-jäsenyyksistä, sekä itse (exp: 5 minuuttia) **_loginTicket_**:in,
ei sisällä arkaluontoista tietoa .Tätä tickettiä ei voi käyttää muihin API-kutsuihin _(audience: tenant-selection)_

Kuva →<img width="783" height="211" alt="Näyttökuva 2025-11-19 151516" src="https://github.com/user-attachments/assets/5181ec09-5e38-4c5c-a5ff-f164b8a19834" />


**_Vaihe 3 — Tenant valitaan_**

Frontend näyttää listan tenanteista, josta käyttäjä valitsee tenantin ---> **POST /auth/select-tenant**
<img width="425" height="134" alt="Näyttökuva 2025-11-19 151559" src="https://github.com/user-attachments/assets/b2614105-a216-44ae-ae75-2c34316c5456" />

**_Vaihe 4 — Access Token_**

Backend verifikoi LoginTicketin, tarkistaa, että käyttäjällä on oikeus valittuun tenanttiin, jonka jälkeen
luo lopullisen **_Access Tokenin_**, jossa:

<img width="781" height="752" alt="Näyttökuva 2025-11-19 152011" src="https://github.com/user-attachments/assets/e63186d7-744b-4b04-85f4-9d9640b31b95" />


**_4. LoginTicket – väliaikainen JWT-**

Hyödyt:
- Ei voi käyttää muihin API-päätepisteisiin
- Ei sisällä henkilökohtaisia tietoja
- Estää tenantin vaihtamisen ilman uutta autentikointia
- Tekee multi-tenant-login-flow’sta selkeän ja turvallisen

<img width="324" height="180" alt="Näyttökuva 2025-11-19 152508" src="https://github.com/user-attachments/assets/a9114ecd-a019-4b4d-9c18-20f04e3baf77" />


**_5. Access Token:**
Kun tenant on valittu luodaan varsinainen pääsytoken.
Kaikki API-kutsut tapahtuvat tässä tenant-kontekstissa
Roolit (ADMIN/DRIVER) määräytyvät membershipin mukaan

<img width="727" height="378" alt="Näyttökuva 2025-11-19 152540" src="https://github.com/user-attachments/assets/72e4143a-95fe-4e9f-8f27-a2f8d5fb7729" />


Yhteenveto

Multi-tenant shared schema -arkkitehtuuri yhdistettynä JWT-autentikaatioon vaatii tarkkaa ja harkittua toteutusta.

Keskeiset havainnot:
- TenantId tulee tallentaa JWT-tokeniin.
- Kaikki tietokantakyselyt suodatetaan tokenin tenantId:n perusteella.
- Käyttäjä voi kuulua useaan tenanttiin → aktiivinen tenant täytyy valita kirjautumisen yhteydessä.
- LoginTicket antaa turvallisen tavan toteuttaa monivaiheinen kirjautuminen.
- Access Token varmistaa oikean tenant-kontekstin kaikissa pyynnöissä.
- Tenant-eristys on ratkaisevan tärkeä osa SaaS-järjestelmän tietoturvaa.

  
