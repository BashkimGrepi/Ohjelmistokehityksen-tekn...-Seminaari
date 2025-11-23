# 🧩 Multi-Tenant Authentication with JWT – Seminaarityö

## 📘 Johdanto
Tässä seminaarityössä tarkastellaan, miten **multi-tenant-arkkitehtuuri** vaikuttaa sovelluksen tietokantarakenteeseen ja käyttäjien **JWT-autentikaatioon**.  
Aihe valittiin, koska moniasiakasympäristö (multi-tenant) on yleinen lähestymistapa SaaS-sovelluksissa, mutta sen jaon toteuttaminen tietoturvallisesti autentikointitasolla voi olla haastavaa.

Tavoitteena on selvittää ja demonstroida:
- miten tenant-kohtainen tietokanta-rakenne muodostetaan,
- miten tenant-tunnus (tenantId) kulkee JWT-tokenin mukana,
- miten tenantit eristetään ohjelmallisesti,
- ja miten tätä voidaan testata automaattisesti.

---

## 🎯 Tavoitteet
1. Toteuttaa yksinkertainen multi-tenant-rakenne PostgreSQL-tietokannalla (Prisma ORM).  
2. Toteuttaa JWT-autentikaatio, joka sisältää `tenantId`-tiedon.  
3. Varmistaa, että kirjautunut käyttäjä pääsee vain oman tenanttinsa dataan.  
4. Luoda perus testit, jotka validoivat eristyksen toimivuuden.  
5. Raportoida opitut asiat ja jatkokehitysmahdollisuudet.

---

## 🏗️ Teknologiat
| Teknologia | Käyttötarkoitus |
|-------------|----------------|
| **NestJS** | Backend-rakenne ja autentikaatio |
| **Prisma ORM** | Tietokantamallinnus ja kyselyt |
| **PostgreSQL** | Relaatiotietokanta |
| **JWT (jsonwebtoken)** | Käyttäjän autentikaatio ja tenantId:n välitys |
| **Jest / Supertest** | Testaaminen |
| **TypeScript** | Tyypitetty kehitysympäristö |

---
