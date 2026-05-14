# Laskuvalidointi-automaatio 🧾

Taloushallinnon laskujen automaattinen validointi n8n-alustalla. Järjestelmä hakee laskudata Google Sheetsistä, muuntaa ulkomaan valuutat reaaliaikaisilla kursseilla euroiksi ja ajaa monivaiheisen validoinnin — tulokset tallentuvat automaattisesti raportointitaulukkoon.

---

## Teknologiat

- **n8n** — työnkulun orkestrointi (low-code automaatioalusta)
- **Google Sheets** — laskudata ja raportointihistoria
- **ExchangeRate API** — reaaliaikaiset valuuttakurssit (EUR/USD/GBP/SEK)
- **JavaScript** — validointilogiikka ja datan transformointi

---

## Työnkulku

```
Manual Trigger
      ↓
Google Sheets — hakee laskurivit
      ↓
ExchangeRate API — hakee reaaliaikaiset valuuttakurssit
      ↓
JavaScript (Code node) — validointi + aggregointi
      ↓
Google Sheets — tallentaa raportin historiataulukkoon
```

---

## Validointilogiikka

Järjestelmä tarkistaa jokaisen laskurivin kolmella säännöllä:

### 1. ALV-prosentin validointi
Tarkistaa että käytetty ALV-kanta on Suomen voimassaolevan lainsäädännön mukainen (2026):
- 25,5 % — yleinen verokanta
- 13,5 % — alennettu (ruoka, ravintolat, lääkkeet)
- 10 % — alennettu (sanoma- ja aikakauslehdet)
- 0 % — verovapaat

Vanhentunut 24 % tai 14 % laukaisee virheen automaattisesti.

### 2. ALV-summan täsmäytys
Laskee ALV-summan nettosummasta ja vertaa ilmoitettuun. Yli 0,05 € ero laukaisee virheen. Ulkomaan valuutat muunnetaan ensin euroiksi reaaliaikaisella kurssilla ennen laskentaa.

### 3. Duplikaattilaskujen tunnistus
Vertaa jokaista laskua aiempiin saman toimittajan laskuihin. Jos sama toimittaja + sama summa (±1 % toleranssi) löytyy aiemmasta rivistä, järjestelmä merkitsee rivin mahdolliseksi duplikaatiksi ja raportoi päivien eron.

---

## Valuuttamuunnos

Reaaliaikaiset kurssit haetaan ExchangeRate API:sta ennen jokaista validointiajoa. Käytetyt kurssit ja hakuaika tallennetaan raporttiin jäljitettävyyden varmistamiseksi — tämä on tärkeää koska kurssit vaihtelevat päivittäin ja voivat vaikuttaa ALV-summien täsmäävyyteen.

---

## Raportointihistoria (Google Sheets)

Jokaisesta ajokerrasta tallennetaan automaattisesti:

| Sarake | Esimerkki |
|---|---|
| Ajettu | 2026-05-13T17:12:01Z |
| Tarkistettu riviä | 30 |
| Valuuttakurssit | USD=0.8519 GBP=1.1527 SEK=0.0917 |
| Virheiden määrä | 4 |
| Virheet | Rivi 17 (TechCorp Ltd): Virheellinen ALV-% 24 \| ... |

---

## Esimerkkiraportti

Alla esimerkki löydetyistä virheistä 30 laskurivin testidatasta:

```
✅ Tarkistettu: 30 riviä
⚠️ Virheitä löydetty: 4

• Rivi 12 (Acme Oy): Mahdollinen duplikaatti — sama summa 1500.00€ 10 päivän välillä
• Rivi 17 (TechCorp Ltd): Virheellinen ALV-% 24 (voimassa oleva yleinen kanta on 25,5%)
• Rivi 23 (Kuljetusliike Ky): ALV-summa ei täsmää — laskettu 688.50€, ilmoitettu 689.85€
• Rivi 29 (Acme Oy): Mahdollinen duplikaatti — sama summa 1500.00€ 27 päivän välillä
```

---

## Jatkokehitysideat

### Toimialakohtainen ALV-validointi 🏗️

Nykyinen järjestelmä tarkistaa vain että ALV-prosentti on teknisesti sallittu — mutta ei osaa arvioida onko se *oikea* kyseiselle toimittajalle tai palvelutyypille.

**Esimerkki:** Konsulttipalvelut kuuluvat aina yleisen 25,5 % verokannan piiriin. Jos konsulttilasku saapuu 13,5 % ALV:lla, se on teknisesti sallittu kanta mutta todennäköisesti virheellinen.

**Toteutusidea:** Lisätään laskudataan sarake `toimiala` tai tunnistetaan toimittajatyyppi nimen perusteella. Koodiin lisätään toimiala–ALV-sääntötaulukko:

```javascript
const alvSaannot = {
  'konsultointi':   [25.5],
  'ravintola':      [13.5],
  'kuljetus':       [25.5],
  'elintarvike':    [13.5],
  'lehti':          [10],
};

const toimiala = row['Toimiala']?.toLowerCase();
const sallitutKannat = alvSaannot[toimiala];

if (sallitutKannat && !sallitutKannat.includes(alvRate)) {
  errors.push(
    `Rivi ${i+2} (${row['Toimittaja']}): Epätodennäköinen ALV-kanta — ` +
    `${toimiala} käyttää tyypillisesti ${sallitutKannat.join('/')}%, ` +
    `ilmoitettu ${alvRate}%`
  );
}
```

Tämä muuttaisi järjestelmän reaktiivisesta (löytää matemaattiset virheet) proaktiiviseksi (tunnistaa liiketoiminnallisesti epätyypilliset tilanteet ennen kirjanpidon sulkemista).

### Käännetyn verovelvollisuuden tunnistus 🌍

Nykyinen järjestelmä validoi ALV-kannat Suomen lainsäädännön mukaan — mutta ulkomaisilta toimittajilta tulevissa laskuissa ei tyypillisesti ole Suomen ALV:a lainkaan. EU:n välisessä B2B-kaupassa sovelletaan käännettyä verovelvollisuutta: ulkomainen toimittaja laskuttaa 0 % ALV:lla ja suomalainen yritys laskee ja tilittää veron itse omassa kirjanpidossaan.

**Toteutusidea:** Tunnistetaan ulkomaan valuutta merkkinä mahdollisesta käännetystä verovelvollisuudesta:

```javascript
if (currency !== 'EUR' && alvRate !== 0) {
  errors.push(
    `Rivi ${i+2} (${row['Toimittaja']}): Ulkomaisella laskulla ` +
    `(${currency}) ALV tulisi tyypillisesti olla 0% — ` +
    `tarkista käännetty verovelvollisuus`
  );
}
```

Huom: valuutta ei yksinään riitä tunnistukseen — esim. ruotsalainen toimittaja voi laskuttaa euroissa. Tarkempi toteutus vaatisi toimittajan maa-tiedon laskudataan.

### Muita jatkokehitysideoita

- **Laskunumeron duplikaattitarkistus** — täsmällisempi kuin summa+toimittaja -yhdistelmä
- **Automaattinen ajastus** — Cron trigger päivittäiseen tai viikoittaiseen ajoon manuaalisen sijaan
- **Sähköpostihälytys** — virheraportti suoraan kirjanpitäjän sähköpostiin
- **Netvisor-integraatio** — datan haku suoraan kirjanpitojärjestelmästä Google Sheetsin sijaan

---

## Toteutus

Projekti toteutettiin alle kahdessa tunnissa sisältäen:
- Testidatan suunnittelu ja virheiden piilottaminen
- n8n-työnkulun rakentaminen (Google Sheets + HTTP Request + Code + Sheets raportti)
- JavaScript-validointilogiikan kehitys ja debuggaus
- Valuuttamuunnoksen integrointi reaaliaikaiseen API:in
- Raportointihistorian rakentaminen

---

## Tekijä Mira Vallenius

Rakennettu portfolioprojektina osoittamaan taloushallinnon automaatio-osaamista:
- Low-code automaatioalustan (n8n) käyttö
- REST API -integraatiot
- JavaScript-validointilogiikka
- Taloushallinnon prosessiymmärrys 
