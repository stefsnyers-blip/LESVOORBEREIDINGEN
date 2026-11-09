---
module: 0
order: 1
moduleTitle: "0: Setup"
published: true
---

# Module 0: Setup

In deze module installeer je de ontwikkelomgeving en maak je verbinding met het labosysteem. Na afloop heb je een werkende ABAP-omgeving waarin je de rest van het vak programmeert. Deze module bevat geen leerinhoud die we later toetsen.

### Leerdoelen

Na deze module kan je:

- Eclipse met de ABAP Development Tools installeren;
- verbinding maken met het labosysteem;
- een ABAP class aanmaken en uitvoeren in de console.

De stappen gelden voor Windows 11 en macOS 15, met de nieuwste versies van Eclipse en ADT. [CONTROLEREN] Vul de exacte versienummers in na de test op het labosysteem.

## 0.1 Checklist: wat heb je nodig

Controleer of je alles hebt voor je begint. Ontbreekt er iets, vraag het dan aan je docent tijdens het eerste labo.

- Een laptop met minstens 8 GB RAM en 4 GB vrije schijfruimte.
- Een Java-runtime: die zit in het Eclipse-installatiepakket, je moet ze niet apart installeren.
- **Eclipse IDE for Java Developers**, de nieuwste versie.
- Toegang tot het netwerk van de hogeschool, op de campus of via VPN.
- Je gebruikersnaam en startwachtwoord voor het labosysteem. Je krijgt ze van je docent.
- Je teamnummer. Elk team krijgt een nummer van twee cijfers, bijvoorbeeld `07`.

Het labosysteem is een **ABAP Cloud Developer Trial** (gratis testversie van het ABAP-platform van SAP) die op een server van de hogeschool draait. Je installeert dus geen SAP-systeem op je eigen laptop: dat vraagt minstens 16 GB RAM, alleen voor SAP.

## 0.2 Eclipse en ADT installeren

ABAP-code schrijf je niet in een browser, maar in Eclipse met een plug-in van SAP: de **ABAP Development Tools** (ADT).

1. Download Eclipse IDE for Java Developers via [eclipse.org/downloads](https://www.eclipse.org/downloads/packages/).
2. Pak het archief uit of voer het installatieprogramma uit. Start Eclipse.
3. Kies een map voor je workspace, bijvoorbeeld `Documenten/eclipse-sap`. Je ziet nu het welkomstscherm.
4. Kies **Help > Install New Software…**.
5. Typ bij **Work with** dit adres en druk op Enter: `https://tools.hana.ondemand.com/latest`
6. Vink **ABAP Development Tools** aan en klik op **Next**.
7. Aanvaard de licentie en klik op **Finish**. Eclipse vraagt om opnieuw op te starten: doe dat.
8. Kies **Window > Perspective > Open Perspective > Other…** en dan **ABAP**. Je ziet nu links de **Project Explorer** en onderaan het tabblad **Problems**.

::: tip
Eclipse vraagt soms of je software met een onbekende handtekening vertrouwt. Voor de update site van SAP mag je dat bevestigen.
:::

## 0.3 Verbinden met het labosysteem

Een **ABAP project** in Eclipse is een verbinding met een SAP-systeem. Je code staat niet op je laptop, maar op de server. Eclipse is een venster op dat systeem.

1. Kies **File > New > Project…** en dan **ABAP > ABAP Project**.
2. Vul de verbindingsgegevens in die je van je docent kreeg: systeem-ID `A4H`, de servernaam en instance number `00`. [CONTROLEREN] Welke verbindingsmethode werkt zonder SAP GUI op het labosysteem.
3. Klik op **Next**. Vul bij **Client** `001` in, en daarna je gebruikersnaam en wachtwoord. Laat de taal op `EN` staan. [CONTROLEREN] Of Nederlands als aanmeldtaal geïnstalleerd is op het labosysteem.
4. Klik op **Finish**. Je ziet nu in de Project Explorer een project met de naam `A4H_001_<gebruikersnaam>_en`.
5. Het systeem vraagt bij de eerste aanmelding om je wachtwoord te wijzigen. Kies een sterk wachtwoord en bewaar het in je wachtwoordbeheerder.

Open het project. Onder **Favorite Packages** staat `$TMP`: dat is je persoonlijke kladmap voor objecten die je niet wil bewaren.

## 0.4 Hello world: je eerste class

We controleren of alles werkt met een kleine class die tekst toont in de console van Eclipse.

1. Klik met de rechtermuisknop op je project en kies **New > ABAP Class**.
2. Vul in: package `$TMP`, naam `ZHELLO_<gebruikersnaam>`, beschrijving `Hello world`. Klik op **Finish**.
3. Vervang de code door het voorbeeld hieronder. Vervang `zhello_kevin` door de naam van jouw class.
4. Druk op **Ctrl+F3** om de class te activeren.
5. Druk op **F9** om de class uit te voeren.

[NIET GETEST]

```abap
CLASS zhello_kevin DEFINITION
  PUBLIC FINAL CREATE PUBLIC.

  PUBLIC SECTION.
    INTERFACES if_oo_adt_classrun.
ENDCLASS.

CLASS zhello_kevin IMPLEMENTATION.
  METHOD if_oo_adt_classrun~main.
    out->write( 'Hallo, SAP! Je labosysteem werkt.' ).
  ENDMETHOD.
ENDCLASS.
```

Je ziet nu in het tabblad **Console**:

```output
Hallo, SAP! Je labosysteem werkt.
```

Zie je die regel? Dan is de setup klaar. De interface `if_oo_adt_classrun` gebruiken we de hele cursus om snel iets uit te proberen. Wat ze precies doet, zie je in [3.3](./module-3.md#3_3).

## 0.5 SAP Learning Hub en de BTP-trial

Naast het labosysteem gebruik je twee gratis diensten van SAP.

**SAP Learning Hub, student edition** geeft je een jaar gratis toegang tot de cursussen van SAP, oefensystemen en twee pogingen voor een SAP-certificaat. In module 1 gebruiken we de oefensystemen om processen in SAP S/4HANA te bekijken. Registreer je via [learning.sap.com/free-student-edition](https://learning.sap.com/free-student-edition) met je e-mailadres van de hogeschool. De activatie duurt een tot twee werkdagen, dus doe dit vandaag.

De **SAP BTP ABAP Environment trial** is een reserveomgeving in de cloud. Je gebruikt ze alleen als het labosysteem niet bereikbaar is. Een trial-account blijft 90 dagen geldig, dus maak het pas aan als je het nodig hebt. Volg dan de tutorial [Create an SAP BTP ABAP Environment Trial User](https://developers.sap.com/tutorials/abap-environment-trial-onboarding.html).

::: warning
**Let op:** code die je in de BTP-trial schrijft, staat niet op het labosysteem. Wil je ze later in het project gebruiken, dan zet je ze over met abapGit ([6.3](./module-6.md#6_3)).
:::

## 0.6 Werkt het niet?

| Probleem | Oplossing |
| --- | --- |
| **Install New Software** vindt geen ADT | Controleer het adres in stap 5 van 0.2. Een proxy op je thuisnetwerk kan de download blokkeren: probeer op de campus. |
| De verbinding met het labosysteem time-out | Ben je buiten de campus? Start de VPN van de hogeschool. |
| `Logon failed` of `Password logon no longer possible` | Je wachtwoord is fout of je account is geblokkeerd na te veel pogingen. Vraag je docent om het te deblokkeren. |
| De class activeert niet | Kijk in het tabblad **Problems**. Meestal is de naam in de code anders dan de naam van de class. |
| F9 doet niets | Klik eerst in de editor van de class, zodat Eclipse weet welke class je wil uitvoeren. |

## Samenvatting

- Je schrijft ABAP in Eclipse met de **ABAP Development Tools** (ADT).
- Een **ABAP project** is een verbinding met een SAP-systeem; de code staat op de server.
- `$TMP` is je persoonlijke package voor objecten die je niet bewaart.
- Een class met `if_oo_adt_classrun` voer je uit met F9 en toont output in de console.
