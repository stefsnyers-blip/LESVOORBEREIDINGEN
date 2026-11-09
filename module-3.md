---
module: 3
order: 1
moduleTitle: "3: Datastructuren in SAP"
published: true
---

# Module 3: Datastructuren in SAP

Een SAP-systeem bevat duizenden tabellen vol bedrijfsdata. In deze module leer je die data vinden, lezen en aanvullen met je eigen structuren, zonder ze te kopiëren. Met wat je hier leert, bouw je het datamodel van je project. Dat is een eigen tabel met views die ze verbinden met de bestaande reizen en klanten.

### Leerdoelen

Na deze module kan je:

- een database table definiëren met domains en data elements;
- bestaande data bevragen met de Data Preview, de SQL Console en ABAP SQL;
- uitleggen waarom je data leest via CDS views en niet rechtstreeks uit tabellen;
- een CDS view entity schrijven met associations en annotations;
- je eigen data koppelen aan bestaande views zonder ze te dupliceren.

::: info
**De voorbeeldcase.** Reisbureau Zuidwind laat zijn agentschappen promoties aanbieden: een korting op reizen met bepaalde airlines, gedurende een periode. Wim van de marketingafdeling plant een promotie, start ze en beëindigt ze.

Vanaf deze module bouwen we die toepassing stap voor stap, met teamnummer `00`. Je project gebruikt dezelfde technieken voor een andere toepassing: de wijzigingsaanvragen.
:::

## 3.1 Het ABAP Dictionary: tables, data elements en domains

In een gewone relationele database maak je een tabel met `CREATE TABLE` en geef je elke kolom een type als `VARCHAR(20)`. Het systeem weet dan dat de kolom tekst bevat, maar niet wat die tekst betekent. Is `ABC123` een klantnummer, een reisnummer of een wachtwoord?

SAP voegt die betekenis toe in het **ABAP Dictionary** (woordenboek): de plaats waar alle datadefinities van het systeem staan. Een kolom krijgt er geen kaal type, maar een type in twee lagen.

Een **domain** (domein) legt het technische type vast en eventueel de toegelaten waarden. Voorbeeld: `ZSI00_PROMO_STATUS` is `CHAR 1` met de vaste waarden `P`, `A` en `E`.

Een **data element** (gegevenselement) geeft er betekenis aan: een naam en labels in verschillende lengtes en talen. Het data element `ZSI00_PROMO_STATUS` gebruikt het gelijknamige domain en heeft als label *Status*. Overal waar een kolom dit data element gebruikt, toont een Fiori-app automatisch het juiste label en de toegelaten waarden.

Domains en data elements maak je in ADT met een formulier: **File > New > Other… > ABAP > Dictionary**. Voor de promoties maken we er deze:

| Object | Naam | Type | Label of vaste waarden |
| --- | --- | --- | --- |
| Domain | `ZSI00_PROMO_STATUS` | `CHAR 1` | `P` Gepland, `A` Actief, `E` Beëindigd |
| Data element | `ZSI00_PROMO_STATUS` | Domain `ZSI00_PROMO_STATUS` | Status |
| Data element | `ZSI00_PROMO_ID` | `NUMC 8` | Promotie |
| Data element | `ZSI00_DISCOUNT_PCT` | `DEC 5,2` | Korting (%) |

Een data element heeft niet altijd een eigen domain nodig. Een volgnummer heeft geen vaste waarden, dus `ZSI00_PROMO_ID` krijgt rechtstreeks het type `NUMC 8`.

Een **database table** gebruikt data elements voor haar kolommen. In ADT schrijf je een tabel als broncode:

[NIET GETEST]

```abap
@EndUserText.label : 'Promoties van agentschappen'
@AbapCatalog.enhancement.category : #NOT_EXTENSIBLE
@AbapCatalog.tableCategory : #TRANSPARENT
@AbapCatalog.deliveryClass : #A
@AbapCatalog.dataMaintenance : #RESTRICTED
define table zsi00_promo {
  key client            : abap.clnt not null;
  key promo_uuid        : sysuuid_x16 not null;
  promo_id              : zsi00_promo_id;
  agency_id             : /dmo/agency_id;
  description           : /dmo/description;
  valid_from            : /dmo/begin_date;
  valid_to              : /dmo/end_date;
  discount_pct          : zsi00_discount_pct;
  @Semantics.amount.currencyCode : 'zsi00_promo.currency_code'
  max_discount          : abap.curr(16,2);
  currency_code         : abap.cuky;
  status                : zsi00_promo_status;
  created_by            : abp_creation_user;
  created_at            : abp_creation_tstmpl;
  last_changed_by       : abp_lastchange_user;
  last_changed_at       : abp_lastchange_tstmpl;
  local_last_changed_at : abp_locinst_lastchange_tstmpl;
}
```

Een paar dingen vallen op:

- De eerste sleutelkolom is altijd `client`. Zo staat de data van elke client in dezelfde tabel, maar zie je alleen die van je eigen client ([1.2](./module-1.md#1_2)).
- De sleutel is een **UUID** (universally unique identifier): een technische sleutel van 16 bytes die nooit botst. Het leesbare volgnummer `promo_id` is een gewone kolom. In [4.4](./module-4.md#4_4) zie je waarom.
- `agency_id` gebruikt het data element van `/DMO/`. Zo heeft je kolom hetzelfde type en dezelfde labels als het agentschap zelf.
- Een bedrag (`abap.curr`) hoort altijd bij een munteenheid (`abap.cuky`). De annotation `@Semantics.amount.currencyCode` legt die koppeling. `max_discount` is de hoogste korting in euro die één reis kan krijgen.
- De vijf laatste kolommen zijn **administrative fields** (beheervelden): wie maakte de rij, wanneer, en wie wijzigde ze het laatst. RAP vult ze later automatisch in.

Activeer eerst de domains, dan de data elements, dan de tabel. Een object activeer je pas als alles waar het van afhangt al actief is.

::: warning
**Let op:** de naam van een tabel telt hoogstens 16 tekens. Hou er rekening mee, want in module 4 krijgt elke tabel er een zus bij met `_D` achteraan.
:::

## 3.2 Data verkennen in ADT: Data Preview en SQL Console

Je wil weten welke data er in het systeem staat voor je er een view op schrijft. Moet je daarvoor eerst een programma schrijven?

Nee. ADT heeft twee hulpmiddelen om data te bekijken.

De **Data Preview** toont de inhoud van een tabel of view. Open de tabel `/DMO/TRAVEL` met **Ctrl+Shift+A** en druk op **F8**. Je ziet de eerste 100 rijen. Met **Add filter** filter je op een kolom, en via het menu van een kolomkop sorteer je.

De **SQL Console** laat je een query in ABAP SQL schrijven en uitvoeren. Open ze vanuit de Data Preview met de knop **SQL Console**. [CONTROLEREN] Plaats van de knop in de huidige ADT-versie.

Een voorbeeld: welke klanten hebben de meeste reizen?

[NIET GETEST]

```sql
SELECT FROM /dmo/travel
  FIELDS customer_id, COUNT( * ) AS travel_count
  GROUP BY customer_id
  ORDER BY travel_count DESCENDING
```

De syntax lijkt op SQL uit het databasevak, met twee verschillen. `FIELDS` staat na `FROM`, zodat ADT de kolomnamen kan aanvullen zodra het de tabel kent. En tussen de haakjes van een functie zoals `COUNT( * )` staan spaties.

::: tip
De SQL Console voert alleen `SELECT` uit. Je kan er geen data mee wijzigen, dus je kan er niets mee stukmaken. Probeer gerust.
:::

Dit betekent voor jou: verken eerst de data in de SQL Console, en schrijf pas daarna een view of een class. Zo weet je welke kolommen gevuld zijn en welke waarden ze bevatten.

## 3.3 ABAP SQL: data lezen vanuit code

In de SQL Console zie je het resultaat op je scherm. In een programma wil je het resultaat verder gebruiken: filteren, tonen of controleren. Daarvoor schrijf je de query in ABAP.

We gebruiken een class met de interface `if_oo_adt_classrun`. Die interface heeft één method, `main`, die ADT uitvoert als je op **F9** drukt. Het object `out` schrijft naar de console. Dat is handig om iets uit te proberen; in een echte toepassing gebruik je het niet.

[NIET GETEST]

```abap
CLASS zsi00_cl_travel_overview DEFINITION
  PUBLIC FINAL CREATE PUBLIC.

  PUBLIC SECTION.
    INTERFACES if_oo_adt_classrun.
ENDCLASS.

CLASS zsi00_cl_travel_overview IMPLEMENTATION.
  METHOD if_oo_adt_classrun~main.
    DATA(today) = cl_abap_context_info=>get_system_date( ).

    SELECT FROM /dmo/travel
      FIELDS travel_id, agency_id, begin_date, total_price, currency_code
      WHERE begin_date >= @today
      ORDER BY begin_date
      INTO TABLE @DATA(travels)
      UP TO 5 ROWS.

    out->write( travels ).
  ENDMETHOD.
ENDCLASS.
```

De algemene vorm van een `SELECT` in ABAP:

```abap
SELECT FROM <data source>
  FIELDS <field list>
  WHERE <condition>
  INTO TABLE @DATA(<internal table>).
```

- Een **host variable** (variabele uit het programma) in een query krijgt een `@` ervoor: `@today`. Zo weet ABAP dat het geen kolom is.
- `INTO TABLE @DATA(travels)` maakt een **internal table** (interne tabel): een tabel in het geheugen van je programma. Het type leidt ABAP af uit de kolommen.
- `cl_abap_context_info=>get_system_date( )` geeft de datum van vandaag. In klassieke ABAP gebruik je daarvoor `sy-datum`, maar in ABAP Cloud is de class de vrijgegeven manier.

Schrijven gaat met `INSERT`, `UPDATE`, `MODIFY` en `DELETE`. Een fragment uit de class die testdata voor de promoties aanmaakt: drie promoties voor de eerste drie agentschappen.

Fragment: werkt enkel binnen de method `main` van een class zoals hierboven. [NIET GETEST]

```abap
SELECT FROM /dmo/agency
  FIELDS agency_id
  ORDER BY agency_id
  INTO TABLE @DATA(agencies)
  UP TO 3 ROWS.

DATA promotions TYPE STANDARD TABLE OF zsi00_promo WITH EMPTY KEY.
DATA(today) = cl_abap_context_info=>get_system_date( ).
GET TIME STAMP FIELD DATA(now).

LOOP AT agencies INTO DATA(agency).
  TRY.
      APPEND VALUE #( promo_uuid    = cl_system_uuid=>create_uuid_x16_static( )
                      promo_id      = sy-tabix
                      agency_id     = agency-agency_id
                      description   = |Voorjaarsactie { sy-tabix }|
                      valid_from    = today + 7
                      valid_to      = today + 37
                      discount_pct  = '10.00'
                      max_discount  = '150.00'
                      currency_code = 'EUR'
                      status        = 'P'
                      created_by    = cl_abap_context_info=>get_user_technical_name( )
                      created_at    = now ) TO promotions.
    CATCH cx_uuid_error.
      out->write( 'Kon geen UUID aanmaken.' ).
      RETURN.
  ENDTRY.
ENDLOOP.

DELETE FROM zsi00_promo.
INSERT zsi00_promo FROM TABLE @promotions.
out->write( |{ lines( promotions ) } promoties aangemaakt.| ).
```

Bij een datum mag je dagen optellen: `today + 7` is volgende week. `sy-tabix` is het volgnummer van de huidige rij in de `LOOP`.

::: warning
**Let op:** vanaf module 4 schrijf je nooit meer rechtstreeks met `INSERT` in je tabel. Dan gaat elke wijziging via het business object, zodat de controles altijd uitgevoerd worden. Een class voor testdata is de enige uitzondering.
:::

## 3.4 CDS view entities: een view op je data

De app voor de promoties moet bij elke promotie de naam en de stad van het agentschap tonen. Je kan in elke class een `SELECT` met een join schrijven, maar dan schrijf je dezelfde join tien keer, en tien keer net iets anders.

Een **CDS view entity** is een view die je één keer definieert en overal hergebruikt: in ABAP-code, in andere views, in services. In tegenstelling tot een gewone databaseview draagt een CDS view ook betekenis: labels, bedragen met munteenheid, relaties met andere views.

[NIET GETEST]

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Agentschap'
define view entity ZSI00_I_Agency
  as select from /dmo/agency
{
  key agency_id     as AgencyId,
      name          as Name,
      street        as Street,
      postal_code   as PostalCode,
      city          as City,
      country_code  as CountryCode,
      email_address as EmailAddress
}
```

Algemene vorm:

```abap
@AccessControl.authorizationCheck: <check>
@EndUserText.label: '<label>'
define view entity <view name>
  as select from <data source>
{
  key <column> as <ElementName>,
      <column> as <ElementName>
}
```

- De kolommen van een view heten **elements**. Je geeft ze een naam in CamelCase. Zo zie je meteen of je met een tabel (`postal_code`) of een view (`PostalCode`) werkt.
- `@AccessControl.authorizationCheck` zegt of de view een autorisatiecontrole heeft. In dit vak zetten we ze op `#NOT_REQUIRED`; in module 4 regelen we de rechten op het business object.
- Een view heeft altijd een sleutel. Die bepaalt wat één rij is.

Je maakt een view in ADT met **File > New > Other… > Core Data Services > Data Definition**. Kies het sjabloon **Define View Entity**.

Waarom niet rechtstreeks uit de tabel lezen? Omdat SAP zijn tabellen mag wijzigen, maar zijn vrijgegeven views stabiel houdt ([2.5](./module-2.md#2_5)). In een S/4HANA-systeem lees je verkooporders dus via `I_SalesOrder`, niet via de tabel eronder. In het labosysteem spelen de `/DMO/`-tabellen de rol van standaarddata; jouw views zijn de laag waarlangs de rest van je toepassing ze leest.

## 3.5 Associations: relaties die je pas volgt als je ze nodig hebt

Elke vlucht wordt uitgevoerd door een airline. In SQL schrijf je daarvoor een join. Maar niet elke query heeft de airline nodig, en een join die je niet nodig hebt, kost toch rekentijd.

Een **association** (associatie) is een relatie tussen twee views die je definieert, maar die pas een join wordt als iemand ze effectief gebruikt. We maken eerst een view op de airlines:

[NIET GETEST]

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Airline'
define view entity ZSI00_I_Airline
  as select from /dmo/carrier
{
  key carrier_id    as AirlineId,
      name          as Name,
      currency_code as CurrencyCode
}
```

Een **connection** (verbinding) in `/DMO/` is een vaste vlucht tussen twee luchthavens, zoals elke dag van Frankfurt naar New York. De view op de verbindingen krijgt een association naar de airline:

[NIET GETEST]

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Verbinding'
define view entity ZSI00_I_Connection
  as select from /dmo/connection
  association [1..1] to ZSI00_I_Airline as _Airline
    on $projection.AirlineId = _Airline.AirlineId
{
  key carrier_id      as AirlineId,
  key connection_id   as ConnectionId,
      airport_from_id as AirportFromId,
      airport_to_id   as AirportToId,
      departure_time  as DepartureTime,
      arrival_time    as ArrivalTime,

      _Airline
}
```

Algemene vorm:

```abap
association [<min>..<max>] to <target view> as _<Name>
  on $projection.<element> = _<Name>.<element>
```

- De **cardinality** (kardinaliteit) `[1..1]` zegt: bij elke verbinding hoort precies één airline. `[0..*]` betekent nul of meer, zoals de verbindingen van één airline.
- Een association begint volgens afspraak met een underscore: `_Airline`.
- `$projection` verwijst naar de elements van je eigen view.
- Door `_Airline` in de lijst van elements op te nemen, **expose** (stel je beschikbaar) je de association. Wie jouw view gebruikt, kan ze dan volgen.

Een association volg je met een **path expression** (padexpressie). In een andere view schrijf je `_Connection._Airline.Name`. In ABAP SQL gebruik je een backslash: `\_Airline-Name`.

[NIET GETEST]

```sql
SELECT FROM zsi00_i_connection
  FIELDS ConnectionId, AirportFromId, AirportToId, \_Airline-Name
  WHERE AirportFromId = 'FRA'
```

Pas als een query `_Airline` gebruikt, voert de database de join uit. Een query die alleen `ConnectionId` en `AirportToId` opvraagt, raakt de tabel met airlines niet aan.

::: warning
**Let op:** een path expression naar een association met cardinality `[0..*]` kan rijen verdubbelen. Volg je vanuit een airline haar verbindingen, dan verschijnt de airline één keer per verbinding. Gebruik zo'n pad alleen als je dat wil.
:::

## 3.6 Het virtual data model: basic, composite en consumption views

SAP S/4HANA bevat tienduizenden CDS views. Hoe vind je daarin de weg, en waar zet je je eigen views?

SAP ordent zijn views in lagen, samen het **virtual data model** (VDM). Elke laag bouwt op de vorige verder.

| Laag | Prefix | Wat erin staat | Voorbeeld |
| --- | --- | --- | --- |
| Basic views | `I_` | Eén view per entiteit, rechtstreeks op de tabellen, met leesbare namen | `I_SalesOrder`, `I_Customer` |
| Composite views | `I_` | Combinaties van basic views voor een bepaald gebruik | `I_SalesOrderItemCube` |
| Transactional views | `R_` | De basis van een business object (module 4) | `R_SalesOrderTP` |
| Consumption views | `C_` | Afgestemd op één app of één rapport | `C_SalesOrderManage` |
| API views | `A_` | Aangeboden als API voor andere systemen | `A_SalesOrder` |

Basic views zijn het meest herbruikbaar. Een consumption view is gemaakt voor één app en verandert mee met die app.

De promoties volgen dezelfde lagen, met het teamnummer in de naam:

| View | Laag | Gebouwd in |
| --- | --- | --- |
| `ZSI00_I_Agency`, `ZSI00_I_Airline`, `ZSI00_I_Connection` | Basic, op `/DMO/` | Deze module |
| `ZSI00_I_Promotion` | Basic, op de eigen tabel | Deze module |
| `ZSI00_R_Promotion`, `ZSI00_R_PromoAirline` | Transactional | Module 4 |
| `ZSI00_C_Promotion`, `ZSI00_C_PromoAirline` | Consumption, voor de Fiori-app | Module 4 |
| `ZSI00_A_Promotion` | API, voor andere systemen | Module 5 |

Dit betekent voor jou: zoek in een S/4HANA-systeem eerst een `I_`-view met de data die je nodig hebt. Bouw je eigen views erbovenop, nooit onder de standaardviews door op de tabellen. Gebruik in je project dezelfde lagen.

## 3.7 Annotations: metadata bij een view

Een Fiori-app toont het bedrag `150.00`. In welke munt? En wat betekent de statuscode `P`? De app weet dat alleen als de view het vertelt.

Een **annotation** (annotatie) is metadata bij een view of een element. Ze begint met `@`. Je zag er al enkele:

| Annotation | Betekenis |
| --- | --- |
| `@EndUserText.label: 'Agentschap'` | Het label van de view voor gebruikers |
| `@AccessControl.authorizationCheck: #NOT_REQUIRED` | Geen autorisatiecontrole op deze view |
| `@Semantics.amount.currencyCode: 'CurrencyCode'` | Dit element is een bedrag; de munt staat in `CurrencyCode` |
| `@ObjectModel.text.association: '_StatusText'` | De leesbare tekst bij deze code vind je via een association |

Voor de status van een promotie wil je *Gepland* tonen in plaats van `P`. De vaste waarden van je domain lees je met de vrijgegeven table function `DDCDS_CUSTOMER_DOMAIN_VALUE_T`:

[NIET GETEST] [CONTROLEREN] Namen van de parameter en de kolommen van `DDCDS_CUSTOMER_DOMAIN_VALUE_T` op ABAP Platform 2025.

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Status van een promotie'
define view entity ZSI00_I_PromoStatusText
  as select from DDCDS_CUSTOMER_DOMAIN_VALUE_T( p_domain_name: 'ZSI00_PROMO_STATUS' )
{
  key value_low as Status,
      @Semantics.text: true
      text      as StatusText
}
where language = $session.system_language
```

In de view van de promoties koppel je de tekst aan de code:

Fragment: werkt enkel als onderdeel van `ZSI00_I_Promotion` in 3.8. [NIET GETEST]

```abap
  association [0..1] to ZSI00_I_PromoStatusText as _StatusText
    on $projection.Status = _StatusText.Status
{
  // ...
  @ObjectModel.text.association: '_StatusText'
  status as Status,
  // ...
  _StatusText
}
```

Annotations die de gebruikersinterface beschrijven, zoals welke kolommen in een lijst staan, zet je niet in de view zelf. Die komen in een aparte **metadata extension** ([4.10](./module-4.md#4_10)). Zo blijft de view herbruikbaar voor andere apps.

## 3.8 Bestaande views hergebruiken en uitbreiden

Je hebt nu views op de agentschappen en de airlines. De laatste stap is je eigen data eraan koppelen, zonder iets te kopiëren.

De view op de eigen tabel krijgt een association naar het agentschap. Via `_Agency` bereik je zo de naam en de stad, zonder ze op te slaan.

[NIET GETEST]

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Promotie'
define view entity ZSI00_I_Promotion
  as select from zsi00_promo
  association [1..1] to ZSI00_I_Agency          as _Agency
    on $projection.AgencyId = _Agency.AgencyId
  association [0..1] to ZSI00_I_PromoStatusText as _StatusText
    on $projection.Status = _StatusText.Status
{
  key promo_uuid    as PromoUuid,
      promo_id      as PromoId,
      agency_id     as AgencyId,
      description   as Description,
      valid_from    as ValidFrom,
      valid_to      as ValidTo,
      discount_pct  as DiscountPct,
      @Semantics.amount.currencyCode: 'CurrencyCode'
      max_discount  as MaxDiscount,
      currency_code as CurrencyCode,
      @ObjectModel.text.association: '_StatusText'
      status        as Status,
      _Agency.Name  as AgencyName,
      _Agency.City  as AgencyCity,

      _Agency,
      _StatusText
}
```

`AgencyName` en `AgencyCity` staan niet in je tabel. De view haalt ze op het moment van de query uit `/DMO/`. Verhuist een agentschap, dan toont je view meteen de nieuwe stad.

Soms wil je de omgekeerde richting: vanuit het agentschap naar zijn promoties. De view `ZSI00_I_Agency` is van jou, dus je kan de association er rechtstreeks aan toevoegen. Maar een standaardview van SAP kan je niet wijzigen. Daarvoor bestaat een **view extension**: een apart object dat een view uitbreidt zonder ze aan te passen.

[NIET GETEST] [CONTROLEREN] Of een association in een extension toegelaten is op ABAP Platform 2025, en of de cyclus tussen agentschap en promotie activeert.

```abap
extend view entity ZSI00_I_Agency with
  association [0..*] to ZSI00_I_Promotion as _Promotions
    on $projection.AgencyId = _Promotions.AgencyId
{
  _Promotions
}
```

Op een S/4HANA-systeem mag je dat alleen bij views met het release contract C0 ([2.5](./module-2.md#2_5)). Het principe is hetzelfde als bij clean core: je voegt toe naast de standaard, je wijzigt de standaard niet.

## Samenvatting

- Het **ABAP Dictionary** bevat alle datadefinities. Een **domain** legt het technische type en de toegelaten waarden vast, een **data element** de betekenis en de labels.
- Een **database table** heeft `client` als eerste sleutel; een **UUID** is een goede technische sleutel. Een tabelnaam telt hoogstens 16 tekens.
- Met de **Data Preview** (F8) en de **SQL Console** verken je data zonder code.
- In **ABAP SQL** krijgt een **host variable** een `@`, en `INTO TABLE @DATA( )` maakt een **internal table**.
- Een **CDS view entity** definieer je één keer en hergebruik je overal; de kolommen heten **elements**.
- Een **association** is een relatie die pas een join wordt als je ze volgt met een **path expression**.
- Het **virtual data model** ordent views in lagen: `I_`, `R_`, `C_`, `A_`.
- **Annotations** voegen metadata toe, zoals bedragen met munteenheid en teksten bij codes.
- Een **view extension** breidt een view uit zonder ze te wijzigen.

## Veelgemaakte fouten

- **Data kopiëren in plaats van ernaar te verwijzen.** Een kolom `agency_name` in de tabel met promoties is na de eerste naamswijziging fout. Een association naar het agentschap blijft juist.
- **Een association behandelen als een join die altijd gebeurt.** Ze kost niets zolang niemand ze gebruikt. Expose ze dus gerust.
- **Een generiek type gebruiken voor een code.** `abap.char(1)` voor een status zegt niets; een data element met een domain geeft labels en toegelaten waarden.
- **Een bedrag zonder munteenheid.** Zonder `@Semantics.amount.currencyCode` weet geen enkele app of `150.00` euro of yen is.
- **Een path expression naar `[0..*]` in een lijst.** Elke airline verschijnt dan één keer per verbinding.
- **Een tabelnaam van meer dan 16 tekens.** ADT weigert de tabel, en de naam van de drafttabel in module 4 wordt nog langer.

::: oefening
[Projectstap P3 Datamodel](./project.md#p3)
:::
