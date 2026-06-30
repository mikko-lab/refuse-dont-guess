# Miten "Refuse, don't guess" rakennettiin
### AI-native PDLC työnäytteenä — discoverystä ylläpitoon, governance edellä

Tämä dokumentti ei kuvaa pelkkää koodin tuottamista. Se on **työnäyte tavasta tehdä AI-avusteista tuotekehitystä**: miten ihminen ja AI-työkalu (Claude Code) jakavat työn niin, että lopputulos on luotettava, todennettava ja vastuullinen. Lopussa sama prosessi on tiivistetty **uudelleenkäytettäväksi malliksi**, jonka voi soveltaa mihin tahansa "LLM kriittisellä polulla" -tapaukseen.

> Konkreettinen esimerkki, ei diaesitys: jokainen vaihe alla tuotti oikean artefaktin (`README.md`, `guardrail.py`), joka ajautuu ja todistaa kriteerinsä.

---

## AI-native PDLC, vaihe vaiheelta

Jokaisessa vaiheessa kysymys on: **mitä ihminen omistaa, mitä AI tekee, ja miten tuotos validoidaan.**

### 1. Discovery — *ihminen omistaa ongelman kehystyksen*

Ongelma rajattiin ennen kuin riviäkään kirjoitettiin: kuka tekee työn, mikä on kipu, mikä on kriittistä. AI toimi sparrauskumppanina, mutta ratkaisevat valinnat — "onko tämä oikea asiakas", "onko tämä oikea pihvi", "mikä rajataan ulos" — olivat ihmisen.

**Periaate:** AI ei saa keksiä ongelmaa. Ihminen tuo liiketoimintakontekstin; AI auttaa jäsentämään sen.

### 2. Määrittely — *hyväksymiskriteerit ennen koodia*

Kriteerit lukittiin etukäteen: determinismi, epävarma → eskaloi, injektio → torju. "Refuse, don't guess" oli **hyväksymiskriteeri**, ei jälkikäteen keksitty ominaisuus. AI luonnosteli speksiä; ihminen asetti riman ja mitattavat ehdot.

**Periaate:** rima määritellään ennen toteutusta, jotta tuotosta voi arvioida — eikä toteutus määrittele rimaa jälkikäteen.

### 3. Toteutus — *AI tuottaa, ihminen rajaa*

Claude Code tuotti turvakerroksen. Ihmisen rooli oli **ohjata ja rajata**: ei riippuvuuksia, LLM-poiminta stubatuksi (jotta turvakerros on testattava), deterministinen sääntö erotettuna mallin tuotoksesta. Rajaukset olivat tietoisia arkkitehtuuripäätöksiä, eivät oletuksia.

**Periaate:** AI:n tuottavuus on arvokas vasta kun se on kytketty ihmisen asettamiin rajoihin.

### 4. Validointi — *käytöksen tasolla, ei rivi riviltä*

Tuotos validoitiin **hyväksymiskriteerejä vasten**: kolme casea (PASS / ESCALATE / BLOCK) ja determinismitarkistus (jokainen case 1000 ajoa, päätöksen oltava aina sama). Tämä on validointia käytöksen tasolla — todistetaan että systeemi tekee oikean asian ja kieltäytyy väärästä, ei lueta jokaista riviä erikseen.

**Periaate (rehellisesti liputettuna):** kriittisellä polulla oikea validoinnin yksikkö on *havaittava käytös*, ei rivimäärä. Tämä on tietoinen valinta, ei oikomista.

### 5. Ylläpito ja governance — *sääntö on asiantuntijan, ei agentin*

Kriittinen sääntö (tässä: ALV-kohtelu) on **domain-asiantuntijan omistama ja versioitu** — agentti vain panee sen toimeen ja eskaloi epävarman. Vastuullinen AI ei ole tarkistuslista lopussa, vaan arkkitehtuuriin sisäänrakennettu: malli ei koskaan päätä kriittistä asiaa yksin.

**Periaate:** governance kuuluu rakenteeseen, ei jälkitarkistukseen.

---

## Katselmointi käytännössä

Yllä oleva ei ole julistus — tässä se sama silmukka oikeasti ajettuna. Valmis koodi annettiin **kriittiseen katselmointiin** (tietoturva + oikeellisuus). Se ei etsinyt tyylivirheitä vaan kysyi: *missä tämä menee hiljaa väärin?* Löydökset asetettiin vakavuusjärjestykseen ja korjattiin — jokainen verifioituna, ei kuitattuna.

Mitä katselmointi nosti esiin (7 löydöstä, tärkein ensin):

→ **Rivinvaihto kiersi injektiosuojan** — kuvio ei ylittänyt rivinvaihtoa, joten BLOCK-lupaus oli todellisuudessa heikompi kuin väitetty. Tämä on tasan se "hiljainen väärin" jota haettiin.
→ **Unicode-normalisointi puuttui** — hajotetut merkit (NFD) olisivat kiertäneet suomenkieliset kuviot.
→ **Osunut kuvio vuoti ulospäin** — olisi auttanut hyökkääjää löytämään kiertotavan.
→ **Luottamusarvoa ei rajattu** välille [0,1]; **audit-jälki katkesi** kun eskaloinnilla oli kaksi syytä; **determinismitesti katsoi vain päätöstä**, ei koko tulosta; **hylkäyksen peruste jäi näkymättömiin**.

Korjausten jälkeen lisättiin **kaksi uutta regressiotestiä** (rivinvaihto- ja NFD-kierto) ja determinismitarkistus laajennettiin koko tulokseen. Ajo vahvistaa kaiken vihreäksi.

**Olennaista menetelmän kannalta:** löydökset suljettiin *verifioimalla käytös* — uusi testi joka todistaa että kierto torjutaan — ei lukemalla rivejä uudelleen ja toteamalla "näyttää korjatulta". Juuri tämä on validointi käytöksen tasolla. Ja yhtä tärkeää: katselmointi johti myös **rehelliseen rajaukseen** — input-skannaus merkittiin puolustuksen kerrokseksi, ei täydelliseksi injektiosuojaksi. Korjaus ei saanut synnyttää ylilupausta.

**Periaate:** kriittinen katselmointi on osa putkea, ei valinnainen lopputarkistus — ja sen tulos verifioidaan, ei uskota.

---

## Uudelleenkäytettävä malli: "LLM kriittisellä polulla"

Tiivistys yllä olevasta — sovellettavissa mihin tahansa tapaukseen jossa LLM koskee taloudellisesti tai juridisesti kriittistä dataa:

1. **Kehystä tarve ihmisvetoisesti.** Tunnista kriittinen polku: missä hiljainen virhe maksaa.
2. **Lukitse hyväksymiskriteerit ensin.** Vähintään: determinismi, epävarman eskalointi, injektion torjunta.
3. **Erota roolit:** LLM poimii faktat sotkusta; deterministinen sääntö päättää; epävarma menee ihmiselle.
4. **Validoi käytöksellä.** Golden-caset + determinismitarkistus, ei fiilistä.
5. **Anna säännön omistus asiantuntijalle.** Versioi se. Agentti panee toimeen, ei keksi.
6. **Liputa rajaukset.** Kirjaa mitä jätettiin ulos ja miksi.

Tämä on se artefaktityyppi jota AI-native-tuotekehitystiimi tuottaa: ei yksittäinen feature, vaan **toimintamalli jonka muut voivat ottaa käyttöön.**

---

## Mitä tämä näyttää — ja mitä ei (liputa, älä piilota)

**Näyttää:** miten AI-avusteinen kehitys jaetaan ihmisen ja työkalun kesken niin että lopputulos on luotettava; miten governance rakennetaan sisään; miten tuotos validoidaan systemaattisesti; miten käytäntö tiivistetään uudelleenkäytettäväksi malliksi.

**Ei näytä:** mallin jalkauttamista koko organisaatioon eikä muutosjohtamista monen tiimin yli. Se on erikseen keskusteltava — tämä työnäyte todistaa menetelmän, ei organisaatiomuutosta.
