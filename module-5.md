---
module: 5
order: 1
moduleTitle: "5: Integratie"
published: true
---

# Module 5: Integratie

Een SAP-systeem staat zelden alleen. Webshops, apps en andere systemen moeten er data uit lezen en er data in schrijven. In deze module spreek je een business object aan vanuit een applicatie buiten SAP. Je verkent ook hoe SAP zijn eigen business objects als API aanbiedt.

### Leerdoelen

Na deze module kan je:

- de metadata en de URL-structuur van een OData V4-service lezen;
- data opvragen met query options zoals `$filter`, `$select` en `$expand`;
- data wijzigen via OData, inclusief CSRF-tokens, ETags en foutmeldingen;
- een externe client bouwen die een SAP-service gebruikt;
- een API van SAP S/4HANA opzoeken en testen in de SAP Business Accelerator Hub;
- verantwoorden welke integratieaanpak bij een scenario past.

## 5.1 OData V4: de taal van SAP-services

De Fiori-app uit module 4 haalt haar data ergens vandaan. Open de ontwikkelaarstools van je browser, tabblad **Network**, en vernieuw de preview. Je ziet een reeks aanvragen naar URL's die eindigen op `Promotion?$count=true&$select=…`. Wat is dat voor taal?

**OData** (Open Data Protocol) is een standaard om data aan te bieden via HTTP. Het lijkt op een REST-API die je in vorige vakken bouwde, maar met vaste afspraken: hoe URL's eruitzien, hoe je filtert, hoe een fout eruitziet. SAP gebruikt OData voor bijna al zijn services. De huidige versie is **OData V4**; je komt in oudere services nog V2 tegen.

Een OData-service heeft een basis-URL. Die vind je in de service binding: klik op de entiteit en kopieer de URL bij **Service URL**. Ze ziet er ongeveer zo uit:

```text
http://sap-lab.ehb.be:50000/sap/opu/odata4/sap/zsi00_ui_promotion_o4/srvd/sap/zsi00_ui_promotion/0001/
```

[CONTROLEREN] Hostnaam en poort van het labosysteem.

Op die basis bouw je verder:

| URL | Resultaat |
| --- | --- |
| `…/0001/$metadata` | De beschrijving van de service: entiteiten, elements, types, actions |
| `…/0001/Promotion` | Alle promoties, als JSON |
| `…/0001/Promotion(PromoUuid=…,IsActiveEntity=true)` | Eén promotie |
| `…/0001/Promotion(…)/_Airlines` | De airlines van die promotie, via de composition |

Een **entity set** zoals `Promotion` komt overeen met een `expose` in je service definition ([4.9](./module-4.md#4_9)). Het antwoord is JSON:

```json
{
  "@odata.context": "$metadata#Promotion",
  "value": [
    {
      "PromoUuid": "a3f1…",
      "PromoId": "00000007",
      "AgencyId": "070001",
      "Description": "Lentekorting",
      "ValidFrom": "2027-03-01",
      "ValidTo": "2027-03-31",
      "DiscountPct": 15,
      "Status": "A",
      "IsActiveEntity": true
    }
  ]
}
```

Het veld `IsActiveEntity` komt van draft ([4.8](./module-4.md#4_8)): de UI-service toont zowel concepten als bewaarde promoties. Een externe client wil meestal alleen de bewaarde. Daarom maken we in 5.4 een aparte service zonder draft.

Open `$metadata` in je browser. Het is lang en in XML, maar het beantwoordt elke vraag over de service: de velden, hun types en de actions die je kan aanroepen. Een goede OData-client leest dat document; een goede ontwikkelaar ook.

## 5.2 Query options: precies de data die je nodig hebt

De website van een agentschap moet alleen zijn eigen actieve promoties tonen. Haal je alle promoties op en filter je in JavaScript? Bij tien promoties werkt dat, bij een miljoen niet.

OData heeft **system query options**: parameters in de URL die beginnen met `$`. De server voert ze uit, zodat alleen de data die je nodig hebt over het netwerk gaat.

| Query option | Betekenis | Voorbeeld |
| --- | --- | --- |
| `$filter` | Rijen selecteren | `$filter=Status eq 'A'` |
| `$select` | Elements selecteren | `$select=PromoId,DiscountPct` |
| `$orderby` | Sorteren | `$orderby=ValidTo desc` |
| `$top`, `$skip` | Pagineren | `$top=20&$skip=40` |
| `$count` | Het aantal rijen meegeven | `$count=true` |
| `$expand` | Een association meenemen | `$expand=_Airlines` |

Je combineert ze met `&`:

```text
…/0001/Promotion?$filter=AgencyId eq '070001' and Status eq 'A'&$select=PromoId,Description,DiscountPct,ValidTo&$orderby=ValidTo
```

In `$filter` gebruik je operatoren als woorden: `eq` (gelijk), `ne` (niet gelijk), `gt` en `lt` (groter en kleiner), `and`, `or`. Tekst staat tussen enkele aanhalingstekens, een datum niet: `ValidTo ge 2027-03-01`.

Met `$expand` krijg je in één aanvraag ook de airlines mee:

```text
…/0001/Promotion?$select=PromoId&$expand=_Airlines($select=AirlineId,AirlineName)
```

```json
{
  "value": [
    {
      "PromoId": "00000007",
      "_Airlines": [
        { "AirlineId": "LH", "AirlineName": "Lufthansa" },
        { "AirlineId": "SQ", "AirlineName": "Singapore Airlines Limited" }
      ]
    }
  ]
}
```

Een composition `[0..*]` geeft een lijst; een association `[1..1]` zou één object geven.

::: warning
**Let op:** in een URL moeten spaties en aanhalingstekens gecodeerd worden. Je browser doet dat voor jou; in code gebruik je `encodeURIComponent` in JavaScript of `rawurlencode` in PHP. Een `$filter` die in de browser werkt maar in je code niet, is bijna altijd een coderingsprobleem.
:::

## 5.3 Data wijzigen via OData: CSRF-tokens, ETags en foutmeldingen

Lezen gaat met een `GET`. Voor schrijven gebruik je de andere HTTP-methods, zoals je ze kent uit een REST-API.

| Bewerking | HTTP-method | URL |
| --- | --- | --- |
| Aanmaken | `POST` | `…/Promotion` |
| Wijzigen | `PATCH` | `…/Promotion(<sleutel>)` |
| Verwijderen | `DELETE` | `…/Promotion(<sleutel>)` |
| Action uitvoeren | `POST` | `…/Promotion(<sleutel>)/<namespace>.startPromotion` |

Drie dingen maken schrijven in SAP lastiger dan lezen.

**CSRF-token.** SAP beschermt elke wijziging tegen **cross-site request forgery** (CSRF): een andere website die in jouw naam een aanvraag stuurt. Voor je iets wijzigt, vraag je een token op met een `GET` en de header `X-CSRF-Token: Fetch`.

Het antwoord bevat het token in dezelfde header, en sessiecookies. Bij de wijziging stuur je token en cookies mee.

```http
GET …/0001/ HTTP/1.1
Authorization: Basic …
X-CSRF-Token: Fetch
```

```http
POST …/0001/Promotion HTTP/1.1
Authorization: Basic …
X-CSRF-Token: 4kq9Z…
Cookie: SAP_SESSIONID_A4H_001=…
Content-Type: application/json

{ "AgencyId": "070001", "DiscountPct": 15, "ValidFrom": "2027-06-01" }
```

**ETag.** Bij een `PATCH` of `DELETE` stuur je de header `If-Match` met de **ETag**: een versiecode die je bij het lezen kreeg. Wijzigde iemand anders de promotie intussen, dan klopt de ETag niet meer en weigert SAP de wijziging met status `412`. Zo overschrijf je nooit ongemerkt andermans werk.

De ETag komt van `etag master` in je behavior definition ([4.3](./module-4.md#4_3)).

**Foutmeldingen.** Faalt een validation, dan antwoordt SAP met status `400` en een vaste structuur:

```json
{
  "error": {
    "code": "…",
    "message": "Agentschap 999999 bestaat niet.",
    "target": "AgencyId",
    "details": []
  }
}
```

`message` is de tekst uit je validation ([4.5](./module-4.md#4_5)). Je client toont die tekst aan de gebruiker; ze moet de regel niet zelf opnieuw controleren.

De namespace van een action vind je in `$metadata`, bij het element `<Action Name="startPromotion">`. [CONTROLEREN] Exacte vorm van de namespace op het labosysteem.

## 5.4 Een externe client bouwen

De agentschappen hebben elk hun eigen systeem. Ze willen de actieve promoties ophalen voor hun website, en zelf een promotie voorstellen aan Zuidwind. Kunnen ze daarvoor dezelfde service gebruiken als de Fiori-app?

Technisch wel, maar het is geen goed idee. De UI-service is gemaakt voor één app: ze toont concepten, draft actions en UI-annotations. Verandert de app, dan verandert de service mee, en breekt de koppeling. Voor andere systemen maak je een aparte service: een **Web API**.

De lagen zijn dezelfde als in [4.9](./module-4.md#4_9), met een `A_`-projectie zonder draft. De agentschappen krijgen alleen de velden die ze nodig hebben:

[NIET GETEST] [CONTROLEREN] Of een behavior projection zonder `use draft` op een business object met draft activeert op ABAP Platform 2025, en of `create` dan meteen een actieve promotie maakt.

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Promotie (API)'
define root view entity ZSI00_A_Promotion
  provider contract transactional_query
  as projection on ZSI00_R_Promotion
{
  key PromoUuid,
      PromoId,
      AgencyId,
      Description,
      ValidFrom,
      ValidTo,
      DiscountPct,
      MaxDiscount,
      CurrencyCode,
      Status,
      LocalLastChangedAt
}
```

[NIET GETEST]

```abap
projection;
strict ( 2 );

define behavior for ZSI00_A_Promotion alias Promotion
{
  use create;
}
```

Een agentschap mag promoties lezen en voorstellen. Starten, wijzigen en verwijderen staan er niet in: dat blijft bij Wim. Maak daarna de service definition `ZSI00_API_PROMOTION` en een service binding `ZSI00_API_PROMOTION_O4` van het type **OData V4 - Web API**.

Een client in Node.js (versie 20 of hoger), zonder extra packages:

[NIET GETEST] Syntax gecontroleerd met `node --check`; niet uitgevoerd tegen het labosysteem.

```js
// agency.mjs
const BASE_URL = process.env.SAP_SERVICE_URL; // ends with /0001/
const AUTH = 'Basic ' + Buffer.from(`${process.env.SAP_USER}:${process.env.SAP_PASSWORD}`).toString('base64');

function odataQuery(options) {
  return Object.entries(options)
    .map(([name, value]) => `${name}=${encodeURIComponent(value)}`)
    .join('&');
}

async function readError(response) {
  try {
    const body = await response.json();
    return body.error?.message ?? `HTTP ${response.status}`;
  } catch {
    return `HTTP ${response.status}`;
  }
}

async function getActivePromotions(agencyId) {
  const query = odataQuery({
    $filter: `AgencyId eq '${agencyId}' and Status eq 'A'`,
    $select: 'PromoId,Description,DiscountPct,ValidTo',
    $orderby: 'ValidTo',
  });
  const response = await fetch(`${BASE_URL}Promotion?${query}`, {
    headers: { Authorization: AUTH, Accept: 'application/json' },
  });
  if (!response.ok) {
    throw new Error(await readError(response));
  }
  const body = await response.json();
  return body.value;
}

async function fetchCsrfToken() {
  const response = await fetch(BASE_URL, {
    headers: { Authorization: AUTH, 'X-CSRF-Token': 'Fetch' },
  });
  const cookies = response.headers.getSetCookie().map((cookie) => cookie.split(';')[0]);
  return { token: response.headers.get('x-csrf-token'), cookies: cookies.join('; ') };
}

async function proposePromotion(agencyId, discountPct, validFrom) {
  const { token, cookies } = await fetchCsrfToken();
  const response = await fetch(`${BASE_URL}Promotion`, {
    method: 'POST',
    headers: {
      Authorization: AUTH,
      Accept: 'application/json',
      'Content-Type': 'application/json',
      'X-CSRF-Token': token,
      Cookie: cookies,
    },
    body: JSON.stringify({
      AgencyId: agencyId,
      Description: 'Voorstel van het agentschap',
      DiscountPct: Number(discountPct),
      ValidFrom: validFrom,
    }),
  });
  if (!response.ok) {
    throw new Error(await readError(response));
  }
  return response.json();
}

const [command, agencyId, ...rest] = process.argv.slice(2);

if (command === 'actief') {
  try {
    const promotions = await getActivePromotions(agencyId);
    console.log(`Actieve promoties voor agentschap ${agencyId}:`);
    for (const promotion of promotions) {
      const discount = `${Number(promotion.DiscountPct).toFixed(0)} %`.padStart(5);
      console.log(`  ${promotion.PromoId}  ${discount}  tot ${promotion.ValidTo}  ${promotion.Description}`);
    }
  } catch (error) {
    console.log(`Opvragen mislukt: ${error.message}`);
  }
} else if (command === 'voorstellen') {
  const [discountPct, validFrom] = rest;
  try {
    const created = await proposePromotion(agencyId, discountPct, validFrom);
    console.log(`Promotie ${created.PromoId} voorgesteld, met status ${created.Status}.`);
  } catch (error) {
    console.log(`Voorstellen mislukt: ${error.message}`);
  }
} else {
  console.log('Gebruik: node agency.mjs actief <agentschap> | voorstellen <agentschap> <korting> <begindatum>');
}
```

Je start de client met de gegevens in omgevingsvariabelen. In PowerShell schrijf je `$env:SAP_USER = "…"` in plaats van `export`.

```bash
export SAP_SERVICE_URL="http://sap-lab.ehb.be:50000/sap/opu/odata4/sap/zsi00_api_promotion_o4/srvd_a2x/sap/zsi00_api_promotion/0001/"
export SAP_USER="ZSI00_AGENCY"
export SAP_PASSWORD="…"
node agency.mjs actief 070001
```

```output
Actieve promoties voor agentschap 070001:
  00000003   10 %  tot 2027-04-15  Voorjaarsactie 3
  00000007   15 %  tot 2027-03-31  Lentekorting
```

```output
$ node agency.mjs voorstellen 999999 15 2027-06-01
Voorstellen mislukt: Agentschap 999999 bestaat niet.
```

Merk op wat de client níét doet: controleren of het agentschap bestaat, of de korting tussen 1 en 50 % ligt. Dat doet het business object. Stuur je een onbestaand agentschap, dan geeft de client de melding van de validation door.

::: warning
**Let op:** zet een wachtwoord nooit in je code of in je Git-repository. Een `.env`-bestand hoort in `.gitignore`.
:::

## 5.5 SAP-API's in de SAP Business Accelerator Hub

Jouw API bestaat sinds vorige week. SAP S/4HANA biedt er honderden aan, voor verkooporders, producten, leveranciers en boekingen. Waar vind je ze, en hoe probeer je ze uit zonder S/4HANA-systeem?

De **SAP Business Accelerator Hub** ([api.sap.com](https://api.sap.com)) is de catalogus van alle gepubliceerde API's van SAP. Per API vind je de beschrijving, de entiteiten en velden, voorbeelden en een **sandbox**: een testsysteem met voorbeelddata waar je zelf aanvragen naar stuurt.

1. Ga naar [api.sap.com](https://api.sap.com) en zoek *Sales Order (A2X)*.
2. Kies de API voor SAP S/4HANA Cloud Public Edition.
3. Meld je aan met je SAP-account en open het tabblad **Try Out**. Je ziet de beschikbare operaties.
4. Kies `GET /A_SalesOrder`, vul bij `$top` de waarde `3` in en klik op **Run**.
5. Kopieer je **API key** via **Show API Key**. Daarmee spreek je de sandbox aan vanuit je eigen code.

[CONTROLEREN] Huidige naam en URL van de sandbox-API voor verkooporders.

```bash
curl -H "APIKey: <jouw API key>" -H "Accept: application/json" \
  "https://sandbox.api.sap.com/s4hanacloud/sap/opu/odata/sap/API_SALES_ORDER_SRV/A_SalesOrder?\$top=3&\$select=SalesOrder,SoldToParty,TotalNetAmount"
```

Twee dingen vallen op. Deze API is OData V2: het resultaat staat in `d.results` in plaats van `value`. En de namen van velden zijn dezelfde als in de CDS views van S/4HANA, zoals `SoldToParty`. Wie de views kent, kent de API.

Een API is niet de enige manier om systemen te koppelen. Welke aanpak past, hangt van het scenario af:

| Aanpak | Hoe het werkt | Geschikt als |
| --- | --- | --- |
| Synchrone API (OData, REST) | De client vraagt en wacht op het antwoord | Het antwoord is meteen nodig: een lijst tonen, een voorstel indienen |
| Events | SAP meldt dat er iets gebeurde; wie wil, luistert | Andere systemen moeten reageren op een wijziging, zonder dat SAP ze kent |
| Bestanden en batch | Data gaat in pakketten op vaste momenten | Grote hoeveelheden, geen haast: een dagelijkse export naar een datawarehouse |

Voor de lijst met actieve promoties volstaat een synchrone API. Moet de website van een agentschap meteen bijwerken zodra Wim een promotie start, dan zijn events een betere keuze. Anders moet elk agentschap elke minuut vragen of er iets veranderde. Zo'n afweging hoort in een ADR.

## Samenvatting

- **OData** is de standaard waarmee SAP data aanbiedt via HTTP; een service heeft `$metadata` en **entity sets**.
- **System query options** zoals `$filter`, `$select`, `$orderby`, `$top` en `$expand` laten de server het werk doen.
- Schrijven gebeurt met `POST`, `PATCH` en `DELETE`; je stuurt een **CSRF-token** mee en bij wijzigingen een **ETag** in `If-Match`.
- Een fout komt terug als JSON met `error.message`; je client toont die tekst.
- Voor andere systemen maak je een aparte **Web API**-binding met een eigen projectie, los van de UI-service.
- De **SAP Business Accelerator Hub** bevat alle API's van SAP, met een **sandbox** om ze te testen.
- Kies tussen synchrone API's, events en bestanden op basis van wat het scenario nodig heeft.

## Veelgemaakte fouten

- **Alles ophalen en in de client filteren.** Gebruik `$filter` en `$select`; dan gaat alleen de nodige data over het netwerk.
- **Een wijziging zonder CSRF-token.** SAP antwoordt met `403` en de header `X-CSRF-Token: Required`.
- **Het token zonder cookies meesturen.** Het token hoort bij een sessie; zonder de cookies uit dezelfde aanvraag is het ongeldig.
- **Een `PATCH` zonder ETag.** SAP weigert met `428`: het wil weten welke versie je wijzigt.
- **Bedrijfsregels dupliceren in de client.** Als de client zelf controleert of een agentschap bestaat, heb je twee versies van dezelfde regel. Eén ervan raakt verouderd.
- **De UI-service gebruiken voor een extern systeem.** Elke wijziging aan de app kan dan de koppeling breken.

::: oefening
[Projectstap P6 Klantenportaal](./project.md#p6)
:::
