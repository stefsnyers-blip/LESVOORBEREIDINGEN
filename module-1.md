---
module: 1
order: 1
moduleTitle: "1: SAP als standaardsoftware"
published: true
---

# Module 1: SAP als standaardsoftware

Een bedrijf koopt SAP om niet alles zelf te moeten bouwen. In deze module leer je hoe SAP bedrijfsprocessen standaard ondersteunt en hoe je een bestaande analyse daarop afstemt. Pas als je weet wat er al is, kan je beslissen wat je nog moet bouwen.

### Leerdoelen

Na deze module kan je:

- de opbouw van SAP S/4HANA beschrijven met organisatie-eenheden, stamdata en transactiedata;
- een order-to-cash- en een procure-to-pay-proces volgen via de documentflow;
- een BPMN-model koppelen aan SAP-business objects en hun API's;
- in het /DMO-referentiescenario aanwijzen welke data en functionaliteit herbruikbaar zijn;
- per requirement verantwoorden of je hergebruikt, configureert of uitbreidt.

## 1.1 ERP: één systeem voor het hele bedrijf

Stel dat een fietsenhandel elke afdeling haar eigen software geeft. Verkoop noteert een order in een webshop, het magazijn werkt met een Excel-lijst en de boekhouding typt de factuur over in haar boekhoudpakket. Elke overdracht is een kans op fouten: een verkeerd aantal, een vergeten factuur, een klant met drie verschillende adressen.

Een **ERP-systeem** (Enterprise Resource Planning) lost dat op met één database voor alle afdelingen. De verkoper maakt een order aan, het magazijn ziet die order meteen en de boekhouding krijgt automatisch een boeking bij de levering. Niemand typt iets over.

Dat is het centrale idee van dit vak: in een ERP-systeem is een bedrijfsproces een reeks documenten die op elkaar verder bouwen. Een stap in het proces maakt een document aan dat naar het vorige verwijst.

**SAP S/4HANA** is het ERP-systeem van SAP. Het draait op SAP HANA, een in-memory database. De S staat voor *simple*, de 4 voor de vierde generatie. S/4HANA bestaat in drie vormen:

| Vorm | Wie beheert de software | Aanpassingen |
| --- | --- | --- |
| SAP S/4HANA (on-premise) | Het bedrijf zelf | Alles mag, ook wat beter niet mag |
| SAP S/4HANA Cloud Private Edition | SAP, op een eigen omgeving per klant | Ruim, binnen afspraken |
| SAP S/4HANA Cloud Public Edition | SAP, gedeelde software voor alle klanten | Alleen via vrijgegeven uitbreidingspunten |

Hoe meer een bedrijf de standaard volgt, hoe makkelijker het overstapt naar een nieuwe versie. Dat idee werken we uit in module 2.

## 1.2 De architectuur van SAP S/4HANA

Een SAP-systeem lijkt aan de buitenkant op een webapplicatie met veel schermen. Daaronder zit een duidelijke gelaagde opbouw, en die opbouw bepaalt hoe je het later uitbreidt.

| Laag | Wat er gebeurt | Technologie |
| --- | --- | --- |
| Gebruikersinterface | Apps in de browser | SAP Fiori (SAPUI5), vroeger SAP GUI |
| Services | Data en acties aanbieden aan apps en andere systemen | OData, REST, SOAP |
| Applicatie | Bedrijfslogica: controles, berekeningen, statussen | ABAP |
| Datamodel | Een leesbare laag boven de tabellen | CDS views |
| Database | Opslag | SAP HANA |

**SAP Fiori** is de gebruikersinterface van S/4HANA: apps met een vaste vormgeving, die je opent vanuit een startpagina, de **Fiori launchpad**. Oudere schermen bereik je via **transactiecodes** zoals `VA01` (verkooporder aanmaken). Je komt die codes nog overal tegen in documentatie en vacatures, dus we noemen ze in deze module telkens mee.

**ABAP** (Advanced Business Application Programming) is de programmeertaal van SAP. De standaardsoftware van SAP is erin geschreven, en jij schrijft er vanaf module 3 je uitbreidingen in.

**CDS views** (Core Data Services) zijn views op de database met extra betekenis: welke kolom een bedrag is, welke tabel met welke andere samenhangt. In module 3 schrijf je ze zelf.

Elk SAP-systeem is opgedeeld in **clients** (mandanten): logisch gescheiden omgevingen in dezelfde installatie. Het labosysteem gebruikt client `001`. Data in client 001 is niet zichtbaar in een andere client, maar de code is dezelfde.

## 1.3 Organisatie-eenheden: hoe SAP een bedrijf opdeelt

Een multinational heeft vestigingen in verschillende landen, met elk hun eigen boekhouding, magazijnen en verkoopteams. SAP moet weten welke order bij welke vestiging hoort, anders klopt geen enkele balans.

Daarom start elke SAP-implementatie met **organisatie-eenheden**: de structuur van het bedrijf zoals SAP ze ziet. De belangrijkste:

| Organisatie-eenheid | Betekenis | Voorbeeld |
| --- | --- | --- |
| **Company code** (bedrijfscode) | Juridische eenheid met een eigen boekhouding | Velo Brussel NV |
| **Plant** (vestiging) | Plaats waar goederen gemaakt of bewaard worden | Magazijn Vilvoorde |
| **Storage location** (opslaglocatie) | Deel van een plant | Hal B |
| **Sales organization** (verkooporganisatie) | Verkoopt en is verantwoordelijk voor de verkoop | Verkoop België |
| **Distribution channel** (distributiekanaal) | Manier waarop je verkoopt | Webshop, groothandel |
| **Purchasing organization** (inkooporganisatie) | Onderhandelt met leveranciers | Inkoop Benelux |

Een verkooporder hoort altijd bij een combinatie van verkooporganisatie, distributiekanaal en **division** (productgroep). Die combinatie heet de **sales area**.

Organisatie-eenheden vul je één keer in bij de implementatie en daarna zelden nog. Dat heet **customizing** (configuratie): instellingen in SAP aanpassen zonder code te schrijven. Een verkeerde keuze hier is duur, want alle documenten bouwen erop verder.

Dit betekent voor jou: begint een analyse met een organigram, dan vertaal je dat eerst naar organisatie-eenheden. Pas daarna kijk je naar de processen.

## 1.4 Stamdata en transactiedata

Een verkooporder bevat een klant, producten, prijzen en een leveringsdatum. Typ je bij elke order naam en adres van de klant opnieuw in? Dan heb je na een jaar honderd varianten van dezelfde klant.

SAP scheidt daarom twee soorten data.

**Master data** (stamdata) verandert weinig en gebruik je in veel documenten: klanten, leveranciers, producten, prijzen. In S/4HANA zijn klanten en leveranciers allebei een **business partner** (zakenpartner). Een bedrijf dat zowel koopt als verkoopt, staat er dus maar één keer in.

**Transactional data** (transactiedata) ontstaat bij elke stap in een proces: een order, een levering, een factuur, een betaling. Elk document verwijst naar stamdata en meestal naar het vorige document.

| Stamdata | Transactiedata |
| --- | --- |
| Business partner (klant, leverancier) | Verkooporder, inkooporder |
| Product | Levering, goederenontvangst |
| Prijsvoorwaarden | Factuur, betaling |
| Grootboekrekeningen | Journaalpost |

Het onderscheid is meer dan een theoretische indeling. Voor stamdata ontwerp je goede controles bij het aanmaken, want elke fout herhaalt zich in duizenden documenten. Voor transactiedata ontwerp je de statussen en de overgangen tussen documenten.

Dit betekent voor jou: in een datamodel uit een analysevak herken je stamdata aan entiteiten die veel relaties hebben en weinig veranderen. In SAP bestaan die meestal al.

## 1.5 Order-to-cash: van verkooporder tot betaling

Een klant bestelt tien fietsen. Hoe wordt dat in SAP een betaling op de bankrekening, en wie doet wat?

Het proces van bestelling tot betaling heet **order-to-cash** (O2C). In SAP is elke stap een document:

| Stap | Document | Transactie | Wat er gebeurt |
| --- | --- | --- | --- |
| 1 | Verkooporder | `VA01` | Klant, producten, aantallen, prijs, leveringsdatum |
| 2 | Uitgaande levering | `VL01N` | Het magazijn plant de levering |
| 3 | Goods issue (goederenuitgifte) | `VL02N` | Voorraad daalt, de kostprijs wordt geboekt |
| 4 | Factuur | `VF01` | Omzet en een openstaande vordering op de klant |
| 5 | Inkomende betaling | `F-28` | De vordering wordt afgeboekt |

Elk document verwijst naar het vorige. Die keten heet de **document flow** (documentflow). Open je in SAP een factuur, dan zie je de order en de levering waaruit ze ontstaan is.

Belangrijk is wat er in de boekhouding gebeurt zonder dat iemand iets intypt. De goederenuitgifte boekt de voorraad af, de factuur boekt omzet en een vordering, de betaling sluit die vordering. Alle boekingen komen in één tabel terecht: de **universal journal** (`ACDOCA`).

::: warning
**Let op:** een verkooporder maakt nog geen boeking in de boekhouding. Pas bij de goederenuitgifte en de factuur verandert er iets in de cijfers. Die stap vergeten verklaart veel verbaasde vragen over "verdwenen" omzet.
:::

In de oefensystemen van SAP Learning Hub voer je dit proces één keer zelf uit. Je docent toont welk oefensysteem en welk scenario je gebruikt. [CONTROLEREN] Welk oefensysteem in de student edition een O2C-scenario in S/4HANA aanbiedt.

## 1.6 Procure-to-pay en de koppeling met finance

Voor die tien fietsen moet de fietsenhandel eerst onderdelen aankopen. Het aankoopproces is het spiegelbeeld van order-to-cash en heet **procure-to-pay** (P2P).

| Stap | Document | Transactie | Wat er gebeurt |
| --- | --- | --- | --- |
| 1 | Aankoopaanvraag | `ME51N` | Iemand vraagt intern iets aan |
| 2 | Inkooporder | `ME21N` | De bestelling bij de leverancier |
| 3 | Goederenontvangst | `MIGO` | Voorraad stijgt |
| 4 | Leveranciersfactuur | `MIRO` | Een openstaande schuld aan de leverancier |
| 5 | Betaling | `F110` | De schuld wordt afgeboekt |

Bij de factuur gebeurt iets typisch voor ERP: de **three-way match** (driewegscontrole). SAP vergelijkt inkooporder, goederenontvangst en factuur. Factureert de leverancier twaalf stuks terwijl je er tien ontving, dan blokkeert SAP de betaling.

De koppeling met finance werkt zoals bij O2C. De goederenontvangst boekt voorraad, de factuur boekt een schuld, de betaling sluit die schuld. De boekhouder ziet elke stap in de universal journal en kan van elke boeking teruggaan naar het document dat ze veroorzaakte.

Dit betekent voor jou: een proces in SAP is nooit "alleen verkoop" of "alleen aankoop". Wijzig je iets aan een stap, ga dan na welke boekingen en welke volgende documenten erop steunen.

## 1.7 Van BPMN naar business objects en API's

In de analysevakken tekende je processen in **BPMN**: taken, gebeurtenissen en beslissingen in swimlanes. Hoe maak je de stap van zo'n model naar SAP?

Neem een klein BPMN-model: *Klant bestelt*, *Order registreren*, *Goederen leveren*, *Factureren*. Elke taak die iets vastlegt, wordt in SAP een document. In SAP heet zo'n document met zijn regels, statussen en acties een **business object** (bedrijfsobject).

| BPMN-taak | Business object in SAP | API |
| --- | --- | --- |
| Order registreren | Sales order | Sales Order (A2X) |
| Goederen leveren | Outbound delivery | Outbound Delivery (A2X) |
| Factureren | Billing document | Billing Document (A2X) |
| Betaling ontvangen | Journal entry | Journal Entry (A2X) |

Een **API** (Application Programming Interface) is de manier waarop een ander systeem een business object aanspreekt. SAP publiceert zijn API's in de **SAP Business Accelerator Hub** ([api.sap.com](https://api.sap.com)). Daar zie je per business object welke velden en bewerkingen er zijn.

Je kan er ook API's uitproberen in een sandbox. Dat doen we in [5.5](./module-5.md#5_5).

Werk bij het vertalen van een BPMN-model in drie stappen:

1. Duid de taken aan die data vastleggen of wijzigen. Taken die alleen een beslissing nemen, worden meestal geen business object.
2. Zoek per taak het business object in SAP. Begin bij de API-lijst in de Business Accelerator Hub.
3. Noteer wat je model nodig heeft en het business object niet biedt. Dat is je gap.

::: warning
**Let op:** een swimlane is geen organisatie-eenheid. Een lane *Verkoop* zegt wie een taak doet, niet welke verkooporganisatie in SAP verantwoordelijk is.
:::

## 1.8 Het /DMO-referentiescenario: een reisbureau in SAP

Het labosysteem is geen volledig S/4HANA-systeem. Er staan geen verkooporders of facturen in. Wat staat er dan wel in om op verder te bouwen?

SAP levert bij het ABAP-platform een voorbeeldtoepassing mee: het **flight reference scenario**, met objecten waarvan de naam begint met `/DMO/`. Het is een reisbureau met reizen, boekingen, klanten, agentschappen en vluchten. SAP gebruikt het zelf in al zijn tutorials over ABAP Cloud.

| Tabel | Inhoud | Soort data |
| --- | --- | --- |
| `/DMO/TRAVEL` | Reizen: klant, agentschap, begin- en einddatum, prijs | Transactiedata |
| `/DMO/BOOKING` | Boekingen van vluchten binnen een reis | Transactiedata |
| `/DMO/CUSTOMER` | Klanten van het reisbureau | Stamdata |
| `/DMO/AGENCY` | Reisagentschappen | Stamdata |
| `/DMO/FLIGHT`, `/DMO/CONNECTION`, `/DMO/CARRIER` | Vluchten, verbindingen en luchtvaartmaatschappijen | Stamdata |

In dit vak behandelen we `/DMO/` als de standaardsoftware van ons bedrijf. De reizen en klanten bestaan al. Jouw project voegt daar iets aan toe zonder die data te kopiëren of te wijzigen. Zo oefen je hetzelfde als een ontwikkelaar die S/4HANA uitbreidt, alleen met reizen in plaats van verkooporders.

Je kan de data nu al bekijken. Open in Eclipse je project, druk op **Ctrl+Shift+A**, typ `/DMO/TRAVEL` en open de tabel. Druk op **F8**: je ziet de reizen in de **Data Preview**. Meer daarover in [3.2](./module-3.md#3_2).

## 1.9 Fit-to-standard: hergebruiken, configureren of uitbreiden

Een analist levert twintig requirements af. De reflex van een ontwikkelaar is om ze allemaal te bouwen. In een SAP-project is dat bijna altijd de duurste keuze.

**Fit-to-standard** is de werkwijze waarbij je elke requirement eerst naast de standaard legt. Per requirement kies je een van drie antwoorden:

| Keuze | Betekenis | Voorbeeld |
| --- | --- | --- |
| **Hergebruiken** | De standaard doet het al | Klantgegevens tonen bij een reis |
| **Configureren** | De standaard doet het na een instelling | Een extra zoekveld in een lijst |
| **Uitbreiden** | Er is nieuwe functionaliteit nodig | Een goedkeuringsstap die SAP niet kent |

Het resultaat is een **fit-gap analysis** (fit-gapanalyse): een tabel met per requirement de keuze en een korte motivering. Voor elke *uitbreiden* schat je de impact: welke data heb je nodig, welke bestaande data lees je, wat moet de gebruiker zien.

Twee vragen helpen bij twijfel:

1. Heeft het bedrijf dit echt anders nodig, of is het een gewoonte? Soms past een bedrijf beter zijn proces aan dan de software.
2. Wat kost het om dit bij elke upgrade opnieuw te testen? Elke uitbreiding moet je onderhouden.

Een goede fit-gapanalyse heeft meer *hergebruiken* dan *uitbreiden*. Een analyse met alleen *uitbreiden* betekent meestal dat de standaard niet bekeken is.

Dit betekent voor jou: de eerste projectstap is een fit-gapanalyse van de case. Je baseert je op wat je in deze module zag van `/DMO/` en van S/4HANA.

## Samenvatting

- Een **ERP-systeem** gebruikt één database voor alle afdelingen; een proces is een reeks documenten die naar elkaar verwijzen.
- **SAP S/4HANA** bestaat uit Fiori-apps, services, ABAP-logica, CDS views en een HANA-database.
- **Organisatie-eenheden** (company code, plant, sales organization, …) beschrijven het bedrijf; je stelt ze in via **customizing**.
- **Master data** verandert weinig; **transactional data** ontstaat bij elke processtap.
- **Order-to-cash** en **procure-to-pay** boeken automatisch in de **universal journal**; de **document flow** verbindt alle documenten.
- Een BPMN-taak die data vastlegt, wordt in SAP een **business object** met een **API**.
- Het **/DMO-referentiescenario** speelt in dit vak de rol van de standaardsoftware.
- **Fit-to-standard**: per requirement kies je hergebruiken, configureren of uitbreiden.

## Veelgemaakte fouten

- **SAP zien als een verzameling schermen.** Wie denkt in schermen, wil een scherm bijbouwen. Wie denkt in business objects, vraagt eerst welk object de data al bevat.
- **Maatwerk voorstellen voor wat de standaard al dekt.** *We bouwen een klantenlijst* terwijl `/DMO/CUSTOMER` en de bijhorende views al bestaan.
- **Stamdata en transactiedata door elkaar halen.** Een klant is stamdata; een reis van die klant is transactiedata. Een wijzigingsaanvraag bij een reis is opnieuw transactiedata.
- **Swimlanes vertalen naar organisatie-eenheden.** Een lane zegt wie iets doet, niet welke juridische of logistieke eenheid verantwoordelijk is.

::: oefening
[Projectstap P1 Fit-gapanalyse](./project.md#p1)
:::
