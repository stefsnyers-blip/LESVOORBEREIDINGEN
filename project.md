---
module: 7
order: 2
tocMode: page
tocLabel: Project
published: true
---

# Project: wijzigingsaanvragen bij een reisbureau

Alle oefeningen van dit vak vormen samen één project. Per module werk je een projectstap af. Aan het einde heb je een werkende uitbreiding op SAP: een app, een API en een dossier dat je keuzes verantwoordt. Je werkt in een team van [CONTROLEREN] 2 of 3 studenten.

## De case {#case}

Reisbureau Zuidwind in Brussel werkt met het reisscenario `/DMO/` in SAP: reizen, boekingen, klanten en agentschappen ([1.8](./module-1.md#1_8)). Klanten vragen geregeld een wijziging aan hun reis aan: een andere vertrekdatum, een extra reiziger of een annulering.

Vandaag komen die vragen binnen per e-mail en telefoon. Een medewerker noteert ze in een gedeelde mailbox, en Ruben, de teamverantwoordelijke, beslist erover als hij tijd heeft. Aanvragen raken zoek, klanten bellen drie keer en niemand weet hoeveel annuleringen er deze maand waren.

Zuidwind wil een uitbreiding op SAP die aanvragen registreert, controleert en laat goedkeuren. Een klantenportaal moet aanvragen kunnen indienen en de status opvragen.

### Het proces

Het analyseteam leverde dit procesmodel af. Je docent bezorgt het ook als BPMN-diagram.

1. De klant vraagt een wijziging aan, via het klantenportaal of telefonisch.
2. Bij een telefonische aanvraag registreert een medewerker de aanvraag. Een aanvraag via het portaal komt rechtstreeks binnen.
3. Het systeem controleert de aanvraag en berekent eventuele kosten.
4. De teamverantwoordelijke keurt de aanvraag goed of wijst ze af.
5. De klant ziet de beslissing in het portaal.

Het effectief wijzigen van de reis valt buiten dit project: dat doet een medewerker na de goedkeuring in de bestaande reistoepassing.

### Requirements

| Code | Requirement |
| --- | --- |
| R1 | Een medewerker registreert een wijzigingsaanvraag voor een bestaande reis. |
| R2 | Een aanvraag heeft een type: datumwijziging, extra reiziger of annulering. |
| R3 | Bij een datumwijziging is de gevraagde nieuwe begindatum verplicht en ligt ze in de toekomst. |
| R4 | Elke aanvraag krijgt automatisch een uniek volgnummer. |
| R5 | Een nieuwe aanvraag krijgt de status *Open*. |
| R6 | De teamverantwoordelijke keurt een open aanvraag goed of wijst ze af. Daarna kan niemand de status nog wijzigen. |
| R7 | Bij een aanvraag ziet de medewerker de gegevens van de reis en de klant. |
| R8 | Medewerkers zoeken aanvragen op status, type en reis. |
| R9 | Een annulering minder dan 14 dagen voor de begindatum van de reis kost € 50. Een annulering daarvoor is gratis. |
| R10 | Het klantenportaal dient aanvragen in en vraagt de status ervan op. |
| R11 | De gegevens van reizen en klanten worden niet dubbel bijgehouden. |
| R12 | Een medewerker kan een onvolledige aanvraag bewaren en later afwerken. |
| R13 | Alleen een teamverantwoordelijke mag aanvragen goedkeuren of afwijzen. |

### Afspraken voor alle projectstappen

- Elk team krijgt een nummer van twee cijfers. Alle objecten van je team beginnen met `ZSI` en dat nummer, bijvoorbeeld `ZSI07_`. In deze opgave gebruiken we team `07` als voorbeeld.
- Je team werkt in één package `ZSI07` met als superpackage `ZLOCAL`. [CONTROLEREN] Superpackage en taalversie op het labosysteem.
- Elk object heeft de taalversie **ABAP for Cloud Development** ([2.3](./module-2.md#2_3)).
- Je houdt een projectdossier bij in de Git-repository van je team. Elke projectstap voegt er iets aan toe.

De cursustekst bouwt vanaf module 3 een andere toepassing: promoties per agentschap, met teamnummer `00`. Die toepassing gebruikt dezelfde technieken als je project, maar heeft een andere structuur en andere regels. Overnemen en hernoemen volstaat dus niet. Bij de verdediging leg je elke regel van je eigen code uit.

## P1 Fit-gapanalyse {#p1}

Hoort bij module 1. Mijlpaal 1, telt mee voor de evaluatie.

Leg elke requirement naast wat `/DMO/` en een standaard Fiori-app al bieden ([1.9](./module-1.md#1_9)).

1. Maak in je projectdossier het bestand `fit-gap.md` met deze tabel:

    ```markdown
    | Req. | Keuze | Wat bestaat er al? | Motivering | Impact |
    | --- | --- | --- | --- | --- |
    | R1 | Uitbreiden | /DMO/TRAVEL bevat de reis, niet de aanvraag | ... | Nieuwe tabel, nieuw business object |
    ```

2. Vul een rij in voor elke requirement R1 tot R13. *Keuze* is *Hergebruiken*, *Configureren* of *Uitbreiden*.
3. Noem bij *Wat bestaat er al?* het concrete object: een tabel, een view of een standaardfunctie van een Fiori-app.
4. Schrijf onder de tabel in maximaal vijf zinnen welke requirement het meeste werk vraagt en waarom.

**Oplevering:** `fit-gap.md` in je repository, ten laatste bij het begin van sessie 5.

**Wat we controleren:**

- Elke requirement heeft precies één keuze met een motivering.
- Minstens drie requirements zijn *Hergebruiken* of *Configureren*. Vind je er minder, kijk dan opnieuw naar R7, R8 en R11.
- Bij *Uitbreiden* staat welke data je nieuw opslaat en welke je leest uit `/DMO/`.

## P2 Keuze van de uitbreidingsroute {#p2}

Hoort bij module 2.

Je weet nu wat je moet uitbreiden. Beslis waar en hoe je dat bouwt.

1. Schrijf `adr/001-uitbreidingsroute.md` volgens de vorm uit [2.6](./module-2.md#2_6).
2. Vergelijk minstens on-stack (ABAP Cloud) en side-by-side voor het business object van de aanvragen.
3. Beslis apart waar het klantenportaal draait. Mag dat een andere keuze zijn dan voor het business object? Verantwoord je antwoord.
4. Ga na of de `/DMO/`-objecten die je wil lezen vrijgegeven zijn ([2.5](./module-2.md#2_5)). Noteer het resultaat bij *Gevolgen*.

**Oplevering:** `adr/001-uitbreidingsroute.md`.

**Wat we controleren:**

- Minstens twee alternatieven, elk met een voordeel en een nadeel.
- De beslissing volgt uit de context en de alternatieven, niet uit een voorkeur.
- *Gevolgen* noemt minstens één nadeel van de gekozen optie.

## P3 Datamodel {#p3}

Hoort bij module 3.

Bouw de tabel voor de aanvragen en de views die ze verbinden met de bestaande reisdata. Kopieer geen reis- of klantgegevens (R11).

1. Maak de domains `ZSI07_REQ_STATUS` en `ZSI07_REQ_TYPE` met vaste waarden, en voor elk een data element met dezelfde naam ([3.1](./module-3.md#3_1)).

    | Domain | Waarde | Betekenis |
    | --- | --- | --- |
    | `ZSI07_REQ_STATUS` | `O` | Open |
    | | `A` | Goedgekeurd |
    | | `R` | Afgewezen |
    | `ZSI07_REQ_TYPE` | `D` | Datumwijziging |
    | | `T` | Extra reiziger |
    | | `C` | Annulering |

2. Maak de tabel `ZSI07_CHGREQ` ([3.1](./module-3.md#3_1)). Ze bevat minstens:
    - een technische sleutel (UUID) en een volgnummer;
    - het reisnummer, het type, de gevraagde datum en een omschrijving;
    - de status, de kost met munteenheid en de beheervelden.
3. Maak de basic views `ZSI07_I_Travel` op `/DMO/TRAVEL` en `ZSI07_I_Customer` op `/DMO/CUSTOMER`, met een association van reis naar klant ([3.4](./module-3.md#3_4), [3.5](./module-3.md#3_5)).
4. Maak de view `ZSI07_I_ChangeRequest` op je tabel, met een association naar `ZSI07_I_Travel`.
5. Schrijf een class `ZSI07_CL_DATA_GENERATOR` die vijf testaanvragen aanmaakt voor bestaande reizen ([3.3](./module-3.md#3_3)).

**Oplevering:** de objecten in package `ZSI07`, en `datamodel.md` in je dossier met een schets van de tabellen, views en associations.

**Wat we controleren:** deze query in de SQL console van ADT geeft vijf rijen met een klantnaam die niet in je eigen tabel staat.

```sql
SELECT FROM zsi07_i_changerequest
  FIELDS RequestId, TravelId, \_Travel\_Customer-LastName
```

[NIET GETEST] De query gebruikt een path expression ([3.5](./module-3.md#3_5)).

## P4 Business object en bedrijfslogica {#p4}

Hoort bij module 4, sessies 8 en 9.

Bouw een RAP business object voor de aanvragen, met de bedrijfsregels uit de analyse.

1. Maak de root view `ZSI07_R_ChangeRequest` en de behavior definition, managed, met draft ([4.2](./module-4.md#4_2) tot [4.4](./module-4.md#4_4), [4.8](./module-4.md#4_8)).
2. Implementeer deze regels:

    | Regel | Requirement | Soort |
    | --- | --- | --- |
    | Het volgnummer wordt ingevuld bij het bewaren | R4 | Determination |
    | De status wordt *Open* bij het aanmaken | R5 | Determination |
    | De kost wordt berekend bij een annulering | R9 | Determination |
    | De reis moet bestaan | R1 | Validation |
    | Bij een datumwijziging is de datum verplicht en in de toekomst | R3 | Validation |
    | Goedkeuren en afwijzen | R6 | Actions |
    | Goedkeuren en afwijzen alleen bij status *Open* | R6 | Instance features |
    | Status, volgnummer en kost zijn niet rechtstreeks te wijzigen | R4, R5, R6, R9 | Field control |

3. Test elke regel met een class die `if_oo_adt_classrun` implementeert en EML gebruikt ([4.3](./module-4.md#4_3)).

**Oplevering:** de objecten in package `ZSI07`.

**Wat we controleren:** je testclass toont voor deze scenario's de verwachte uitkomst.

```output
Aanvraag voor reis 00000099 (bestaat niet): afgewezen - Reis 00000099 bestaat niet.
Datumwijziging zonder datum: afgewezen - Vul een nieuwe begindatum in.
Annulering 5 dagen voor vertrek: bewaard - kost 50.00 EUR
Annulering 60 dagen voor vertrek: bewaard - kost 0.00 EUR
Goedkeuren van een goedgekeurde aanvraag: niet toegelaten
```

## P5 Fiori-app {#p5}

Hoort bij module 4, sessie 10. Mijlpaal 2, telt mee voor de evaluatie.

Bied het business object aan als Fiori elements-app voor de medewerkers.

1. Maak de projectie `ZSI07_C_ChangeRequest` met haar behavior projection ([4.9](./module-4.md#4_9)).
2. Maak de service definition `ZSI07_UI_CHANGEREQUEST` en een service binding van het type OData V4 - UI ([4.9](./module-4.md#4_9)).
3. Voeg met een metadata extension de annotaties toe voor een lijst en een detailpagina ([4.10](./module-4.md#4_10)).
4. Implementeer de authorizations voor R13 ([4.11](./module-4.md#4_11)). Je docent maakt het authorization object `ZSI_REQ` met het veld `ACTVT` aan, en per team twee testgebruikers: een medewerker en een teamverantwoordelijke. Beslissen over een aanvraag is activiteit `43`. [CONTROLEREN] Authorization object, rollen en testgebruikers aanmaken op het labosysteem.

**Oplevering:** de app werkt in de preview van de service binding.

**Wat we controleren:**

- De lijst toont volgnummer, reis, type, status en kost, en filtert op status, type en reis (R8).
- De detailpagina toont de begindatum van de reis en de naam van de klant (R7).
- De knoppen **Goedkeuren** en **Afwijzen** staan in de lijst en op de detailpagina, en zijn uitgeschakeld bij een aanvraag die niet open is.
- Een onvolledige aanvraag blijft bewaard als concept na het sluiten van de browser (R12).
- Een foutmelding van een validation verschijnt bij het juiste veld.

## P6 Klantenportaal {#p6}

Hoort bij module 5.

Bouw een klein klantenportaal buiten SAP dat je business object gebruikt via OData (R10). Je kiest de technologie: Node.js, Laravel of Python.

1. Maak een API-projectie en een service binding van het type OData V4 - Web API ([5.4](./module-5.md#5_4)).
2. Bouw een client die:
    - de aanvragen voor een reisnummer toont, met type, status en kost;
    - een nieuwe aanvraag indient;
    - een foutmelding van SAP leesbaar toont aan de gebruiker.
3. Bewaar gebruikersnaam en wachtwoord voor SAP in omgevingsvariabelen, nooit in je code.
4. Schrijf `adr/002-integratie.md`: waarom een aparte Web API-binding en niet de UI-binding van P5?

**Oplevering:** de code in de map `portal/` van je repository, met een `README.md` die uitlegt hoe je de client start.

**Wat we controleren:**

```output
$ node portal.js status 00000012
Aanvragen voor reis 00000012:
  00000004  Annulering      Open          50.00 EUR
  00000007  Datumwijziging  Goedgekeurd    0.00 EUR

$ node portal.js indienen 00000099 C
Indienen mislukt: Reis 00000099 bestaat niet.
```

De client controleert zelf niet of de reis bestaat: die regel zit in het business object.

## P7 Oplevering {#p7}

Hoort bij module 6.

Werk het project af zodat een ander team erop verder zou kunnen bouwen.

1. Schrijf ABAP Unit-tests voor minstens drie regels uit P4, waaronder R3 en R9 ([6.1](./module-6.md#6_1), [6.2](./module-6.md#6_2)).
2. Zet het package `ZSI07` in een Git-repository met abapGit ([6.3](./module-6.md#6_3)).
3. Werk het projectdossier af ([6.4](./module-6.md#6_4)): fit-gapanalyse, 3 tot 5 ADR's, datamodel, korte handleiding voor de app en het portaal.
4. Vul in `bijdragen.md` per teamlid in welke objecten hij of zij bouwde.

**Oplevering:** de URL van de repository, ten laatste op de datum die je docent meedeelt.

**Wat we controleren:**

- Alle tests slagen; elke test werkt ook op een systeem zonder jouw testdata.
- De repository bevat alle objecten van het package en het volledige dossier.
- Elke ADR heeft alternatieven en gevolgen.

De mondelinge verdediging gaat over je eigen aandeel in dit project. Bereid je voor op vragen over elke keuze in je ADR's en over elk object dat je bouwde.
