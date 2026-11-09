---
module: 6
order: 1
moduleTitle: "6: Kwaliteit en oplevering"
published: true
---

# Module 6: Kwaliteit en oplevering

Werkende code is pas af als je kan aantonen dat ze werkt, en als iemand anders erop kan verder bouwen. In deze module test je de bedrijfslogica van een business object, zet je je code onder versiebeheer en werk je het projectdossier af. Het dossier is ook de basis van je verdediging.

### Leerdoelen

Na deze module kan je:

- de bedrijfslogica van een business object testen met ABAP Unit;
- afhankelijkheden van de database vervangen door test doubles;
- ABAP-code beheren in een Git-repository met abapGit;
- je ontwerpkeuzes samenbrengen en verdedigen in een projectdossier.

## 6.1 ABAP Unit: logica automatisch testen

Je verandert de standaardduur van een promotie van 30 naar 45 dagen. Werkt de controle van de datums daarna nog? Om dat na elke wijziging met de hand te controleren, klik je tien minuten door de app. Na de derde keer sla je het over.

**ABAP Unit** is het testframework van ABAP. Een test is een method die code uitvoert en het resultaat vergelijkt met wat je verwacht. ADT voert alle tests in een paar seconden uit, zo vaak als je wil.

Tests schrijven gaat het makkelijkst als de logica los staat van de database en van RAP. Daarom halen we de regels voor de datums uit de behavior pool en zetten we ze in een eigen class:

[NIET GETEST]

```abap
CLASS zsi00_cl_promo_rules DEFINITION
  PUBLIC FINAL CREATE PUBLIC.

  PUBLIC SECTION.
    CONSTANTS default_duration_days TYPE i VALUE 30.

    CLASS-METHODS default_valid_to
      IMPORTING valid_from      TYPE d
      RETURNING VALUE(valid_to) TYPE d.

    CLASS-METHODS check_dates
      IMPORTING valid_from     TYPE d
                valid_to       TYPE d
                today          TYPE d
      RETURNING VALUE(message) TYPE string.
ENDCLASS.

CLASS zsi00_cl_promo_rules IMPLEMENTATION.
  METHOD default_valid_to.
    valid_to = valid_from + default_duration_days.
  ENDMETHOD.

  METHOD check_dates.
    message = COND #(
      WHEN valid_from IS INITIAL THEN `Vul een begindatum in.`
      WHEN valid_to < valid_from THEN `De einddatum ligt voor de begindatum.`
      WHEN valid_to < today      THEN `De einddatum ligt in het verleden.` ).
  ENDMETHOD.
ENDCLASS.
```

De determination `setDefaultValidTo` en de validation `validateDates` ([4.5](./module-4.md#4_5), [4.6](./module-4.md#4_6)) roepen nu deze class aan in plaats van zelf te rekenen. Merk op dat `today` een parameter is. Zo kan een test zelf kiezen welke dag *vandaag* is.

Een test schrijf je in het tabblad **Test Classes** onderaan de editor van de class:

[NIET GETEST]

```abap
CLASS ltc_promo_rules DEFINITION FINAL
  FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.

  PRIVATE SECTION.
    METHODS default_runs_30_days FOR TESTING.
    METHODS end_before_start_is_refused FOR TESTING.
    METHODS end_in_past_is_refused FOR TESTING.
    METHODS end_today_is_accepted FOR TESTING.
ENDCLASS.

CLASS ltc_promo_rules IMPLEMENTATION.
  METHOD default_runs_30_days.
    cl_abap_unit_assert=>assert_equals(
      exp = CONV d( '20270331' )
      act = zsi00_cl_promo_rules=>default_valid_to( '20270301' ) ).
  ENDMETHOD.

  METHOD end_before_start_is_refused.
    DATA(message) = zsi00_cl_promo_rules=>check_dates( valid_from = '20270310'
                                                       valid_to   = '20270301'
                                                       today      = '20270101' ).
    cl_abap_unit_assert=>assert_not_initial( message ).
  ENDMETHOD.

  METHOD end_in_past_is_refused.
    DATA(message) = zsi00_cl_promo_rules=>check_dates( valid_from = '20270101'
                                                       valid_to   = '20270131'
                                                       today      = '20270201' ).
    cl_abap_unit_assert=>assert_not_initial( message ).
  ENDMETHOD.

  METHOD end_today_is_accepted.
    DATA(message) = zsi00_cl_promo_rules=>check_dates( valid_from = '20270101'
                                                       valid_to   = '20270131'
                                                       today      = '20270131' ).
    cl_abap_unit_assert=>assert_initial( message ).
  ENDMETHOD.
ENDCLASS.
```

Algemene vorm:

```abap
CLASS <test class> DEFINITION FINAL
  FOR TESTING DURATION <SHORT | MEDIUM | LONG> RISK LEVEL <HARMLESS | DANGEROUS | CRITICAL>.

  PRIVATE SECTION.
    METHODS <test method> FOR TESTING.
ENDCLASS.
```

- `FOR TESTING` maakt van een class of method een test. Ze komt nooit in de productiecode terecht.
- `RISK LEVEL HARMLESS` zegt dat de test niets blijvends wijzigt.
- `cl_abap_unit_assert` bevat de controles: `assert_equals`, `assert_initial`, `assert_true` en andere.
- De naam van een test beschrijft het verwachte gedrag: `end_in_past_is_refused`. Faalt hij, dan weet je meteen wat er misloopt.

Voer de tests uit met **Ctrl+Shift+F10**. Je ziet het resultaat in het venster **ABAP Unit**: groen als alles slaagt, rood met de reden als een test faalt.

Test vooral de grensgevallen. Mag een promotie die vandaag eindigt nog bewaard worden? Volgens onze regel wel, want ze ligt niet *in het verleden*. De test `end_today_is_accepted` legt die interpretatie vast.

Je project heeft net zo'n grens in R9; schrijf er een test voor.

## 6.2 Test doubles: testen zonder echte data

De datums kan je testen zonder database. Maar de validation `validateAgency` leest agentschappen uit `ZSI00_I_Agency`. Een test die rekent op agentschap `070001` faalt op een systeem waar dat agentschap niet bestaat, of waar een collega het verwijderde.

Een **test double** vervangt een afhankelijkheid door een versie die de test zelf vult. Voor data gebruik je de **ABAP SQL test environment**: tijdens de test lezen al je `SELECT`s uit tijdelijke tabellen in plaats van uit de echte. Na de test zijn die tabellen weg.

Een test van de validation, via EML op het business object:

Fragment: test class in het tabblad **Test Classes** van de behavior pool. [NIET GETEST] [CONTROLEREN] Of de SQL test environment de writes van een managed business object met draft correct opvangt; anders `cl_botd_txbufdbl_bo_test_env` gebruiken.

```abap
CLASS ltc_promotion DEFINITION FINAL
  FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.

  PRIVATE SECTION.
    CLASS-DATA sql_environment TYPE REF TO if_osql_test_environment.

    CLASS-METHODS class_setup.
    CLASS-METHODS class_teardown.
    METHODS setup.
    METHODS teardown.

    METHODS unknown_agency_is_refused FOR TESTING.
    METHODS existing_agency_is_accepted FOR TESTING.

    METHODS create_promotion
      IMPORTING agency_id     TYPE /dmo/agency_id
      RETURNING VALUE(failed) TYPE abap_bool.
ENDCLASS.

CLASS ltc_promotion IMPLEMENTATION.
  METHOD class_setup.
    sql_environment = cl_osql_test_environment=>create(
      i_dependency_list = VALUE #( ( 'ZSI00_I_AGENCY' )
                                   ( 'ZSI00_PROMO' )
                                   ( 'ZSI00_PROMO_D' )
                                   ( 'ZSI00_PROMOAIR' )
                                   ( 'ZSI00_PROMOAIR_D' ) ) ).
  ENDMETHOD.

  METHOD class_teardown.
    sql_environment->destroy( ).
  ENDMETHOD.

  METHOD setup.
    sql_environment->clear_doubles( ).
    DATA agencies TYPE STANDARD TABLE OF zsi00_i_agency WITH EMPTY KEY.
    agencies = VALUE #( ( AgencyId = '070001' Name = 'Testagentschap' City = 'Brussel' ) ).
    sql_environment->insert_test_data( agencies ).
  ENDMETHOD.

  METHOD teardown.
    ROLLBACK ENTITIES.
  ENDMETHOD.

  METHOD create_promotion.
    MODIFY ENTITIES OF zsi00_r_promotion
      ENTITY Promotion
        CREATE FIELDS ( AgencyId ValidFrom ValidTo DiscountPct )
        WITH VALUE #( ( %cid        = 'PROMO1'
                        %is_draft   = if_abap_behv=>mk-off
                        AgencyId    = agency_id
                        ValidFrom   = '20990101'
                        ValidTo     = '20990131'
                        DiscountPct = '10.00' ) ).

    COMMIT ENTITIES RESPONSE OF zsi00_r_promotion
      FAILED DATA(commit_failed).

    failed = xsdbool( commit_failed IS NOT INITIAL ).
  ENDMETHOD.

  METHOD unknown_agency_is_refused.
    cl_abap_unit_assert=>assert_true( create_promotion( '999999' ) ).
  ENDMETHOD.

  METHOD existing_agency_is_accepted.
    cl_abap_unit_assert=>assert_false( create_promotion( '070001' ) ).
  ENDMETHOD.
ENDCLASS.
```

Het patroon heeft vier stappen, die je in elke test terugziet:

1. **Arrange** (klaarzetten): de test environment met precies de data die deze test nodig heeft.
2. **Act** (uitvoeren): één bewerking op het business object.
3. **Assert** (controleren): is het resultaat wat je verwacht?
4. **Opruimen**: `ROLLBACK ENTITIES` en `clear_doubles( )`, zodat de volgende test schoon begint.

`%is_draft = if_abap_behv=>mk-off` maakt meteen een actieve promotie, zonder concept. Zo loopt de validation bij `COMMIT ENTITIES`. De datums liggen in 2099, zodat `validateDates` de test nooit doet falen omdat de einddatum intussen voorbij is.

::: warning
**Let op:** een test die slaagt omdat de juiste data toevallig in het systeem staat, bewijst niets. Laat elke test zijn eigen data klaarzetten.
:::

## 6.3 abapGit: ABAP in een Git-repository

In andere vakken staat je code in een Git-repository: je kan teruggaan naar een vorige versie, wijzigingen vergelijken en samenwerken via branches. ABAP-code staat in het SAP-systeem, niet in bestanden. Hoe krijg je die in Git?

**abapGit** is een open-sourcetool die ABAP-objecten omzet naar bestanden en terug. Een class wordt een `.abap`-bestand, een CDS view een `.ddls.asddls`-bestand, een tabel een XML-bestand. Die bestanden zet abapGit in een gewone Git-repository, bijvoorbeeld op GitHub.

Op het labosysteem is abapGit geïnstalleerd. Je gebruikt het vanuit Eclipse met de abapGit-plug-in. [CONTROLEREN] Installatie van abapGit op het labosysteem en de werking van de ADT-plug-in met ABAP Platform 2025.

1. Installeer de plug-in via **Help > Install New Software…** met het adres `https://eclipse.abapgit.org/updatesite/`.
2. Maak op GitHub een lege, private repository voor je team.
3. Maak op GitHub een **personal access token** met rechten op die repository.
4. Open in Eclipse **Window > Show View > Other… > ABAP > abapGit Repositories**.
5. Klik op **Link New abapGit Repository**. Vul de URL van je repository in en kies het package van je team.
6. Klik met de rechtermuisknop op de repository en kies **Stage and Push**. Selecteer de objecten, schrijf een commitbericht en push. Je ziet de bestanden nu op GitHub.

Na elke werksessie push je je wijzigingen. Commit kleine stappen met een duidelijk bericht: *Validation voor de datums toegevoegd* zegt meer dan *update*.

abapGit werkt anders dan Git in een gewone code-editor. De **waarheid** staat in het SAP-systeem, niet in je bestanden. Werken twee teamleden tegelijk aan dezelfde class, dan wijzigen ze hetzelfde object op de server. Spreek daarom af wie aan welk object werkt, en gebruik Git vooral als geschiedenis en als oplevering.

::: info
Grote SAP-klanten gebruiken naast abapGit ook **gCTS** (git-enabled Change and Transport System), de Git-integratie van SAP zelf. Het principe is hetzelfde: ABAP-objecten als bestanden in een repository.
:::

## 6.4 Het projectdossier: keuzes samenbrengen

Je project werkt. Over een jaar neemt een ander team het over. Wat hebben ze nodig om binnen een dag aan de slag te kunnen?

Het **projectdossier** beantwoordt drie vragen: wat doet het, hoe zit het in elkaar, en waarom is het zo gebouwd. Het staat in dezelfde repository als de code, in Markdown.

```text
README.md            Wat het project doet en hoe je het start
fit-gap.md           P1: requirements tegenover de standaard
adr/
  001-uitbreidingsroute.md
  002-integratie.md
  003-….md
datamodel.md         Tabellen, views en associations
handleiding.md       Hoe een medewerker de app en het portaal gebruikt
beperkingen.md       Wat niet af is, en wat in productie anders moet
bijdragen.md         Wie bouwde wat
src/                 ABAP-objecten (abapGit)
portal/              Het klantenportaal
```

Een paar tips per onderdeel:

- **README.md** begint met één zin over wat het project doet. Daarna: welke objecten je in welke volgorde bekijkt, en hoe je de app en het portaal start.
- **ADR's**: 3 tot 5, voor de keuzes die er echt toe doen. Kandidaten: de uitbreidingsroute, UUID en volgnummer, draft of niet, de aparte Web API, waar de kost berekend wordt. Een ADR over de kleur van een knop is geen ADR.
- **datamodel.md**: een schets is genoeg. Toon welke data van jou is en welke je leest uit `/DMO/`.
- **beperkingen.md**: eerlijk zijn levert punten op. Het volgnummer met `MAX + 1` ([4.6](./module-4.md#4_6)) is een goed voorbeeld van een beperking die je kent en kan uitleggen.
- **bijdragen.md**: per teamlid de objecten die hij of zij bouwde. Bij de verdediging stellen we vragen over die objecten.

Bereid de verdediging voor door elkaars ADR's te lezen en elkaar de vraag te stellen: *waarom niet het andere alternatief?* Kan je dat beantwoorden voor je eigen objecten, dan ben je klaar.

## Samenvatting

- **ABAP Unit** voert tests uit die het resultaat van je code vergelijken met wat je verwacht; je start ze met Ctrl+Shift+F10.
- Logica die los staat van de database en van RAP, test je het makkelijkst. Geef *vandaag* mee als parameter.
- Een **test double** vervangt een afhankelijkheid; de **ABAP SQL test environment** vervangt tabellen en views door tijdelijke versies.
- Een test volgt **arrange, act, assert** en ruimt daarna op.
- **abapGit** zet ABAP-objecten om naar bestanden in een Git-repository; de waarheid blijft het SAP-systeem.
- Het **projectdossier** legt vast wat het project doet, hoe het in elkaar zit en waarom.

## Veelgemaakte fouten

- **Tests die afhangen van data die toevallig in het systeem staat.** Ze slagen bij jou en falen bij de docent. Zet je eigen testdata klaar.
- **Alleen het geslaagde scenario testen.** Een validation test je vooral met data die ze moet weigeren.
- **De systeemdatum gebruiken in een test.** Een test met *vandaag* slaagt vandaag en faalt over drie weken. Geef de datum mee, of kies datums ver in de toekomst.
- **Eén grote commit aan het einde.** Dan is de geschiedenis nutteloos, en een fout in de laatste week kost je alles.
- **ADR's achteraf schrijven om een keuze te rechtvaardigen.** Bij de verdediging merken we dat aan alternatieven die nooit serieus overwogen zijn.

::: oefening
[Projectstap P7 Oplevering](./project.md#p7)
:::
