---
module: 2
order: 1
moduleTitle: "2: Clean core en uitbreidbaarheid"
published: true
---

# Module 2: Clean core en uitbreidbaarheid

Wie de code van SAP aanpast, moet bij elke nieuwe versie opnieuw nagaan of alles nog werkt. In deze module leer je SAP uitbreiden zonder de standaard te wijzigen. Je leert ook hoe je zo'n keuze vastlegt, zodat een collega over een jaar nog begrijpt waarom het zo gebouwd is.

### Leerdoelen

Na deze module kan je:

- uitleggen waarom een clean core upgrades en onderhoud beheersbaar houdt;
- key-user-, developer- en side-by-side-extensibility onderscheiden;
- voor een requirement de gepaste uitbreidingsroute kiezen;
- in ADT nagaan of een object vrijgegeven is en voor welk gebruik;
- een keuze vastleggen in een architecture decision record met alternatieven en gevolgen.

## 2.1 Clean core: waarom je de standaard niet wijzigt

Jarenlang konden ontwikkelaars in SAP alles aanpassen, ook de code van SAP zelf. Een bedrijf wilde een extra controle bij het bewaren van een order? Dan zette een ontwikkelaar die controle midden in de standaardcode. Dat werkte, tot de volgende upgrade.

Bij een upgrade vervangt SAP zijn eigen code door een nieuwe versie. Elke aanpassing in die code moet iemand dan terugvinden, vergelijken en opnieuw inbouwen. Bedrijven met duizenden aanpassingen deden er maanden over, en sommige stelden een upgrade jarenlang uit.

**Clean core** is het principe dat je de standaard niet wijzigt. Je bouwt uitbreidingen ernaast, en je gebruikt alleen onderdelen van SAP waarvan SAP belooft dat ze stabiel blijven. Zo kan SAP zijn code vervangen zonder dat jouw uitbreiding breekt.

Clean core gaat over meer dan code. SAP rekent er ook bij:

- **processen** die zo dicht mogelijk bij de standaard blijven (fit-to-standard, [1.9](./module-1.md#1_9));
- **data** die niet gedupliceerd wordt en van goede kwaliteit is;
- **integraties** die via gepubliceerde API's lopen, niet via rechtstreekse toegang tot tabellen.

In SAP S/4HANA Cloud Public Edition is clean core verplicht: je kan de standaard daar niet eens aanpassen. In de andere vormen is het een keuze, maar wel de keuze die SAP en de meeste bedrijven vandaag maken.

Dit betekent voor jou: elke uitbreiding in het project moet de vraag doorstaan *breekt dit als SAP de standaard vervangt?* Is het antwoord ja, dan kies je een andere route.

## 2.2 Key-user-extensibility

De verkoopafdeling wil bij elke order een extra veld *Campagnecode*. Moet daar een ontwikkelaar aan te pas komen?

Niet altijd. **Key-user extensibility** (uitbreidbaarheid door sleutelgebruikers) laat een ervaren gebruiker van een afdeling, de **key user**, kleine uitbreidingen maken in Fiori-apps, zonder ontwikkelomgeving.

| App | Wat je ermee doet |
| --- | --- |
| **Custom Fields** | Een veld toevoegen aan een business object, met de schermen, API's en rapporten waar het moet verschijnen |
| **Custom Logic** | Een kort stuk ABAP-code schrijven op een vrijgegeven uitbreidingspunt, bijvoorbeeld een extra controle bij het bewaren |
| **Custom CDS Views** | Een eigen view samenstellen op basis van vrijgegeven views, voor een rapport of een API |
| **Adapt UI** | Velden op een scherm verbergen, verplaatsen of hernoemen |

Key-user-uitbreidingen zijn veilig voor upgrades, omdat ze alleen vrijgegeven uitbreidingspunten gebruiken. De keerzijde: je kan alleen uitbreiden wat SAP daarvoor openzet, en de logica blijft beperkt.

Het labosysteem heeft deze apps niet: ze horen bij S/4HANA, niet bij het ABAP-platform. Je ziet ze in de oefensystemen van SAP Learning Hub. Voor de fit-gapanalyse is het genoeg dat je weet dat ze bestaan en wat ze kunnen.

Dit betekent voor jou: vraagt een requirement alleen een extra veld of een extra controle op een bestaand business object, overweeg dan eerst key-user-extensibility.

## 2.3 Developer extensibility met ABAP Cloud

Het reisbureau wil dat agentschappen promoties aanbieden, met een eigen planning, statussen en controles. Daar is geen Custom Fields-app tegen opgewassen. Je hebt een ontwikkelaar nodig, maar je wil wel clean core blijven.

**Developer extensibility** (uitbreidbaarheid door ontwikkelaars) betekent dat je in hetzelfde systeem eigen objecten bouwt met **ABAP Cloud**. ABAP Cloud is geen andere taal, maar een strengere manier van ABAP schrijven:

- Je gebruikt alleen objecten van SAP die vrijgegeven zijn ([2.5](#2_5)).
- Je gebruikt een beperkte taalversie: **ABAP for Cloud Development**. Statements die rechtstreeks met de gebruikersinterface of het besturingssysteem werken, zijn verboden.
- Je bouwt business objects met het ABAP RESTful Application Programming Model (RAP, module 4).
- Je ontwikkelt in Eclipse met ADT, niet in SAP GUI.

De compiler controleert die regels. Gebruik je een object dat niet vrijgegeven is, dan krijg je een fout en kan je niet activeren. Dat is streng, maar het is precies wat clean core afdwingt.

Omdat jouw code in hetzelfde systeem draait, kan ze de data van SAP lezen zonder netwerkverkeer. Dat noemen we **on-stack**: op dezelfde technologiestack als de standaard.

::: info
Sinds augustus 2025 deelt SAP uitbreidingen in vier **clean core levels** in. Level A gebruikt alleen vrijgegeven, stabiele objecten: ABAP Cloud on-stack of side-by-side op SAP BTP. Level B gebruikt ook klassieke API's van SAP, level C ook interne objecten. Level D zijn technieken die SAP afraadt, zoals wijzigingen in de standaardcode.

Dit model vervangt de drie *tiers* die je in oudere documentatie tegenkomt. In dit vak werken we alleen op level A ([bron](https://news.sap.com/2025/08/extend-sap-s4hana-cloud-right-way-clean-clear/)).
:::

Het labosysteem laat alleen on-stack-ontwikkeling in ABAP Cloud toe. Dat is een beperking van de opleiding, geen argument: in je ADR weeg je de opties af alsof je vrij kon kiezen. Activeert een object niet omdat het niet vrijgegeven is? Dan is dat geen bug in je code, maar een signaal om een andere weg te zoeken.

## 2.4 Side-by-side-extensibility op SAP BTP

Een klantenportaal moet ook buiten de kantooruren werken, voor duizenden klanten, met een eigen look. Moet je dat in het ERP-systeem bouwen?

Bij **side-by-side extensibility** bouw je de uitbreiding buiten het ERP-systeem, meestal op **SAP Business Technology Platform** (SAP BTP), het cloudplatform van SAP. De uitbreiding spreekt S/4HANA aan via gepubliceerde API's.

| | On-stack (developer extensibility) | Side-by-side (SAP BTP) |
| --- | --- | --- |
| Waar draait de code | In het ERP-systeem | Op een apart platform |
| Toegang tot SAP-data | Rechtstreeks via vrijgegeven views en API's | Via het netwerk, met API's |
| Geschikt voor | Logica die nauw met SAP-data verweven is | Aparte apps, andere gebruikers, eigen schaal |
| Talen | ABAP | ABAP, Java, JavaScript (Node.js) |
| Nadeel | Belast het ERP-systeem | Netwerkverkeer, aparte beveiliging, aparte kosten |

Er is geen beste keuze. Een controle bij het bewaren van een reis hoort on-stack: ze moet in dezelfde transactie gebeuren. Een publieke webapp voor klanten hoort side-by-side: je wil geen duizenden anonieme gebruikers in je ERP-systeem.

Vaak combineer je beide. De promoties in de voorbeelden van deze cursus bouwen we on-stack. De agentschappen spreken ze aan met een klein programma buiten SAP dat de API gebruikt (module 5). Dat is side-by-side, alleen niet op SAP BTP maar met een technologie die je al kent.

## 2.5 Released objects: het contract met SAP

Je vindt in het systeem een tabel met precies de data die je nodig hebt. Mag je die zonder meer lezen? In ABAP Cloud hangt dat af van één eigenschap: is het object vrijgegeven?

Een **released object** (vrijgegeven object) is een object waarvan SAP belooft dat het stabiel blijft. SAP wijzigt het niet op een manier die jouw code breekt, of kondigt dat lang op voorhand aan. Elk vrijgegeven object heeft een of meer **release contracts**:

| Contract | Betekenis | Voorbeeld |
| --- | --- | --- |
| C0 | Je mag het uitbreiden | Een view waar je velden aan toevoegt |
| C1 | Je mag het gebruiken in je eigen ABAP Cloud-code | Een view of class die je leest of aanroept |
| C2 | Je mag het gebruiken als API van buitenaf | Een OData-service |
| C3 | Je mag er configuratie mee beheren | Instellingen voor een uitbreiding |

Of een object vrijgegeven is, zie je in ADT. Open het object en kies in het venster **Properties** het tabblad **API State**. Je ziet daar de contracten en voor welke taalversie ze gelden.

[CONTROLEREN] Welke `/DMO/`-objecten op het labosysteem vrijgegeven zijn voor ABAP for Cloud Development. Pas de voorbeelden in module 3 aan als de tabellen niet rechtstreeks leesbaar zijn.

Is een object niet vrijgegeven, dan zoek je een alternatief. SAP publiceert per object een opvolger als die bestaat. Een niet-vrijgegeven tabel heeft vaak een vrijgegeven CDS view met dezelfde data. Die lijst staat ook op GitHub: [SAP/abap-atc-cr-cv-s4hc](https://github.com/SAP/abap-atc-cr-cv-s4hc).

::: warning
**Let op:** *het werkt* is geen argument. Een niet-vrijgegeven object dat vandaag werkt, kan bij de volgende upgrade verdwijnen of anders werken. In klassieke ABAP merk je dat pas als het misgaat; in ABAP Cloud vangt de compiler het op.
:::

## 2.6 Architecture decision records: een keuze vastleggen

Over een jaar vraagt een nieuwe collega waarom het klantenportaal niet in SAP zelf zit. Niemand van het oorspronkelijke team werkt nog in het bedrijf. Waar staat het antwoord?

Een **architecture decision record** (ADR) is een kort document dat één belangrijke technische keuze vastlegt. Een ADR telt een halve tot één pagina en blijft bewaard, ook als de keuze later verandert. Dan schrijf je een nieuwe ADR die de oude vervangt.

Algemene vorm:

```markdown
# ADR <nummer>: <keuze in één zin>

Status: <voorgesteld | aanvaard | vervangen door ADR <nummer>>
Datum: <datum>

## Context
<Welk probleem lossen we op? Welke requirements en beperkingen spelen mee?>

## Alternatieven
<Minstens twee opties, elk met voor- en nadelen.>

## Beslissing
<Welke optie we kiezen.>

## Gevolgen
<Wat wordt makkelijker, wat moeilijker? Wat moeten we later opvolgen?>
```

Een voorbeeld uit de promoties die we vanaf module 3 bouwen:

```markdown
# ADR 2: Airlines van een promotie als child entity

Status: aanvaard
Datum: 12 maart 2027

## Context
Een promotie geldt voor één of meer airlines.
De marketingafdeling voegt per promotie airlines toe en verwijdert ze.

## Alternatieven
1. Eén tekstveld met airlinecodes, gescheiden door komma's.
   + Eén tabel, één invoerveld. − Geen controle op geldige codes, moeilijk te doorzoeken.
2. Een child entity met één rij per airline (composition).
   + Elke code is gecontroleerd en doorzoekbaar. − Extra tabel, views en behavior.

## Beslissing
Optie 2.

## Gevolgen
De detailpagina krijgt een tabel met airlines.
Een promotie zonder airlines is mogelijk; we beslissen later of dat mag.
```

Het belangrijkste onderdeel is *Alternatieven*. Een ADR zonder alternatieven legt een beslissing vast, maar geen afweging. Bij de verdediging vragen we precies naar die afweging.

## Samenvatting

- **Clean core**: je wijzigt de standaard niet en gebruikt alleen vrijgegeven onderdelen, zodat upgrades je uitbreiding niet breken.
- **Key-user extensibility**: kleine uitbreidingen in Fiori-apps zoals Custom Fields en Custom Logic, zonder ontwikkelomgeving.
- **Developer extensibility**: eigen objecten in ABAP Cloud, **on-stack** in hetzelfde systeem.
- **Side-by-side extensibility**: een aparte applicatie, meestal op **SAP BTP**, die SAP aanspreekt via API's.
- Een **released object** heeft **release contracts** (C0 tot C3); je controleert ze in ADT via **Properties > API State**.
- Een **ADR** legt één technische keuze vast met context, alternatieven, beslissing en gevolgen.

## Veelgemaakte fouten

- **Rechtstreeks lezen uit een niet-vrijgegeven tabel.** In ABAP Cloud activeert de code dan niet. Zoek de vrijgegeven view die dezelfde data aanbiedt.
- **Side-by-side kiezen voor alles.** Een controle die bij het bewaren moet gebeuren, kan niet in een aparte app: die zit niet in dezelfde transactie.
- **Een uitbreiding bouwen waar configuratie volstaat.** Een extra zoekveld is geen ontwikkelwerk.
- **Een ADR zonder alternatieven.** Dan is het geen afweging, maar een verslag.

::: oefening
[Projectstap P2 Keuze van de uitbreidingsroute](./project.md#p2)
:::
