# ADV B2C dokumentace LK

# Souhrn fungovani zaslani B2C advice note

Tento dokument popisuje tok B2C advice note v repozitari `java_advicenote-b2c-master` tak, aby byl citelny i pro nevyvojare, ale zaroven obsahoval technicke detaily potrebne pro hledani problemu s dvojim vyfakturovanim.

## Slovnicek

- **Advice note (AN)**: seznam polozek z kosiku, ktery se posila do MPOS/MARS, aby z nej v prodejnim/fakturacnim toku mohl vzniknout dodaci list/faktura.
- **Kosik / B2C reference**: v kodu `customerOrderRef`, ulozena v DB sloupci `CUST_ORDER_REF`. Je to business reference objednavky/kosiku. V UI se zobrazuje jako "Cislo objednavky zak.".
- **`header.txNumber`**: technicka transakcni reference v tele advice note. Podle ni se pozdeji dohledava faktura v Erice jako `szExternalTxNumber`.
- **MPOS/MARS**: cilovy system, kam tato aplikace posila advice note.
- **Erika**: system, ze ktereho se pozdeji cte uz vystaveny packing list/faktura.
- **MPOS AdviceNoteId**: ID, ktere vrati MPOS/MARS po prijeti advice note. V logu je ulozene jako `MPOS_ADVICE_NOTE_ID`.
- **Packing list / dodaci list**: v kodu `packingListNo`, v XML z Eriky `szInvoiceID`.

## Hlavni tok: prvni odeslani advice note

1. Klient zavola REST endpoint teto aplikace:
   `POST /api/{mposServerUrl}/AdviceNotes`.

2. Spring prijme JSON do tridy `AdviceNoteSaveRequest`. Ta rozsiruje MPOS model `AdviceNoteSaveRequestViewModel` o povinne pole `customerOrderRef`.

3. `AdviceNoteService.postAdviceNote(...)` nejdrive ocisti textova mnozstvi a vahy, aby kolem nich nebyly mezery. Potom validuje, ze `quantity` a `weight` jdou prevest na `BigDecimal`.

4. `AdviceNoteB2CFactory.createFromRequest(...)` vytvori DB entitu `AdviceNoteB2C`.
   Ulozi hlavicku, zakaznika, polozky a business referenci `customerOrderRef`.

5. Entita se ulozi do tabulky `KOSIK_B2C_ADVICE_NOTE` se stavem `NEW`.

6. Aplikace vytvori zaznam v `KOSIK_B2C_ADVICE_NOTE_LOG`, kam ulozi JSON requestu, ktery se posila do MPOS/MARS.

7. Aplikace vola vygenerovaneho MPOS klienta:
   `adviceNotesApi.postWithHttpInfo(adviceNote.getMposServerUrl(), model)`.

8. Pokud MPOS/MARS odpovi bez HTTP vyjimky, aplikace nastavi stav advice note na `MARS_ACCEPTED`.
   Pokud prijde `HttpClientErrorException` nebo `HttpServerErrorException`, nastavi stav `MARS_REJECTED`.

9. Do hlavni advice note i do logu se ulozi odpoved z MPOS/MARS. Z odpovedi se ulozi zejmena:
   `header.adviceNoteId` jako `mposAdviceNoteId`, HTTP status, cas odeslani a chyby.

10. Aplikace vrati odpoved puvodnimu klientovi.

Relevantni kod:

- `AdviceNoteController.postRequestViewModel(...)`
- `AdviceNoteService.postAdviceNote(...)`
- `AdviceNoteB2CFactory.createFromRequest(...)`
- `AdviceNoteMapper.mapRequestToAdviceNote(...)`
- `AdviceNoteService.tryPost(...)`
- `AdviceNoteLogService.mapResponseAndSave(...)`

## Stavovy model

Pouzivane stavy jsou definovane v `AdviceNoteStatus`:

- `NEW`: advice note byla prijata a ulozena.
- `VALID` / `INVALID`: pripraveno v modelu, v B2C toku se prakticky nepouziva.
- `MARS_ACCEPTED`: MPOS/MARS advice note prijal.
- `MARS_REJECTED`: MPOS/MARS ji odmitl nebo vratil chybu.
- `INVOICE_FOUND`: aplikace pozdeji nasla fakturu/packing list v Erice.
- `CANCELED`: obecny stav pro zrusene advice note; v B2C implementaci ale ruseni predchozich advice note neni provedene.

Dulezite: v B2C variante je `cancelPreviousAdviceNotes(...)` prazdne. To znamena, ze kdyz prijde stejna business reference nebo stejny obsah znovu, aplikace sama nezrusi predchozi advice note.

## Resend: znovuodeslani advice note

Resend endpoint je:

`POST /api/{mposServerUrl}/AdviceNotes/{adviceNoteId}/resend`

Tok:

1. Aplikace najde existujici `AdviceNoteB2C` podle interniho DB ID.

2. Vezme ulozeny puvodni JSON request z `ADVICE_NOTE.REQ`.

3. Prevede ho zpet na `AdviceNoteSaveRequest`.

4. Prepisuje `model.header.txNumber` na novou nahodnou hodnotu dlouhou 32 hex znaku.

5. Znovu vola MPOS/MARS pres `tryPost(...)`.

6. Vytvori se dalsi log pro stejne `B2C_ADVICE_NOTE_ID`, ale s novym `txNumber`.

7. Pokud MPOS/MARS odpovi uspesne, stav se znovu nastavi na `MARS_ACCEPTED`.

Nejdulezitejsi technicky fakt: resend neposila "stejnou transakci znovu". Posila stejne polozky pod novym `txNumber`. Pro downstream system to muze vypadat jako nova advice note.

## Dohledani faktury v Erice

Faktura se nevystavuje primo v teto aplikaci. Aplikace pouze pravidelne hleda, jestli uz v Erice vznikl packing list/faktura.

Tento proces bezi:

- po startu aplikace,
- potom kazdych 10 minut.

Tok:

1. `InvoiceSearchingExpert.getInvoiceNoB2C()` nacte vsechny advice note se stavem `MARS_ACCEPTED`, ktere byly prijate za poslednich 7 dni.

2. Pro kazdou advice note vezme **posledni log** podle `sentTime`.

3. Z posledniho logu se snazi ziskat `txNumber`:
   nejdriv z response, potom z requestu.

4. Z advice note a logu sestavi dotaz do Eriky:
   z `mposServerUrl` odvozuje prodejnu,
   z `customerBarcode` odvozuje zakaznika a home store,
   z `sentTime` bere datum.

5. Erika se dotaze na packing listy pro danou prodejnu, zakaznika, home store a den.

6. Z kazdeho XML packing listu se parsuje:
   `szExternalTxNumber`, `lReceiptNmbr`, `szInvoiceID`.

7. Aplikace spoleha na to, ze `szExternalTxNumber == header.txNumber` z posledniho odeslani.

8. Kdyz najde shodu, ulozi zaznam do `KOSIK_B2C_INVOICE` a stav advice note nastavi na `INVOICE_FOUND`.

Relevantni kod:

- `InvoiceSearchingExpert.getInvoiceNoB2C()`
- `InvoiceSearchingExpert.parseHeaderTxNumber(...)`
- `ErikaInvoiceDataRepository.prepareData(...)`
- `ErikaInvoiceDataRepository.getInvoiceB2C(...)`
- `custom-invoice-struct.json`, objekt `ADVICENOTE_HEADER`

## Proc muze vzniknout dvojita fakturace

Nejpravdepodobnejsi problem neni v samotnem ulozeni faktury, ale v kombinaci techto vlastnosti:

1. MPOS/MARS prijme puvodni advice note a downstream z ni muze vytvorit fakturu.

2. Nase aplikace ale fakturu jeste nenajde. To muze byt zpozdenim Eriky, chybou parovani, chybnym/nezachycenym `txNumber`, jinym datem, chybejicim logem, chybejicim `mposAdviceNoteId`, nebo tim, ze se hleda jen podle posledniho logu.

3. Advice note zustane ve stavu `MARS_ACCEPTED`, tedy vypada jako "prijato v MPOS, ale bez faktury".

4. Operator, UI nebo skript ji posle pres resend.

5. Resend zmeni `header.txNumber` na novou hodnotu.

6. MPOS/MARS muze novy `txNumber` vyhodnotit jako novou transakci a vytvorit dalsi advice note / dalsi packing list / dalsi fakturu.

7. Pri dalsim dohledani faktury se pouzije jen posledni log. Aplikace tedy muze sparovat az druhou fakturu a prvni fakturu s puvodnim `txNumber` uz nevidi jako stav teto advice note.

V jednoduche reci: aplikace si mysli "posilam znovu stejnou vec", ale MPOS/MARS muze slyset "posilam novou vec".

## Proc jedna faktura muze byt s referenci a druha bez

Je potreba oddelit dve reference:

1. `customerOrderRef`
   Business reference objednavky/kosiku. Aplikace ji vyzaduje pri normalnim prijmu, uklada ji do `CUST_ORDER_REF` a zobrazuje v UI.

2. `header.txNumber`
   Technicka transakcni reference pro MPOS/MARS a pozdejsi parovani s Erikou.

MPOS swagger model neobsahuje `customerOrderRef`; ten existuje az v nasi subclass `AdviceNoteSaveRequest`. Proto:

- aplikace `customerOrderRef` pouziva pro vlastni evidenci,
- fakturu v Erice nepari podle `customerOrderRef`,
- resend neni chraneny business referenci,
- downstream system nemusi `customerOrderRef` vubec prenaset do faktury,
- log requestu v `AdviceNoteLogService.createAdviceNoteLog(...)` se serializuje jako `AdviceNoteSaveRequestViewModel`, takze business reference v logovanem requestu nemusi byt videt.

Pokud se v realu objevi jedna faktura "s referenci" a druha "bez reference", tento kod k tomu dava logicke vysvetleni: prvni a druhe odeslani nejsou pro MPOS stejna idempotentni operace a business reference neni hlavni parovaci ani ochranny klic.

## Rizikova mista v kodu

### 1. Resend meni `txNumber`

`AdviceNoteService.resendAdviceNote(...)` vola `model.getHeader().setTxNumber(getRandomTxNumber())`.

To je nejvetsi kandidat na dvojitou fakturaci. Novy `txNumber` vytvori novou technickou identitu transakce.

### 2. Backend nema statusovou ochranu proti resend

UI tlacitko vypina resend, pokud je stav `INVOICE_FOUND` nebo pokud uz se dnes odeslalo. Backend endpoint ale takovou kontrolu nema.

Prime volani API nebo skript tedy muze resendnout i zaznam, ktery by UI nepustilo.

### 3. B2C nema implementovane ruseni predchozich AN

Metoda `cancelPreviousAdviceNotes(...)` je v `AdviceNoteB2CService` prazdna. Pokud prijde stejna objednavka znovu, aplikace neudela automaticke zruseni predchozi verze.

### 4. DB nema zjevnou idempotencni pojistku

Entity a repository neukazuji unikatni omezeni na:

- `CUST_ORDER_REF`,
- `HEADER_TX_NUMBER`,
- `MPOS_ADVICE_NOTE_ID`,
- kombinaci zakaznik + reference + polozky,
- fakturu / packing list v `KOSIK_B2C_INVOICE`.

Bez DB constraintu muze vzniknout vice radku pro stejnou obchodni udalost.

### 5. Faktura se hleda jen podle posledniho logu

`getNewestLogBy(...)` bere posledni log podle `sentTime`. Po resendu je posledni log resend log.

Kdyz existovala faktura k puvodnimu `txNumber`, ale po resendu uz se hleda jen novy `txNumber`, puvodni faktura se muze ztratit z pohledu aplikace.

### 6. Manualni generovani ID logu pres `MAX(id)+1`

`AdviceNoteB2CLogService` generuje ID logu dotazem:

`SELECT NVL(MAX(B2C_ADVICE_NOTE_LOG_ID), 0) + 1 FROM KOSIK_B2C_ADVICE_NOTE_LOG`

Pri paralelnich odeslanich to neni bezpecne. Dva requesty mohou dostat stejne dalsi ID. To muze zpusobit chybu ulozeni logu, a tim rozbit pozdejsi parovani.

### 7. Scheduler neni cluster-lockovany

`AtomicBoolean` chrani jen jednu bezici JVM instanci. Pokud bezi vice instanci aplikace, kazda muze spoustet dohledavani faktur.

To neni primarni duvod dvojite fakturace v MPOS, ale muze to zpusobit duplicitni lokalni zaznamy faktur nebo zavodni chovani.

## Co ukazuji lokalni reporty v repozitari

V `report_data` je vysledek analyzy/resendu:

- zdroj ma 730 radku,
- resend vybral 643 radku,
- 479 resend pokusu skoncilo HTTP 200,
- 164 resend pokusu skoncilo HTTP 400,
- ve zdroji je jedna duplicitni business reference `4903868794`,
- tato reference existuje pro `B2CAdviceNoteId` `390689` ve stavu `NEW` a `409074` ve stavu `MARS_ACCEPTED`.

To je prakticky priklad, ze business reference neni v datech striktne unikatni.

## Jak overit konkretni dvojite faktury v DB

### Najit vice advice note pro stejnou business referenci

```sql
SELECT CUST_ORDER_REF,
       COUNT(*) AS CNT,
       LISTAGG(B2C_ADVICE_NOTE_ID, ', ') WITHIN GROUP (ORDER BY B2C_ADVICE_NOTE_ID) AS IDS
FROM KOSIK_B2C_ADVICE_NOTE
WHERE CUST_ORDER_REF IS NOT NULL
GROUP BY CUST_ORDER_REF
HAVING COUNT(*) > 1;
```

### Najit advice note s vice uspesnymi logy

```sql
SELECT B2C_ADVICE_NOTE_ID,
       COUNT(*) AS SUCCESS_LOGS,
       LISTAGG(MPOS_ADVICE_NOTE_ID, ', ') WITHIN GROUP (ORDER BY SENT_TIME) AS MPOS_IDS
FROM KOSIK_B2C_ADVICE_NOTE_LOG
WHERE STATUS BETWEEN 200 AND 299
GROUP BY B2C_ADVICE_NOTE_ID
HAVING COUNT(*) > 1;
```

### Porovnat `txNumber` v jednotlivych logach

Toto je potreba delat parsovanim JSONu v `REQ`/`RESP`. Cilem je najit stejny `B2C_ADVICE_NOTE_ID`, ktery ma vice ruznych `header.txNumber`.

Pokud ma jedna interni advice note vice ruznych `txNumber` a pro oba existuje packing list v Erice, je to silny dukaz, ze dvojita fakturace vznikla resend cestou.

### Najit vice ulozenych faktur pro stejnou advice note

```sql
SELECT B2C_ADVICE_NOTE_ID,
       COUNT(*) AS INVOICE_ROWS,
       LISTAGG(PACKING_LIST, ', ') WITHIN GROUP (ORDER BY INVOICE_DATE, INVOICE_TIME) AS PACKING_LISTS
FROM KOSIK_B2C_INVOICE
GROUP BY B2C_ADVICE_NOTE_ID
HAVING COUNT(*) > 1;
```

### Najit faktury ke stejne business referenci

```sql
SELECT an.CUST_ORDER_REF,
       an.B2C_ADVICE_NOTE_ID,
       an.ADVICE_NOTE_STATUS,
       inv.PACKING_LIST,
       inv.TILL_NO,
       inv.INVOICE_NO,
       inv.INVOICE_DATE,
       inv.INVOICE_TIME
FROM KOSIK_B2C_ADVICE_NOTE an
LEFT JOIN KOSIK_B2C_INVOICE inv
       ON inv.B2C_ADVICE_NOTE_ID = an.B2C_ADVICE_NOTE_ID
WHERE an.CUST_ORDER_REF = :reference
ORDER BY an.B2C_ADVICE_NOTE_ID, inv.INVOICE_DATE, inv.INVOICE_TIME;
```

## Doporučene opravy / ochrany

### Nejdulezitejsi okamzita ochrana

Na backendu zakazat resend pro:

- `INVOICE_FOUND`,
- zaznamy, kde existuje faktura v `KOSIK_B2C_INVOICE`,
- zaznamy, ktere maji uspesny MPOS log z dneska,
- pripadne vsechny `MARS_ACCEPTED`, dokud se pred resend nepodiva Erika podle vsech drivejsich `txNumber`.

UI ochrana nestaci, protoze skript a prime API volani ji obejdou.

### Pred resend kontrolovat Eriku podle vsech historickych logu

Pred zmenou `txNumber` a odeslanim je potreba:

1. nacist vsechny uspesne logy dane advice note,
2. z kazdeho vytahnout `txNumber`,
3. zeptat se Eriky, jestli uz k nekteremu `txNumber` existuje packing list,
4. pokud existuje, ulozit fakturu a resend neprovadet.

### Neprenastavovat `txNumber` bez jasneho duvodu

Pokud MPOS podporuje idempotentni opakovani stejneho `txNumber`, resend by mel pouzit puvodni `txNumber`.

Pokud MPOS vyzaduje novy `txNumber`, musi se to brat jako nova transakce a musi existovat explicitni ochrana proti dvojimu vyfakturovani.

### Pouzit duplicate/cancel mechanismus MPOS, pokud existuje

Swagger obsahuje pole `duplicatedAdviceNoteId` a `isDuplicateAdviceNote`. Aktualni resend je nepouziva. Pokud MPOS ma oficialni zpusob, jak oznacit duplikat nebo nahradu, resend by mel posilat prave tato pole a/nebo predchozi advice note zrusit.

### Pridat DB unikaty nebo idempotencni tabulku

Doporucene minimalni pojistky:

- unikatni index na `HEADER_TX_NUMBER`, pokud ma byt unikatni v ramci systemu,
- kontrola duplicity `CUST_ORDER_REF` podle business pravidel,
- unikatni index na `KOSIK_B2C_INVOICE.PACKING_LIST`,
- unikatni index na kombinaci faktury, napr. `TILL_NO + INVOICE_NO + INVOICE_DATE`,
- sekvence pro `KOSIK_B2C_ADVICE_NOTE_LOG` misto `MAX(id)+1`.

### Logovat business referenci i v resend logu

Log requestu se dnes serializuje jako zakladni MPOS model. Pro vysetrovani by bylo lepsi explicitne logovat:

- `customerOrderRef`,
- puvodni `txNumber`,
- novy `txNumber`,
- duvod resendu,
- kdo resend spustil,
- zda pred resend byla provedena kontrola v Erice.

## Kratky zaver

Nejpravdepodobnejsi pricina dvojite fakturace je resend advice note, ktera uz byla v MPOS prijata, ale nase aplikace ji jeste neoznacila jako `INVOICE_FOUND`. Resend vygeneruje novy `header.txNumber`, takze downstream system muze vytvorit druhou fakturu. Business reference `customerOrderRef` neni pouzita jako idempotencni klic ani jako parovaci klic do Eriky, a proto sama o sobe dvojimu odeslani nezabrani.


