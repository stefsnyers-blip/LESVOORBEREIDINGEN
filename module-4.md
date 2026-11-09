---
module: 4
order: 1
moduleTitle: "4: Nieuwe functionaliteit met RAP"
published: true
---

# Module 4: Nieuwe functionaliteit met RAP

Wat SAP niet standaard biedt, bouw je zelf, maar wel volgens de regels van SAP. In deze module bouw je een volledige transactionele applicatie met het ABAP RESTful Application Programming Model. Ze bestaat uit een business object met bedrijfsregels, een service en een Fiori-app. We bouwen de app voor de promoties uit module 3; in je project pas je dezelfde technieken toe op de wijzigingsaanvragen.

### Leerdoelen

Na deze module kan je:

- een managed business object opbouwen uit een root entity en child entities;
- in een behavior definition vastleggen welke bewerkingen en velden toegelaten zijn;
- bedrijfsregels uit een analyse implementeren als validations, determinations en actions;
- draft inschakelen en uitleggen wanneer je het nodig hebt;
- een business object aanbieden als OData-service met een Fiori elements-app;
- toegang tot bewerkingen beperken met authorizations.

## 4.1 RAP: de bouwstenen in een overzicht

Een app om promoties te beheren moet data lezen en bewaren, controles uitvoeren, statussen bijhouden, meerdere gebruikers tegelijk aankunnen en een gebruikersinterface hebben. Dat is veel code als je het allemaal zelf schrijft.

Het **ABAP RESTful Application Programming Model** (RAP) is het programmeermodel van SAP voor zulke apps. RAP neemt het standaardwerk over: bewaren, vergrendelen, transacties, de OData-service. Jij schrijft alleen wat specifiek is voor je proces.

Een RAP-app bestaat uit lagen. Je bouwt ze van onder naar boven.

| Laag | Object | Wat het doet | Sectie |
| --- | --- | --- | --- |
| Data | Database table | Opslag | [3.1](./module-3.md#3_1), 4.2 |
| Datamodel | Root view entity en child view entities (`R_`) | De structuur van het business object | 4.2 |
| Gedrag | Behavior definition en behavior pool | Toegelaten bewerkingen en bedrijfsregels | 4.3 tot 4.8 |
| Projectie | Projection views (`C_`) en behavior projection | Wat één app ervan te zien krijgt | 4.9 |
| Service | Service definition en service binding | De OData-service | 4.9 |
| UI | Metadata extensions | Hoe de Fiori-app eruitziet | 4.10 |

Het eerste deel (data, datamodel, gedrag) is het **business object**: alles wat altijd geldt, welke app de promoties ook gebruikt. Het tweede deel (projectie, service, UI) bouw je per app. Zo kan je later een tweede app of een API maken op hetzelfde business object, met dezelfde regels.

::: tip
SAP publiceert een volledige oefenreeks met RAP: [RAP100 op GitHub](https://github.com/SAP-samples/abap-platform-rap100). Loopt het vast in je project, vergelijk dan met hoe RAP100 het doet.
:::

## 4.2 Business objects en compositions: root en child

Een promotie geldt voor één of meer airlines: *15 % korting op vluchten van Lufthansa en Singapore Airlines*. Je bewaart nooit een airline-regel zonder promotie, en als je een promotie verwijdert, verdwijnen haar regels mee. Hoe leg je dat vast?

Een business object in RAP is een boom van entiteiten. Bovenaan staat de **root entity**. Daaronder hangen **child entities** via een **composition** (compositie): een association die zegt *dit hoort bij mij en bestaat niet zonder mij*. De promotie is de root, de airline-regel een child.

De child entity krijgt een eigen tabel. De kolom `parent_uuid` verwijst naar de promotie:

[NIET GETEST]

```abap
@EndUserText.label : 'Airlines van een promotie'
@AbapCatalog.enhancement.category : #NOT_EXTENSIBLE
@AbapCatalog.tableCategory : #TRANSPARENT
@AbapCatalog.deliveryClass : #A
@AbapCatalog.dataMaintenance : #RESTRICTED
define table zsi00_promoair {
  key client            : abap.clnt not null;
  key airline_uuid      : sysuuid_x16 not null;
  parent_uuid           : sysuuid_x16;
  airline_id            : /dmo/carrier_id;
  local_last_changed_at : abp_locinst_lastchange_tstmpl;
}
```

De root view entity bouw je op de tabel van de promoties. Ze lijkt op `ZSI00_I_Promotion` uit [3.8](./module-3.md#3_8), met drie verschillen:

[NIET GETEST]

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Promotie'
define root view entity ZSI00_R_Promotion
  as select from zsi00_promo
  composition [0..*] of ZSI00_R_PromoAirline  as _Airlines
  association [1..1] to ZSI00_I_Agency          as _Agency
    on $projection.AgencyId = _Agency.AgencyId
  association [0..1] to ZSI00_I_PromoStatusText as _StatusText
    on $projection.Status = _StatusText.Status
{
  key promo_uuid            as PromoUuid,
      promo_id              as PromoId,
      agency_id             as AgencyId,
      description           as Description,
      valid_from            as ValidFrom,
      valid_to              as ValidTo,
      discount_pct          as DiscountPct,
      @Semantics.amount.currencyCode: 'CurrencyCode'
      max_discount          as MaxDiscount,
      currency_code         as CurrencyCode,
      @ObjectModel.text.association: '_StatusText'
      status                as Status,
      @Semantics.user.createdBy: true
      created_by            as CreatedBy,
      @Semantics.systemDateTime.createdAt: true
      created_at            as CreatedAt,
      @Semantics.user.lastChangedBy: true
      last_changed_by       as LastChangedBy,
      @Semantics.systemDateTime.lastChangedAt: true
      last_changed_at       as LastChangedAt,
      @Semantics.systemDateTime.localInstanceLastChangedAt: true
      local_last_changed_at as LocalLastChangedAt,

      _Airlines,
      _Agency,
      _StatusText
}
```

- Het woord `root` maakt van de view de top van het business object.
- `composition [0..*] of` verbindt de promotie met haar airline-regels.
- De `@Semantics`-annotations bij de beheervelden zorgen dat RAP ze zelf invult bij elke wijziging.

De child view entity verwijst terug naar haar parent met `association to parent`:

[NIET GETEST]

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Airline van een promotie'
define view entity ZSI00_R_PromoAirline
  as select from zsi00_promoair
  association to parent ZSI00_R_Promotion as _Promotion
    on $projection.ParentUuid = _Promotion.PromoUuid
  association [1..1] to ZSI00_I_Airline   as _Airline
    on $projection.AirlineId = _Airline.AirlineId
{
  key airline_uuid          as AirlineUuid,
      parent_uuid           as ParentUuid,
      airline_id            as AirlineId,
      @Semantics.systemDateTime.localInstanceLastChangedAt: true
      local_last_changed_at as LocalLastChangedAt,

      _Promotion,
      _Airline
}
```

De twee views verwijzen naar elkaar, dus geen van beide activeert alleen. Activeer ze samen met **Ctrl+Shift+F3** (alle inactieve objecten).

Merk op welke associations geen composition zijn. `_Agency` en `_Airline` verwijzen naar data die niet van de promotie is. Verwijder je een promotie, dan blijven het agentschap en de airline bestaan.

## 4.3 De behavior definition: wat mag er met je data

Je views beschrijven hoe een promotie eruitziet. Maar mag je een promotie aanmaken? Wijzigen? Welke velden mag de gebruiker zelf invullen?

Dat legt de **behavior definition** (gedragsdefinitie) vast: een apart object met dezelfde naam als de root view. Je maakt het in ADT met een rechtermuisklik op de root view: **New Behavior Definition**. Eén behavior definition beschrijft het hele business object, root en children samen.

[NIET GETEST]

```abap
managed implementation in class zbp_si00_r_promotion unique;
strict ( 2 );

define behavior for ZSI00_R_Promotion alias Promotion
persistent table zsi00_promo
lock master
authorization master ( instance )
etag master LocalLastChangedAt
{
  create;
  update;
  delete;

  association _Airlines { create; }

  field ( numbering : managed, readonly ) PromoUuid;
  field ( readonly ) PromoId, Status, CreatedBy, CreatedAt,
                     LastChangedBy, LastChangedAt, LocalLastChangedAt;
  field ( mandatory ) AgencyId, ValidFrom, DiscountPct;

  mapping for zsi00_promo
  {
    PromoUuid          = promo_uuid;
    PromoId            = promo_id;
    AgencyId           = agency_id;
    Description        = description;
    ValidFrom          = valid_from;
    ValidTo            = valid_to;
    DiscountPct        = discount_pct;
    MaxDiscount        = max_discount;
    CurrencyCode       = currency_code;
    Status             = status;
    CreatedBy          = created_by;
    CreatedAt          = created_at;
    LastChangedBy      = last_changed_by;
    LastChangedAt      = last_changed_at;
    LocalLastChangedAt = local_last_changed_at;
  }
}

define behavior for ZSI00_R_PromoAirline alias PromoAirline
persistent table zsi00_promoair
lock dependent by _Promotion
authorization dependent by _Promotion
etag master LocalLastChangedAt
{
  update;
  delete;

  association _Promotion;

  field ( numbering : managed, readonly ) AirlineUuid;
  field ( readonly ) ParentUuid, LocalLastChangedAt;
  field ( mandatory ) AirlineId;

  mapping for zsi00_promoair
  {
    AirlineUuid        = airline_uuid;
    ParentUuid         = parent_uuid;
    AirlineId          = airline_id;
    LocalLastChangedAt = local_last_changed_at;
  }
}
```

Regel per regel:

- `managed` betekent dat RAP het bewaren in de database zelf afhandelt. `implementation in class` noemt de **behavior pool**: de class waarin jouw code komt.
- `strict ( 2 )` zet de strengste controles aan. Zo vang je fouten bij het activeren in plaats van bij de gebruiker.
- `alias Promotion` geeft de entiteit een korte naam voor in je code.
- `lock master` vergrendelt een promotie terwijl iemand ze wijzigt. `etag master` detecteert wijzigingen van iemand anders ([5.3](./module-5.md#5_3)).
- `create`, `update` en `delete` zijn de toegelaten **standard operations**.
- `association _Airlines { create; }` laat toe dat je airline-regels aanmaakt via de promotie. Een child maak je nooit los aan, altijd via zijn parent.
- De `field`-regels heten **field control**: welke velden alleen-lezen of verplicht zijn.
- `mapping for` koppelt de elements van de view aan de kolommen van de tabel.

De child heeft geen eigen `lock master` en `authorization master`. Ze volgt haar parent: `lock dependent by _Promotion`. Wie een promotie vergrendelt, vergrendelt ook haar airline-regels.

Activeer de behavior definition. ADT meldt dat de behavior pool ontbreekt: gebruik de quick fix (**Ctrl+1**) om ze aan te maken. ADT maakt ook een lege method `get_instance_authorizations`; die vullen we in 4.11. [CONTROLEREN] Of een lege `get_instance_authorizations` alle bewerkingen toelaat.

Je spreekt een business object vanuit ABAP aan met **EML** (Entity Manipulation Language): `READ ENTITIES` en `MODIFY ENTITIES`. Een test in een classrun-class maakt een promotie met twee airlines aan:

Fragment: werkt enkel binnen de method `main` van een class met `if_oo_adt_classrun`. [NIET GETEST]

```abap
MODIFY ENTITIES OF zsi00_r_promotion
  ENTITY Promotion
    CREATE FIELDS ( AgencyId Description ValidFrom ValidTo DiscountPct )
    WITH VALUE #( ( %cid        = 'PROMO1'
                    AgencyId    = '070001'
                    Description = 'Lentekorting'
                    ValidFrom   = '20270301'
                    ValidTo     = '20270331'
                    DiscountPct = '15.00' ) )
    CREATE BY \_Airlines FIELDS ( AirlineId )
    WITH VALUE #( ( %cid_ref = 'PROMO1'
                    %target  = VALUE #( ( %cid = 'AIR1' AirlineId = 'LH' )
                                        ( %cid = 'AIR2' AirlineId = 'SQ' ) ) ) )
  FAILED DATA(failed)
  REPORTED DATA(reported).

COMMIT ENTITIES RESPONSE OF zsi00_r_promotion
  FAILED DATA(commit_failed)
  REPORTED DATA(commit_reported).

out->write( COND string( WHEN commit_failed IS INITIAL THEN `Bewaard` ELSE `Niet bewaard` ) ).
```

- `%cid` is een tijdelijke naam voor de nieuwe promotie zolang ze nog geen sleutel heeft.
- `CREATE BY \_Airlines` maakt children aan via de composition. `%cid_ref` zegt bij welke nieuwe promotie ze horen.
- `COMMIT ENTITIES` bewaart alles in één transactie: de promotie en haar twee airlines, of niets.

::: warning
**Let op:** `field ( mandatory )` toont een sterretje in de app, maar houdt een lege waarde bij het bewaren niet tegen. Voor een echte controle schrijf je een validation (4.5).
:::

## 4.4 Managed implementation en numbering

Een nieuwe promotie heeft een sleutel nodig. Wie bedenkt die: de gebruiker, de database of jouw code?

Dat heet **numbering** (nummertoekenning). Onze tabel heeft twee nummers, en ze krijgen elk een andere aanpak.

De technische sleutel `PromoUuid` krijgt **managed numbering**: RAP maakt bij het aanmaken zelf een UUID. Dat staat in de regel `field ( numbering : managed, readonly ) PromoUuid;`. De gebruiker ziet dit nummer nooit. De airline-regels krijgen op dezelfde manier een `AirlineUuid`.

Het leesbare volgnummer `PromoId` is geen sleutel, maar een veld zoals de andere. Wim gebruikt het in een e-mail aan de agentschappen: *promotie 12 start maandag*. Je vult het in met een determination (4.6) op het moment dat de promotie bewaard wordt.

Waarom die twee nummers? Een UUID kan RAP meteen maken, zonder de database te raadplegen, ook voor een promotie die nog niet bewaard is. Een volgnummer zonder gaten kan je pas toekennen als zeker is dat de promotie bewaard wordt. Anders verbruikt elke geannuleerde poging een nummer.

In een `managed` business object schrijf je geen `INSERT` of `UPDATE`. Je wijzigt de data met EML, en RAP schrijft alles in één keer naar de database bij `COMMIT ENTITIES`. Daartussen leeft de wijziging in de **transactional buffer** (transactiebuffer). Jouw validations en determinations werken op die buffer, niet op de database.

Dit betekent voor jou: in je behavior pool lees je je eigen data met `READ ENTITIES … IN LOCAL MODE`, niet met `SELECT` op je tabel. Een `SELECT` ziet de wijzigingen in de buffer niet. `IN LOCAL MODE` zegt dat de code van het business object zelf leest. Field control en authorizations gelden dan niet: de regel *status is alleen-lezen* geldt voor de gebruiker, niet voor je eigen logica.

## 4.5 Validations: ongeldige data tegenhouden

Wim plant een promotie die eindigt vóór ze begint. Wanneer moet het systeem dat melden, en hoe?

Een **validation** is een controle die RAP uitvoert vlak voor het bewaren. Faalt ze, dan wordt niets bewaard en krijgt de gebruiker een melding bij het juiste veld. Je declareert een validation in de behavior definition, in het blok van `Promotion`:

Fragment: in de behavior definition. [NIET GETEST]

```abap
validation validateAgency on save { create; field AgencyId; }
validation validateDates on save { create; field ValidFrom, ValidTo; }
validation validateDiscount on save { create; field DiscountPct; }
```

Tussen de accolades staat de **trigger**: wanneer moet de validation lopen? Hier: bij het aanmaken, en bij elke wijziging van de genoemde velden. Gebruik de quick fix om de lege methods in de behavior pool aan te maken.

De implementatie volgt altijd hetzelfde patroon: lezen, controleren, melden.

Fragment: method in de behavior pool `zbp_si00_r_promotion`. [NIET GETEST]

```abap
METHOD validateDates.
  READ ENTITIES OF zsi00_r_promotion IN LOCAL MODE
    ENTITY Promotion
      FIELDS ( ValidFrom ValidTo ) WITH CORRESPONDING #( keys )
    RESULT DATA(promotions).

  DATA(today) = cl_abap_context_info=>get_system_date( ).

  LOOP AT promotions INTO DATA(promotion).
    APPEND VALUE #( %tky        = promotion-%tky
                    %state_area = 'VALIDATE_DATES' ) TO reported-promotion.

    DATA(message) = COND string(
      WHEN promotion-ValidFrom IS INITIAL        THEN `Vul een begindatum in.`
      WHEN promotion-ValidTo < promotion-ValidFrom THEN `De einddatum ligt voor de begindatum.`
      WHEN promotion-ValidTo < today             THEN `De einddatum ligt in het verleden.` ).

    IF message IS NOT INITIAL.
      APPEND VALUE #( %tky = promotion-%tky ) TO failed-promotion.
      APPEND VALUE #( %tky             = promotion-%tky
                      %state_area      = 'VALIDATE_DATES'
                      %msg             = new_message_with_text(
                                           severity = if_abap_behv_message=>severity-error
                                           text     = message )
                      %element-ValidTo = if_abap_behv=>mk-on ) TO reported-promotion.
    ENDIF.
  ENDLOOP.
ENDMETHOD.
```

De belangrijkste onderdelen:

- `keys` bevat de sleutels van de promoties die gecontroleerd moeten worden. Een validation werkt altijd op een reeks promoties tegelijk, nooit op één.
- `%tky` is de **transactional key**: de sleutel plus de informatie of het om een concept of een bewaarde versie gaat (4.8).
- `failed` zegt welke promoties niet bewaard mogen worden. `reported` bevat de meldingen voor de gebruiker.
- `%element-ValidTo` plaatst de melding bij het veld *Geldig tot* in de app.
- `%state_area` groepeert meldingen. De lege regel aan het begin van de loop wist een oude melding van dezelfde validation, zodat een opgeloste fout niet blijft staan.
- `new_message_with_text` maakt een melding met een vaste tekst. In een productiesysteem gebruik je een **message class**, zodat de tekst vertaald kan worden.

`validateAgency` controleert iets anders: bestaat het agentschap? Daarvoor lees je de bestaande agentschappen in één `SELECT` voor alle promoties tegelijk:

Fragment: uit de method `validateAgency`, na de `READ ENTITIES`. [NIET GETEST]

```abap
SELECT FROM zsi00_i_agency
  FIELDS AgencyId
  FOR ALL ENTRIES IN @promotions
  WHERE AgencyId = @promotions-AgencyId
  INTO TABLE @DATA(existing_agencies).

LOOP AT promotions INTO DATA(promotion).
  IF NOT line_exists( existing_agencies[ AgencyId = promotion-AgencyId ] ).
    " ...
  ENDIF.
ENDLOOP.
```

In de lege `IF` vul je `failed` en `reported` zoals in `validateDates`, met `%element-AgencyId`. `validateDiscount` schrijf je volgens hetzelfde patroon: de korting ligt tussen 1 en 50 %.

::: warning
**Let op:** `FOR ALL ENTRIES` met een lege tabel selecteert alle rijen. Controleer daarom eerst of `promotions` gevuld is, met `IF promotions IS INITIAL. RETURN. ENDIF.`
:::

## 4.6 Determinations: waarden automatisch invullen

Een nieuwe promotie moet de status *Gepland* krijgen. En vult Wim geen einddatum in, dan loopt de promotie standaard 30 dagen. Die waarden laat je niet aan het geheugen van de gebruiker over. Wie vult ze in?

Een **determination** (afleiding) berekent waarden op basis van andere waarden. Je declareert ze zoals een validation, met een trigger:

Fragment: in de behavior definition. [NIET GETEST]

```abap
determination setInitialValues on modify { create; }
determination setDefaultValidTo on modify { create; field ValidFrom; }
determination setPromoId on save { create; }
```

Het verschil tussen `on modify` en `on save`:

- `on modify` loopt meteen na de wijziging. De gebruiker ziet het resultaat terwijl hij of zij nog aan het invullen is: de einddatum verschijnt zodra de begindatum ingevuld is.
- `on save` loopt pas vlak voor het bewaren. Dat is de plaats voor het volgnummer: dat ken je pas toe als de promotie effectief bewaard wordt (4.4).

Vaste waarden zet je als constanten in de handler class, zodat je nergens `'P'` of `30` moet onthouden:

Fragment: in de definitie van `lhc_Promotion` in de behavior pool. [NIET GETEST]

```abap
CONSTANTS:
  BEGIN OF promo_status,
    planned TYPE zsi00_promo_status VALUE 'P',
    active  TYPE zsi00_promo_status VALUE 'A',
    ended   TYPE zsi00_promo_status VALUE 'E',
  END OF promo_status,
  default_duration_days TYPE i VALUE 30.
```

Een determination leest de promoties, berekent iets en wijzigt de promoties met `MODIFY ENTITIES … IN LOCAL MODE`. De standaardeinddatum:

Fragment: method in de behavior pool. [NIET GETEST]

```abap
METHOD setDefaultValidTo.
  READ ENTITIES OF zsi00_r_promotion IN LOCAL MODE
    ENTITY Promotion
      FIELDS ( ValidFrom ValidTo ) WITH CORRESPONDING #( keys )
    RESULT DATA(promotions).

  DELETE promotions WHERE ValidFrom IS INITIAL OR ValidTo IS NOT INITIAL.
  IF promotions IS INITIAL.
    RETURN.
  ENDIF.

  MODIFY ENTITIES OF zsi00_r_promotion IN LOCAL MODE
    ENTITY Promotion
      UPDATE FIELDS ( ValidTo )
      WITH VALUE #( FOR promotion IN promotions
                    ( %tky    = promotion-%tky
                      ValidTo = promotion-ValidFrom + default_duration_days ) ).
ENDMETHOD.
```

De `DELETE` haalt de promoties eruit waar niets te doen valt: zonder begindatum, of met een einddatum die Wim zelf koos. Een determination die waarden van de gebruiker overschrijft, is een bron van verbaasde telefoontjes.

Het volgnummer: lees het hoogste nummer in de tabel en tel er één bij.

Fragment: method in de behavior pool. [NIET GETEST]

```abap
METHOD setPromoId.
  READ ENTITIES OF zsi00_r_promotion IN LOCAL MODE
    ENTITY Promotion
      FIELDS ( PromoId ) WITH CORRESPONDING #( keys )
    RESULT DATA(promotions).

  DELETE promotions WHERE PromoId IS NOT INITIAL.
  IF promotions IS INITIAL.
    RETURN.
  ENDIF.

  SELECT SINGLE FROM zsi00_promo
    FIELDS MAX( promo_id )
    INTO @DATA(max_promo_id).

  MODIFY ENTITIES OF zsi00_r_promotion IN LOCAL MODE
    ENTITY Promotion
      UPDATE FIELDS ( PromoId )
      WITH VALUE #( FOR promotion IN promotions INDEX INTO i
                    ( %tky    = promotion-%tky
                      PromoId = max_promo_id + i ) ).
ENDMETHOD.
```

`INDEX INTO i` nummert de promoties in de loop: bewaart iemand er drie tegelijk, dan krijgen ze drie opeenvolgende nummers. `setInitialValues` zet de status op `promo_status-planned` en de munteenheid op `EUR` als die leeg is. Die schrijf je volgens hetzelfde patroon.

::: warning
**Let op:** het hoogste nummer opzoeken en er één bijtellen gaat mis als twee gebruikers op hetzelfde moment bewaren. In een echt systeem gebruik je daarvoor een **number range object**. Voor het project volstaat deze aanpak, maar vermeld de beperking in je dossier.
:::

## 4.7 Actions: starten en beëindigen

Een promotie gaat live. Mag Wim daarvoor de status op *Actief* zetten in een formulier?

Beter niet. Een statuswijziging is een bewuste handeling met eigen regels: starten kan alleen bij een geplande promotie, beëindigen alleen bij een actieve. Een **action** (actie) is zo'n handeling met een naam. In de app wordt ze een knop.

Fragment: in de behavior definition. [NIET GETEST]

```abap
action ( features : instance ) startPromotion result [1] $self;
action ( features : instance ) endPromotion result [1] $self;
```

`result [1] $self` betekent dat de action de gewijzigde promotie teruggeeft, zodat de app ze meteen opnieuw toont. `features : instance` zegt dat per promotie bepaald wordt of de action beschikbaar is.

Fragment: method in de behavior pool. [NIET GETEST]

```abap
METHOD startPromotion.
  MODIFY ENTITIES OF zsi00_r_promotion IN LOCAL MODE
    ENTITY Promotion
      UPDATE FIELDS ( Status )
      WITH VALUE #( FOR key IN keys ( %tky   = key-%tky
                                      Status = promo_status-active ) ).

  READ ENTITIES OF zsi00_r_promotion IN LOCAL MODE
    ENTITY Promotion
      ALL FIELDS WITH CORRESPONDING #( keys )
    RESULT DATA(promotions).

  result = VALUE #( FOR promotion IN promotions ( %tky   = promotion-%tky
                                                  %param = promotion ) ).
ENDMETHOD.
```

`endPromotion` is dezelfde method met `promo_status-ended`.

Wanneer is een action beschikbaar? Dat bepaal je in de method `get_instance_features`. RAP roept die op voor elke promotie die de app toont:

Fragment: method in de behavior pool. [NIET GETEST]

```abap
METHOD get_instance_features.
  READ ENTITIES OF zsi00_r_promotion IN LOCAL MODE
    ENTITY Promotion
      FIELDS ( Status ) WITH CORRESPONDING #( keys )
    RESULT DATA(promotions)
    FAILED failed.

  result = VALUE #(
    FOR promotion IN promotions
    LET is_planned = xsdbool( promotion-Status = promo_status-planned )
        is_active  = xsdbool( promotion-Status = promo_status-active )
        is_ended   = xsdbool( promotion-Status = promo_status-ended )
    IN ( %tky                   = promotion-%tky
         %action-startPromotion = COND #( WHEN is_planned = abap_true
                                          THEN if_abap_behv=>fc-o-enabled
                                          ELSE if_abap_behv=>fc-o-disabled )
         %action-endPromotion   = COND #( WHEN is_active = abap_true
                                          THEN if_abap_behv=>fc-o-enabled
                                          ELSE if_abap_behv=>fc-o-disabled )
         %update                = COND #( WHEN is_ended = abap_true
                                          THEN if_abap_behv=>fc-o-disabled
                                          ELSE if_abap_behv=>fc-o-enabled )
         %delete                = COND #( WHEN is_planned = abap_true
                                          THEN if_abap_behv=>fc-o-enabled
                                          ELSE if_abap_behv=>fc-o-disabled ) ) ).
ENDMETHOD.
```

Met `%update` en `%delete` leg je nog twee regels vast. Een beëindigde promotie wijzigt niemand nog. En een promotie die al liep, verwijder je niet: agentschappen hebben er reizen mee verkocht. Zet daarvoor in de behavior definition `update ( features : instance );` en `delete ( features : instance );`.

Waarom geen validation die controleert of de nieuwe status geldig is? Omdat een action de bedoeling van de gebruiker vastlegt. *Starten* is iets anders dan *het veld status wijzigen*, en alleen bij het eerste hoort de vraag of de gebruiker dat mag (4.11).

## 4.8 Draft: onafgewerkte data bewaren

Wim begint een promotie in te vullen, en dan gaat de telefoon. Een half uur later is de browser gesloten. Is het werk verloren?

Zonder draft wel: een promotie bestaat pas na een geslaagde bewaaractie, en die faalt zolang een validation faalt. Met **draft** (concept) bewaart RAP elke wijziging meteen in een aparte tabel, ook als de promotie nog niet geldig is. De gebruiker kan later verdergaan, en pas bij **Activate** (bewaren) lopen de validations en wordt het concept de echte promotie.

Draft zet je aan in de behavior definition. De volledige versie:

[NIET GETEST]

```abap
managed implementation in class zbp_si00_r_promotion unique;
strict ( 2 );
with draft;

define behavior for ZSI00_R_Promotion alias Promotion
persistent table zsi00_promo
draft table zsi00_promo_d
lock master total etag LastChangedAt
authorization master ( instance )
etag master LocalLastChangedAt
{
  create;
  update ( features : instance );
  delete ( features : instance );

  association _Airlines { create; with draft; }

  field ( numbering : managed, readonly ) PromoUuid;
  field ( readonly ) PromoId, Status, CreatedBy, CreatedAt,
                     LastChangedBy, LastChangedAt, LocalLastChangedAt;
  field ( mandatory ) AgencyId, ValidFrom, DiscountPct;

  action ( features : instance ) startPromotion result [1] $self;
  action ( features : instance ) endPromotion result [1] $self;

  determination setInitialValues on modify { create; }
  determination setDefaultValidTo on modify { create; field ValidFrom; }
  determination setPromoId on save { create; }

  validation validateAgency on save { create; field AgencyId; }
  validation validateDates on save { create; field ValidFrom, ValidTo; }
  validation validateDiscount on save { create; field DiscountPct; }

  draft action ( features : instance ) Edit;
  draft action Activate optimized;
  draft action Discard;
  draft action Resume;
  draft determine action Prepare
  {
    validation validateAgency;
    validation validateDates;
    validation validateDiscount;
  }

  mapping for zsi00_promo
  {
    PromoUuid          = promo_uuid;
    PromoId            = promo_id;
    AgencyId           = agency_id;
    Description        = description;
    ValidFrom          = valid_from;
    ValidTo            = valid_to;
    DiscountPct        = discount_pct;
    MaxDiscount        = max_discount;
    CurrencyCode       = currency_code;
    Status             = status;
    CreatedBy          = created_by;
    CreatedAt          = created_at;
    LastChangedBy      = last_changed_by;
    LastChangedAt      = last_changed_at;
    LocalLastChangedAt = local_last_changed_at;
  }
}

define behavior for ZSI00_R_PromoAirline alias PromoAirline
persistent table zsi00_promoair
draft table zsi00_promoair_d
lock dependent by _Promotion
authorization dependent by _Promotion
etag master LocalLastChangedAt
{
  update;
  delete;

  association _Promotion { with draft; }

  field ( numbering : managed, readonly ) AirlineUuid;
  field ( readonly ) ParentUuid, LocalLastChangedAt;
  field ( mandatory ) AirlineId;

  mapping for zsi00_promoair
  {
    AirlineUuid        = airline_uuid;
    ParentUuid         = parent_uuid;
    AirlineId          = airline_id;
    LocalLastChangedAt = local_last_changed_at;
  }
}
```

Wat er nieuw is:

- `with draft` en `draft table` zetten draft aan, voor de root en voor de child. Je maakt de drafttabellen met de quick fix op hun naam: `zsi00_promo_d` en `zsi00_promoair_d`.
- `with draft` bij de associations zegt dat een concept van een promotie ook concepten van airline-regels heeft.
- `total etag` is verplicht bij draft. Het detecteert of iemand de bewaarde versie wijzigde terwijl jij aan een concept werkte.
- De **draft actions**: `Edit` maakt een concept van een bewaarde promotie en `Activate` bewaart het concept. `Discard` gooit het weg, `Resume` gaat verder met een verlopen concept.
- `Prepare` voert de genoemde validations uit vlak voor `Activate`. Zo ziet de gebruiker alle fouten tegelijk.

`Edit` heeft `features : instance` gekregen. Voeg `%action-Edit` toe in `get_instance_features`, met dezelfde voorwaarde als `%update`: een beëindigde promotie gaat niet meer in bewerking.

Elke promotie bestaat nu in twee mogelijke versies: actief of concept. Daarom gebruiken we overal `%tky` en niet alleen de sleutel: `%tky` bevat ook `%is_draft`.

## 4.9 Projectie, service definition en service binding

Het business object is af, maar niemand kan het nog gebruiken behalve je eigen ABAP-code. Hoe maak je er een app van?

Eerst bepaal je wat één app te zien krijgt, met een **projection view** (projectie). Ze toont een selectie van het business object, met extra elements voor de app:

[NIET GETEST]

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Promotie (app)'
@Metadata.allowExtensions: true
@Search.searchable: true
define root view entity ZSI00_C_Promotion
  provider contract transactional_query
  as projection on ZSI00_R_Promotion
{
  key PromoUuid,
      @Search.defaultSearchElement: true
      PromoId,
      @EndUserText.label: 'Agentschap'
      @Consumption.valueHelpDefinition: [{ entity: { name: 'ZSI00_I_Agency', element: 'AgencyId' } }]
      @ObjectModel.text.element: ['AgencyName']
      AgencyId,
      _Agency.Name           as AgencyName,
      @EndUserText.label: 'Omschrijving'
      @Search.defaultSearchElement: true
      Description,
      @EndUserText.label: 'Geldig vanaf'
      ValidFrom,
      @EndUserText.label: 'Geldig tot'
      ValidTo,
      DiscountPct,
      @EndUserText.label: 'Maximale korting per reis'
      MaxDiscount,
      CurrencyCode,
      @ObjectModel.text.element: ['StatusText']
      Status,
      _StatusText.StatusText as StatusText,
      CreatedBy,
      LocalLastChangedAt,

      _Airlines : redirected to composition child ZSI00_C_PromoAirline,
      _Agency
}
```

[NIET GETEST]

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Airline van een promotie (app)'
@Metadata.allowExtensions: true
define view entity ZSI00_C_PromoAirline
  as projection on ZSI00_R_PromoAirline
{
  key AirlineUuid,
      ParentUuid,
      @EndUserText.label: 'Airline'
      @Consumption.valueHelpDefinition: [{ entity: { name: 'ZSI00_I_Airline', element: 'AirlineId' } }]
      @ObjectModel.text.element: ['AirlineName']
      AirlineId,
      _Airline.Name as AirlineName,
      LocalLastChangedAt,

      _Promotion : redirected to parent ZSI00_C_Promotion,
      _Airline
}
```

- `@Consumption.valueHelpDefinition` geeft een veld een zoekhulp: de gebruiker kiest een agentschap uit een lijst in plaats van het nummer te typen.
- `@ObjectModel.text.element` toont de naam naast het nummer: *070001 (Sunshine Travel)*.
- `@EndUserText.label` geeft een veld een Nederlands label, ook als het data element van `/DMO/` er geen heeft.
- `redirected to` zegt dat de associations in de projectie naar de andere projectie wijzen, niet naar de `R_`-views.

Daarnaast maak je een **behavior projection**: welke bewerkingen deze app mag gebruiken.

[NIET GETEST]

```abap
projection;
strict ( 2 );
use draft;

define behavior for ZSI00_C_Promotion alias Promotion
{
  use create;
  use update;
  use delete;

  use action startPromotion;
  use action endPromotion;

  use action Edit;
  use action Activate;
  use action Discard;
  use action Resume;
  use action Prepare;

  use association _Airlines { create; with draft; }
}

define behavior for ZSI00_C_PromoAirline alias PromoAirline
{
  use update;
  use delete;

  use association _Promotion { with draft; }
}
```

De **service definition** zegt welke entiteiten de service aanbiedt:

[NIET GETEST]

```abap
@EndUserText.label: 'Promoties'
define service ZSI00_UI_PROMOTION {
  expose ZSI00_C_Promotion    as Promotion;
  expose ZSI00_C_PromoAirline as PromoAirline;
  expose ZSI00_I_Agency       as Agency;
  expose ZSI00_I_Airline      as Airline;
}
```

De **service binding** zegt via welk protocol. Maak er een met een rechtermuisklik op de service definition: **New Service Binding**, naam `ZSI00_UI_PROMOTION_O4`, type **OData V4 - UI**. Activeer ze en klik op **Publish**. Selecteer daarna `Promotion` en klik op **Preview**: je browser opent de app.

## 4.10 Fiori elements: een app uit annotations

De preview toont een lege lijst met de melding dat er geen kolommen zijn. Je hebt nog geen HTML of JavaScript geschreven. Hoe wordt dit een bruikbare app?

**SAP Fiori elements** maakt apps op basis van annotations. Je beschrijft *wat* er op het scherm moet staan; Fiori elements bepaalt *hoe*, in de vaste stijl van SAP. Een **List Report** toont een lijst met filters, een **Object Page** de details van één promotie.

De UI-annotations zet je in een **metadata extension**, apart van de view. Maak er een met een rechtermuisklik op de projection view: **New Metadata Extension**.

[NIET GETEST]

```abap
@Metadata.layer: #CORE
@UI.headerInfo: { typeName:       'Promotie',
                  typeNamePlural: 'Promoties',
                  title:          { type: #STANDARD, value: 'Description' },
                  description:    { type: #STANDARD, value: 'AgencyName' } }
annotate entity ZSI00_C_Promotion with
{
  @UI.facet: [ { id:       'Promotion',
                 purpose:  #STANDARD,
                 type:     #IDENTIFICATION_REFERENCE,
                 label:    'Promotie',
                 position: 10 },
               { id:            'Airlines',
                 purpose:       #STANDARD,
                 type:          #LINEITEM_REFERENCE,
                 label:         'Airlines',
                 position:      20,
                 targetElement: '_Airlines' } ]

  @UI.hidden: true
  PromoUuid;

  @UI: { lineItem:       [ { position: 10 },
                           { type: #FOR_ACTION, dataAction: 'startPromotion', label: 'Starten' },
                           { type: #FOR_ACTION, dataAction: 'endPromotion',   label: 'Beëindigen' } ],
         identification: [ { position: 10 },
                           { type: #FOR_ACTION, dataAction: 'startPromotion', label: 'Starten' },
                           { type: #FOR_ACTION, dataAction: 'endPromotion',   label: 'Beëindigen' } ] }
  PromoId;

  @UI: { lineItem: [ { position: 20 } ], identification: [ { position: 20 } ],
         selectionField: [ { position: 10 } ] }
  AgencyId;

  @UI: { lineItem: [ { position: 30 } ], identification: [ { position: 30 } ] }
  Description;

  @UI: { lineItem: [ { position: 40 } ], identification: [ { position: 40 } ] }
  ValidFrom;

  @UI: { lineItem: [ { position: 50 } ], identification: [ { position: 50 } ] }
  ValidTo;

  @UI: { lineItem: [ { position: 60 } ], identification: [ { position: 60 } ] }
  DiscountPct;

  @UI.identification: [ { position: 70 } ]
  MaxDiscount;

  @UI: { lineItem: [ { position: 70 } ], identification: [ { position: 80 } ],
         selectionField: [ { position: 20 } ] }
  Status;
}
```

De airline-regels krijgen een eigen metadata extension. De facet `#LINEITEM_REFERENCE` hierboven toont ze als tabel op de detailpagina; hun `lineItem`-annotations bepalen de kolommen.

[NIET GETEST]

```abap
@Metadata.layer: #CORE
@UI.headerInfo: { typeName:       'Airline',
                  typeNamePlural: 'Airlines',
                  title:          { type: #STANDARD, value: 'AirlineName' } }
annotate entity ZSI00_C_PromoAirline with
{
  @UI.facet: [ { id:       'Airline',
                 purpose:  #STANDARD,
                 type:     #IDENTIFICATION_REFERENCE,
                 label:    'Airline',
                 position: 10 } ]

  @UI.hidden: true
  AirlineUuid;

  @UI.hidden: true
  ParentUuid;

  @UI: { lineItem: [ { position: 10 } ], identification: [ { position: 10 } ] }
  AirlineId;
}
```

| Annotation | Waar het verschijnt |
| --- | --- |
| `@UI.lineItem` | Kolom in de lijst |
| `@UI.selectionField` | Filter boven de lijst |
| `@UI.identification` | Veld op de detailpagina |
| `@UI.facet` | Een blok op de detailpagina: velden of een tabel met children |
| `@UI.headerInfo` | Titel van de detailpagina |
| `type: #FOR_ACTION` | Een knop voor een action |

Vernieuw de preview na het activeren. Je ziet nu een lijst met filters, en een detailpagina met de knoppen **Starten** en **Beëindigen** en een tabel met airlines. De status toont *Gepland* in plaats van `P`, dankzij de text association uit [3.7](./module-3.md#3_7).

## 4.11 Authorizations: wie mag wat

Iedereen kan nu elke promotie verwijderen, ook die van een collega. Het reisbureau wil dat alleen wie een promotie plande, ze ook kan verwijderen. Waar controleer je dat?

Niet in de app: een knop verbergen is geen beveiliging, want de OData-service blijft bereikbaar. De controle hoort in het business object, zodat ze voor elke app en elke API geldt.

In de behavior definition staat al `authorization master ( instance )`. RAP roept daardoor de method `get_instance_authorizations` op voor elke bewerking op een bestaande promotie. Per promotie geef je terug welke bewerkingen toegelaten zijn:

Fragment: method in de behavior pool. [NIET GETEST]

```abap
METHOD get_instance_authorizations.
  READ ENTITIES OF zsi00_r_promotion IN LOCAL MODE
    ENTITY Promotion
      FIELDS ( CreatedBy ) WITH CORRESPONDING #( keys )
    RESULT DATA(promotions)
    FAILED failed.

  DATA(current_user) = cl_abap_context_info=>get_user_technical_name( ).

  result = VALUE #(
    FOR promotion IN promotions
    ( %tky                   = promotion-%tky
      %update                = if_abap_behv=>auth-allowed
      %action-Edit           = if_abap_behv=>auth-allowed
      %action-startPromotion = if_abap_behv=>auth-allowed
      %action-endPromotion   = if_abap_behv=>auth-allowed
      %delete                = COND #( WHEN promotion-CreatedBy = current_user
                                       THEN if_abap_behv=>auth-allowed
                                       ELSE if_abap_behv=>auth-unauthorized ) ) ).
ENDMETHOD.
```

Probeer het uit met twee gebruikers. Een promotie van een collega kan je nog openen en wijzigen, maar verwijderen faalt met een melding.

Deze regel hangt af van de data: *wie maakte deze promotie?* Vaak hangt een recht af van de rol van de gebruiker: *is deze persoon teamverantwoordelijke?*

Dat regelt SAP met **authorization objects** (autorisatieobjecten). Een authorization object heeft velden, zoals `ACTVT` voor de activiteit. Een beheerder kent de rechten toe via rollen, en in je code vraag je ze op met `AUTHORITY-CHECK`.

Algemene vorm:

```abap
AUTHORITY-CHECK OBJECT '<authorization object>'
  ID '<field>' FIELD '<value>'.
IF sy-subrc <> 0.
  " ...
ENDIF.
```

`sy-subrc` is `0` als de gebruiker het recht heeft. Voor het veld `ACTVT` gebruikt SAP vaste codes:

| Code | Activiteit |
| --- | --- |
| `01` | Aanmaken |
| `02` | Wijzigen |
| `03` | Bekijken |
| `06` | Verwijderen |
| `43` | Vrijgeven |

In `get_instance_authorizations` zet je het resultaat van zo'n check om naar `if_abap_behv=>auth-allowed` of `if_abap_behv=>auth-unauthorized`, net als hierboven. Voor het project krijg je van je docent een authorization object en twee testgebruikers ([P5](./project.md#p5)).

Wat is het verschil tussen **instance features** (4.7) en **authorizations**? Features hangen af van de toestand van de data: *is de promotie gepland?* Authorizations hangen af van de gebruiker: *mag deze persoon dit?* Een action moet langs beide.

## Samenvatting

- **RAP** is het programmeermodel van SAP voor transactionele apps. Je bouwt het **business object** één keer, en per app een projectie, service en UI.
- Een **root view entity** is de top van het business object; **child entities** hangen eraan via een **composition** en verwijzen terug met `association to parent`.
- De **behavior definition** legt de **standard operations**, **field control**, validations, determinations en actions vast; je code staat in de **behavior pool**.
- Met **EML** (`READ ENTITIES`, `MODIFY ENTITIES`, `COMMIT ENTITIES`) spreek je een business object aan vanuit ABAP. Children maak je aan met `CREATE BY`.
- **Managed numbering** maakt UUID-sleutels; een leesbaar volgnummer ken je toe in een determination `on save`.
- Een **validation** controleert vlak voor het bewaren en vult `failed` en `reported`.
- Een **determination** berekent waarden, `on modify` of `on save`.
- Een **action** is een handeling met een naam; **instance features** bepalen wanneer ze beschikbaar is.
- **Draft** bewaart onafgewerkte data in een **draft table**; `%tky` bevat `%is_draft`.
- De **projection view**, **behavior projection**, **service definition** en **service binding** maken van het business object een OData-service.
- **Fiori elements** bouwt een List Report en Object Page op basis van UI-annotations in een **metadata extension**.
- **Authorizations** controleren of de gebruiker een bewerking mag, op basis van de data of met `AUTHORITY-CHECK` op een **authorization object**.

## Veelgemaakte fouten

- **Bedrijfslogica in de app of de client.** Een controle die alleen in de Fiori-app zit, geldt niet voor de API. Zet ze in het business object.
- **Een validation die een waarde wijzigt.** Een validation meldt alleen. Een waarde aanpassen is het werk van een determination.
- **Een determination die de keuze van de gebruiker overschrijft.** Vul alleen in wat leeg is, zoals `setDefaultValidTo` doet.
- **Een status rechtstreeks laten wijzigen.** Zet `Status` op `readonly` en wijzig de status alleen via actions.
- **`SELECT` op je eigen tabel in de behavior pool.** Je mist dan de wijzigingen in de transactional buffer. Gebruik `READ ENTITIES … IN LOCAL MODE`.
- **De sleutel gebruiken in plaats van `%tky`.** Met draft bestaat dezelfde sleutel twee keer: als concept en als actieve versie.
- **`FOR ALL ENTRIES` met een lege tabel.** Dan selecteer je alles. Controleer eerst of de tabel gevuld is.
- **Een knop verbergen als beveiliging.** Wie de service rechtstreeks aanspreekt, ziet geen knoppen. Gebruik authorizations.
- **`COND #` met tekst van verschillende lengte.** Het type komt van de eerste `THEN`, dus `'Niet bewaard'` wordt `'Niet be'`. Gebruik `COND string( )` en backticks.

::: oefening
[Projectstap P4 Business object en bedrijfslogica](./project.md#p4)
:::

::: oefening
[Projectstap P5 Fiori-app](./project.md#p5)
:::
