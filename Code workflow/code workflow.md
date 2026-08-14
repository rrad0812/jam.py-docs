
# `Jam.py` projekat iznutra

## Neki od tokova u Jam.py

### Aplikacioni start

Framework otvara `admin.sqlite` i podiže `admin` metapodatke

- App start ide kroz wsgi.py:87 i wsgi.py:145.
- U konstruktoru App se poziva `create_admin`: wsgi.py:164, admin.py:57.
- AdminTask dobija putanju do admin baze: admin.py:61.
- Zatim ide `init_admin`: admin.py:43.

### Izgradnja projektnog task stabla

Iz `admin` metapodataka pravi se runtime Task sa objektima

- `create_task` čita `sys_items` iz admin taska i traži root task zapis: task.py:9.
- `load_task` gradi celo stablo objekata: task.py:331.
- `fill_rec_dicts` čita `sys_items`, `sys_fields`, `sys_filters`, `sys_report_params`: task.py:307.
- `create_groups/create_items` mapiraju redove na Group/Item instance i pune
  `server_code` iz `f_server_module`: task.py:109, task.py:138, task.py:158.
- Posle toga `bind_items` i `compile_all` povezuju i kompajliraju server kod: task.
  py:339.
- Kompajliranje izvršava kod iz `server_code` i kači funkcije na `item/task`: items.
 py:665.

### Izbor glavne projektne baze

- Admin builder čita `sys_tasks` i postavlja `task_db_info/task_db_type`: builder.py:96.
- `load_task` preuzima to u runtime task i kreira `DB adapter/pool`: task.py:342.

### Korisnik bira objekat iz task stabla projekta

- HTTP ulaz ide kroz `App.call` i rutira se na `index/api`: wsgi.py:206, wsgi.
  py:241, wsgi.py:300.
- Frontend inicijalno šalje `load` preko `/api`: task.js:330, task.js:110.
- Backend `on_api` parsira JSON paket i zove `get_response`: wsgi.py:667, wsgi.
  py:726.
- Za `load` vraća `task.get_info` + `templates/settings/locale/language`: wsgi.
  py:520.

### Kada korisnik otvori konkretan objekat

- Frontend šalje `open`: item.js:1778.
- Backend `get_response` za `open` zove `select_records`: wsgi.py:727, items.py:232.
- `select_records` radi hookove i SQL read kroz `execute_open`: items.py:240, items.py:178.

### Kada frontend zove task.server("ime_funkcije")

- JS metoda `server` je ovde: abstr_item.js:373.
- Ona šalje `server` request: abstr_item.js:389.
- Backend to rutira na `server_func/getattr` i izvrši zahtevanu funkciju po imenu:
  wsgi.py:731,  wsgi.py:743.

**Suština**:

`admin.sqlite` je metadata baza podataka iz koje framework sastavlja Task/Item objekte i njihov server kod; posle toga klijent radi preko `/api`, a za konkretan objekat ide - `open` ili `server` poziv kroz `item_id` i `ime funkcije`.

## /api endpoint

Kroz wsgi.py izložena je praktično jedna API ulazna tačka: putanja `/api`.

Endpoint prima:

- POST JSON paket u formatu [method, task_id, item_id, params, modification]
- Parsiranje je u wsgi.py

Kao method izloženi su:

- `load`
- `open`
- `apply`
- `server`
- `print`

To grananje je u wsgi.py.

Kako se mapira:

1. `task_id` bira kontekst taska:
   - `0` znači `admin` task
   - `\>` 0 znači `projektni` task

   Logika je u wsgi.py.

2. `item_id` bira objekat:
   - `task.item_by_ID(item_id)`

   Sve to  u wsgi.py

3. method bira operaciju:
   - `open` -> `item.select_records(...)`
   - `apply` -> `item.apply_changes(...)`
   - `server` -> `server_func(item, func_name, params)`
   - `print` -> `item.print_report(...)`
   - `load` -> `init_client(...)`

   Sve to  u wsgi.py

4. Server poziv po imenu: `server_func` radi `getattr(obj, func_name)` pa poziva
   funkciju.

   Sve to  u wsgi.py

5. Frontend strana koja gađa `/api`:

   - JSON POST ide iz `process_request` u task.js
   - URL je eksplicitno api u task.js
   - `task.server(...)` pakuje server method poziv u abstr_item.js

6. Van `/api` su odvojeni endpointi:

   - `upload` na `/upload`: wsgi.py
   - `ext` na `/ext`: wsgi.py
   - `index/login/logout` routing u wsgi.py

### Primer 1: `load` (inicijalizacija task-a na klijentu)

Request body ka /api:

```json
["load", 1, 1, null, 0]
```

- `method = load`
- `task_id = 1` (projektni task)
- `item_id = 1` (task objekat)
- `params = null`
- `modification = 0`

Backend obrada:

- Parsiranje: wsgi.py
- Grananje `load`: wsgi.py
- Povrat `task_info`: wsgi.py

Tipičan response shape:

```json
{
  "result": {
    "status": 9,
    "data": [
      {
        "task": { "id": 1, "name": "test", "items": [] },
        "templates": "<html...>",
        "settings": { "SAFE_MODE": true },
        "locale": {},
        "language": {},
        "user_info": {},
        "privileges": null
      },
      ""
    ],
    "modification": 123
  },
  "error": null
}
```

### Primer 2: `open` (učitavanje reda iz item-a)

Frontend sklapa params sa ključevima:

- `__expanded`
- `__fields`
- `__open_empty`
- `__order`
- `__filters`
- `__limit`
- `__offset`

To se vidi u: item.js

Primer request:

```json
[
  "open",
  1,
  57,
  {
    "__expanded": true,
    "__fields": ["id", "customer", "total"],
    "__open_empty": false,
    "__order": [["id", false]],
    "__filters": [["id", 1, 10]],
    "__limit": 50,
    "__offset": 0
  },
  123
]
```

Backend obrada:

- `open` => `select_records`: wsgi.py
- `select_records pipeline`: items.py

Tipičan response:

```json
{
  "result": {
    "status": 9,
    "data": [
      [
        [10, 3, 1250.5],
        [11, 5, 990.0]
      ],
      ""
    ],
    "modification": 123
  },
  "error": null
}
```

### Primer 3: `server` (poziv server funkcije po imenu)

Frontend poziv:

- `task.server("prepare_users")` u JS
- Metoda `server`: abstr_item.js
- Ona šalje: `send_request("server", [func_name, params])`: abstr_item.js

Request body:

```json
["server", 1, 1, ["prepare_users", []], 123]
```

Backend obrada:

- `server` => `server_func`: wsgi.py
- `getattr` i poziv funkcije: wsgi.py

Tipičan response:

```json
{
  "result": {
    "status": 9,
    "data": [null, ""],
    "modification": 123
  },
  "error": null
}
```

Suština:

- `/api` prima jedan uniforman JSON paket.
- `method` određuje šta se radi (`load/open/apply/server/print`).
- Za `server` pozive, ime funkcije je string u params[0], a argumenti su params[1].

## Tok čuvanja zapisa (edit forma -> baza)

Korisnik klikne `OK` u formi

### `apply_record`

Poziva se `apply_record` u item.js:3327

- `apply_record` prvo radi `post`: item.js:3339.
  - `post` radi:
    - validaciju zapisa (`check_record_valid`),
    - `on_before_post`,
    - post za detalje koji su u edit/insert,
    - upis promene u `change_log`,
    - prelaz stanja u browse,
    - `on_after_post`.

    Sve to je u item.js:2325.  

    **Važna stvar**: `post` ne ide na server. `post` je lokalni commit u `client dataset/change_log`. Nema `/api` poziva u toj funkciji.  

- Nakon `post` ide `apply`, `apply_record` poziva `apply` sa callback-om: item.js:3353.
  
  - `apply` proverava:
    - ako je detail sa `master_applies`, propagira ka master-u,
    - ako je item još u edit/insert, radi `post`,
    - uzima `delta` izmene iz `change_log`.

    Glavna logika je u item.js:2402.  
    Pre slanja `apply` pakuju se hook parametri.

    - `on_before_apply` se skuplja kroz lanac caller -> master: item.js:2378.
    - Slanje ka serveru
    - Ako ima promena, šalje se request `apply` sa payload-om [changes, params_dict]: item.js:2447.
    - Taj request ide na `/api` kroz `process_request`: task.js:86, task.js:110.

- Backend dispatch
  - `/api` parsira `method/task_id/item_id/params/modification`: wsgi.py:667.
  - Za method `apply` zove `item.apply_changes(...)`: wsgi.py:734.
  - Server-side `apply_changes`
    - Rekonstruiše `delta` iz client promena.
    - Pokreće `on_apply` hook-ove (task pa item).
    - Ako hook ne vrati rezultat, ide default `apply_delta`.
    - Na kraju commit konekcije.
    Sve je u items.py:289, items.py:301, items.py:306, items.py:308.

    - `apply_delta` (default put)
      - Radi validaciju delta zapisa.
      - Poziva `db.process_changes` za SQL operacije.
      - Vraća update paket za client `change_log`.

      U items.py:272.

- Povratak na frontend
  - `_process_apply` prima response,
    - proverava error,
    - update-uje `change_log`,
    - okida `on_after_apply`
    - i osvežava kontrole.

    Sve to u item.js:2462.

**Ukratko**:

- `post` = lokalno zatvaranje edit-a i priprema delta.
- `apply` = slanje delta na server + DB commit + povrat update-a.

## COPY item

COPY ne pravi novu tabelu nego označava item kao kopiju (`f_copy_of`) i tada backend namerno preskače DDL.

Šta COPY zapravo radi u v7:

1. COPY u UI pravi novi zapis u `sys_items` kao kopiju postojećeg item-a

   - klik na copy dugme: admin.js  
   - postavlja `f_copy_of` na originalni item: admin.js  
   - odmah pokreće biranje/punjenje polja iz izvora: admin.js

2. Novi copy item preuzima tehničke atribute izvora, uključujući tabelu  

   - pri copy kliku kopiraju se gotovo sva polja item-a (osim par sistemskih referenci): admin.js  
   - to uključuje i `f_table_name/f_gen_name`, pa copy pokazuje na istu fizičku tabelu kao original.

3. Kod čuvanja, backend namerno gasi DDL za copy item  

   - ključna linija: `manual_update = params['manual_update'] or delta.f_copy_of.value` u builder.py  
   - kada je `f_copy_of` postavljen, `manual_update` postaje `true`.

4. Kada je `manual_update` `true`, ne pravi se/menja tabela u bazi  

   - `insert_item_query` radi `create_table` samo ako `not manual_update`: admin.
     py  
   - `update_item_query` radi `alter/recreate` samo ako `not manual_update`: admin.
     py  
   - `delete_item_query` radi `drop table` samo ako `not manual_update`: admin.py

5. Ipak se metadata snima normalno  

   - `apply_delta` i `commit` i dalje rade: builder.py, builder.py  
   - znači `sys_items/sys_fields` definicije kopije se čuvaju, ali fizička tabela ostaje ista.

Zašto ti izgleda da sve ide u postojeći item/tabelu:

- zato što copy item i original dele isti `f_table_name`, a copy režim zabranjuje
  DDL promene nad šemom.  
- praktično, COPY je više "alternativna definicija/interfejs nad istom tabelom", ne
  "clone sa novom tabelom".

Dodatni signali u kodu da je to koncept:

- u više lookup i detail mesta `copy` koristi `owner_id = f_copy_of (original)`
  umesto svog `id`: admin.js, admin.js, admin.js  

- postoji i zaštita "used in copy definitions" pri brisanju polja: builder.py

Zaključak:

- ponašanje koje si opisao nije slučajno; COPY u AppBuilder-u je copy definicije
  nad postojećom tabelom, ne automatsko kreiranje nove fizičke tabele.  

- "obično ponašanje" (create/alter/drop table) vraća se tek kada item nije copy,
  tj. nema `f_copy_of` i `manual_update` je `false`.

## Rutiranje

U Jam.py praktično postoje 2 nivoa "rutiranja":

- **HTTP rutiranje** (vrlo malo, hardkodovano)
  Ovo je ono klasično: **index**, **login**, **logout**, plus **api/upload/ext**.
  U dokumentaciji koju imaš stoji da `App.call` grana ulaz baš na index/api i da su index/login/logout posebni endpointi u wsgi sloju: code workflow.md:43, code workflow.md:118.
  Znači: ovde nemaš veliki router kao u Django/Flask aplikacijama.

- **Aplikaciono "rutiranje"** (glavnina posla)
  Posle učitavanja stranice, frontend uglavnom radi POST na `/api`.
  Unutar jednog uniformnog JSON paketa method određuje šta se radi (`load/open/apply/server/print`), a `task_id + item_id` biraju objekat nad kojim radiš.

Dakle, tok aplikacije se ne vodi kroz URL rute, nego kroz RPC dispatch nad task/item objektima.

Kako to izgleda za index/login/logout:

- `index`: služi kao ulazna web stranica (bootstrap klijenta).
- `login`: endpoint za autentikaciju i uspostavu session-a.
- `logout`: endpoint za gašenje session-a.

Nakon toga, skoro sav "realan rad" ide preko `/api`, ne preko novih URL ruta.

**Suština**: u Jam.py `index/login/logout` su infrastrukturne rute, a poslovna navigacija i logika su prebačene u `/api + task/item` mehanizam.

Konkretno si dobio operativnu stabilnost u web režimu, ne "lepši routing".

Najkraće: u novijem Jam.py routing je i dalje minimalan, ali je oko njega dodat ceo zaštitni sloj koji V5 praktično nije imao u ovom obliku.

### Šta je konkretno dobijeno

- Stabilan `session` i `auth` tok

- Jasno odvojeni ulazi za `index`, `login` i `logout`.

- `Session cookie` se čuva sa bezbednijim pravilima (HttpOnly, SameSite), uz
  opcionalnu proveru IP/UUID u safe modu.  
  Efekat: manji rizik zloupotrebe sesije i manje "čudnih" stanja korisnika.

- Detekcija zastarele verzije klijenta  

- Server vraća status tipa PROJECT_MODIFIED kada se build ili metadata promene.  
  Klijent to prepoznaje i ne nastavlja slepo sa starim stanjem.  
  Efekat: manje korupcije podataka kada više korisnika radi dok se sistem menja.

- Maintenance režim bez rušenja korisničkog toka  
  Tokom održavanja server vraća maintenance status, klijent prikazuje poruku i
  pokušava ponovni load.  
  Efekat: kontrolisano ponašanje umesto random grešaka.

- Jedan uniforman API dispatch  
  Poslovni pozivi idu kroz jednu tačku sa metodama load/open/apply/server/print,
  umesto mnogo URL ruta.  
  Efekat: jednostavniji protokol klijent-server i lakše centralno logovanje/obrada
  grešaka.  

- Precizniji apply model u master-detail lancu  

- master_applies je eksplicitno deo metadata i klijentske logike.  
  Ako je uključeno, detail apply se ne završava parcijalno, već se propagira na
  master tok.  
  Efekat: bolja transakciona konzistentnost dokumenta.  

- Kontrolisan reload projekta u radu  
  Build/version mehanizam proverava da li je kod/metadata promenjen i po potrebi
  reloaduje task.  
  Efekat: manje potrebe za ručnim restartima i manje nekonzistentnih instanci
  procesa.  

- Bolja skalabilnost konekcija  
  Uveden connection pool model (QueuePool/NullPool) i jasna granica commit-a kod
  apply.  
  Efekat: predvidljivije ponašanje pod opterećenjem.

### Zašto deluje da u V5 "nije bilo ničega od toga"

U V5 je više stvari bilo implicitno i monolitno u okviru framework toka.

U novijem modelu su iste ili slične potrebe "izvučene na površinu" jer je web runtime ozbiljnije tretiran kao distribuiran sistem (više tabova, više korisnika, hot izmene, održavanje).

### Glavne promene

Top 3 promene koje su najbitnije baš za tvoj master/detail + PostgreSQL scenario:

- Master-driven `apply` preko `master_applies`
  Dobitak: detail izmene se ne snimaju parcijalno, već ulaze u isti tok sa master zapisom.
  Zašto je bitno u PostgreSQL: smanjuje šansu za "orphan" detail redove i loš redosled upisa kada imaš FK, NOT NULL, unique i slična ograničenja.
  Praktično: dokument tipa header + stavke se čuva kao jedna celina, umesto više odvojenih pokušaja.
- Centralizovan `delta apply` sa jednim commit-om
  Dobitak: server rekonstruiše delta i radi apply kroz jedinstveni transakcioni put.
  Zašto je bitno u PostgreSQL: ACID radi u tvoju korist, pa su greške konzistentne i lakše za rollback/ponovni pokušaj.
  Praktično: manje "poluupisanih" stanja kada više tabela učestvuje u jednom snimanju.
- Zaštita od zastarelog klijenta i sesije
  Dobitak: statusi tipa not logged / modified sprečavaju da stari tab ili stara verzija klijenta upisuje pogrešan delta paket.
  Zašto je bitno u PostgreSQL: manje konfliktnih upisa i manje situacija gde korisnik nesvesno radi nad promenjenom strukturom ili podacima.
  Praktično: stabilniji rad kada ima više korisnika ili kad se sistem menja "u hodu".

Konkretan primer end-to-end za jedan dokument (master + detail), sa fokusom na šta se zaista dešava:

1. **Korisnik ulazi u aplikaciju**

   HTTP ulaz ide na `index/login/logout` sloj, a ne direktno na poslovne rute (opisano u sekciji "Rutiranje").

2. **Klijent inicijalizuje task**

   Front šalje `load` preko jedinstvene API tačke (opisano u sekciji "/api endpoint").

3. **Otvaranje master objekta**

   Kada korisnik otvori master, ide `open` poziv za taj item; detail se ne učitava automatski u istom koraku (opisano u sekciji "Master detail odnosi").

4. **Korisnik menja master + detail redove**

   Edit radi lokalno u klijentskom `dataset/change_log` sloju.
   Važna razlika: `post` ne ide odmah na server, nego zatvara lokalni edit i beleži delta promene (opisano u sekciji "Tok čuvanja zapisa").

5. **Klik na OK / apply_record**

   Klijent prvo odradi lokalni `post`, pa onda `apply` (opisano u sekciji "Tok čuvanja zapisa").

6. **master_applies odluka**

   - Ako je detail podešen sa `master_applies`, `apply` se propagira ka master toku (opisano u sekciji "master_applies atribut Details objekata").
   - Ovo je ključ za "čuvanje dokumenta kao celine".

7. **Slanje delta paketa na server**

   - Klijent šalje `apply` kroz API sa promenama i hook parametrima (opisano u sekciji "Tok čuvanja zapisa").
   - API je uniforman paket `method/task/item/params/modification` (opisano u sekciji "/api endpoint").

8. **Server apply pipeline**

   - Server rekonstruiše delta, izvršava `on_apply` logiku i ako nema override ide default `apply_delta`, pa commit konekcije (opisano u sekciji "Tok čuvanja zapisa").

9. Povrat odgovora i sinhronizacija klijenta

   - Klijent obrađuje response, ažurira `change_log`, pali `on_after_apply` i osvežava UI (opisano u sekciji "Tok čuvanja zapisa").

Šta je ovde najvažnije za praksu:

1. `Post` je lokalni korak, `apply` je serverski transakcioni korak.
2. `master_applies` utiče na to da li detail čuvaš samostalno ili kroz master
   transakciju.
3. Ako želiš "dokument kao celina", onda je `master-propagacija` ključna.

### Sa kakvim relacijama raditi

Evo praktičnog testa kako da odlučiš: V5-stil ili čist FK-stil.

- Biraj V5-stil ako ti je primarni cilj fleksibilan dokumentni tok
  - Jedan Detail treba da "služi" više različitih Master tipova.
  - Želiš brže modelovanje kroz OWNER logiku i task tree.
  - Prihvataš da deo integriteta držiš kroz framework pravila i event kod.

- Biraj čist FK-stil ako ti je primarni cilj stroga baza
  - Hoćeš da PostgreSQL sam garantuje integritet relacija.
  - Radiš ozbiljne analitike, integracije i SQL izvan Jam.py.
  - Važno ti je da svaki odnos bude eksplicitan i proverljiv na DB nivou.

Moja preporuka za tvoj slučaj:

- Za novu poslovnu aplikaciju na PostgreSQL: idi sa FK-first.
- V5-stil koristi samo gde realno imaš polymorphic detail scenario koji bi sa
  čistim FK postao nepraktičan.

Pragmatičan hibrid (često najbolji):

- Core tabele: čisti FK, unique, not null, check.
- Dokumentni UI tok: `master_applies` uključen gde treba atomsko čuvanje.
- Event kod: samo za poslovna pravila, ne za osnovni referencijalni integritet.

Crvene zastavice da si previše otišao u V5-stil:

- Ne možeš lako napisati jasan SQL JOIN bez posebnog "tumačenja" OWNER polja.
- Integritet zavisi od "ako se event okine".
- Teško testiraš šta je tačno dozvoljena relacija.

Crvene zastavice da si previše rigidan FK-stil:

- Previše pomoćnih tabela za ono što je u domenu prirodno dokumentno ponašanje.
- Svaka mala promena workflow-a traži migraciju šeme.
- UI postaje komplikovaniji nego što posao zahteva.

### Kada šta praktično koristi

Ako Jam.py upravlja šemom kroz svoj model, nemoj se oslanjati na stroge FK kao na primarni mehanizam.

Ako ti ručno vodiš PostgreSQL šemu, možeš imati FK, ali moraš testirati Jam.py tokove snimanja i brisanja da se ne sudare sa njima.

Zašto ova razlika:

U tvojoj dokumentaciji stoji da framework istorijski ide bez FK u svojoj logici i admin šemi: Understanding the Excellence of the Jam.py Framework.md:168.

Jam.py često koristi soft-delete i svoj owner model, što nije isto što i klasična RI logika baze.

Zato sam predložio FK-first kao arhitektonski cilj za PostgreSQL, ali to važi kada ti držiš DB dizajn pod svojom kontrolom, ne kada 100% prepuštaš Jam.py da vodi relacije.

Praktično kako da odlučiš:

- Jam.py-first projekat: minimalno FK ili bez FK, pravila kroz Jam.py evente.
  DB-first projekat: uvedi FK, ali:
  - koristi `deferrable` ograničenja gde treba,
  - pazi na `delete tokove` (soft-delete vs hard FK),
  - testiraj `apply` master/detail scenarije.

Dakle, nisam mislio "Jam.py nativno podržava FK pa samo uključi". Mislio sam: možeš ih imati u PostgreSQL ako ti vodiš šemu, ali to je svesna integracija koju moraš testirati.

Odlično. Evo praktičan checklist za bezbedno uvođenje FK u Jam.py + PostgreSQL.

1. **Kreni od jedne pilot relacije**  
   Ne uvodi FK svuda odjednom. Uzmi jednu master-detail vezu i validiraj ceo tok.

2. **Uvek uključi `master_applies` za dokumentne relacije**  
   Za header + stavke relacije, cilj je jedan `apply` tok i jedna transakciona celina.

3. **Očisti postojeće podatke pre dodavanja FK**  
   Pre ALTER TABLE proveri orphan redove, null vrednosti i duplikate. FK uvodi tek kada je data clean.

4. **Prvo dodaj indekse pa onda FK**  
    Na child kolonama napravi indeks pre FK, da izbegneš usporenja i lock probleme.

5. **Razdvoji `soft-delete` i hard-delete pravila**  
   Ako Jam.py logika koristi DELETED flag, ne oslanjaj se na ON DELETE CASCADE kao glavni mehanizam.

6. **Za kritične relacije koristi DEFERRABLE**  
   Kod kompleksnih upisa u jednoj transakciji koristi DEFERRABLE INITIALLY DEFERRED gde je potrebno.

7. **Testiraj četiri obavezna scenarija**
   - Insert master + detail,
   - update oba,
   - delete master,
   - delete detail.
   Testiraj i UI `apply` i direktan SQL rollback.

8. **U eventima drži poslovna pravila, ne osnovni integritet**  
   Referencijalni integritet neka čuva baza, a evente koristi za domen (npr. status dokumenta, limiti, obračuni).

9. **Uvedi migracije korak-po-korak sa rollback planom**  
   Svaki FK u posebnoj migraciji, sa jasnim planom povratka ako produkcija pokaže problem.

10. **Uključi monitoring FK grešaka od prvog dana**
    Loguj i prati violacije ograničenja odmah posle deploy-a, da brzo vidiš gde Jam.py tok i DB pravila nisu usklađeni.

## Master detail odnosi

### Učitavanje master i details podataka

Pri `open` za master, framework ne učitava detail zapise u istom API pozivu. Details se otvaraju posebno, klasično, kada UI zatreba.

Kako to vidiš u kodu:

- `open` request učitava dataset samo za trenutni item
- Front šalje `open`: item.js:1778
- Backend to obrađuje kao `select_records` za taj jedan item: wsgi.py:731, items.
  py:232
- Posle open, `_do_after_load` popunjava samo `this._dataset` tog item-a
- Nema automatskog loop-a koji otvara detail-e u tom trenutku: item.js:1787  
  Details se otvaraju naknadno, na UI događaj
- Kada se promeni master red (`on_after_scroll`), aktivni detail radi `detail.open
  (true)`: item.js:3664
- Pri izboru detail taba takođe ide `detail.open(true)`: item.js:3634
- `__expanded` nije "učitaj details", nego "expanded prikaz/lookup polja"  
  Parametar ide kroz QueryData.expanded: common.py:435  
  DB ga koristi u grananju za lookup/order/group SQL, ne za master-detail eager load: db.py:762, db.py:801, db.py:861

**Zaključak**:

- Master i details nisu jedan "joined payload" pri običnom `open`.
- Master se otvara prvo.
- Details se otvaraju odvojeno (lazy, po potrebi u UI toku).

### master_applies atribut Details objekata

U Jam.py postoje 2 koraka koja se često mešaju:

- `post`: lokalno zatvaranje edit stanja i upis u client `change_log`
- `apply`: slanje delta izmena na server i DB commit

Poenta atributa `master_applies` je: ko je "vlasnik" `apply` operacije u master-detail vezi.

Iz tvoje dokumentacije se vidi upravo ovo pravilo: kada je detail sa `master_applies`, `apply` se propagira na master (Code workflow/code workflow.md).

Kako to praktično radi:

- `master_applies = 1` (`true`)
  - Ako si na detail-u i klikneš `apply`, iz detail konteksta se ne šalje poseban `apply` request na server; tok se propagira na master (u V5 je UX često delovao kao "ne događa se ništa").
  - Master i svi relevantni detail-i ulaze u isti `apply` ciklus.
  - Šalje se jedan konsolidovani delta ka serveru.
  - U praksi dobijaš jednu transakcijsku celinu za ceo dokument (header + stavke).

- `master_applies = 0` (`false`)
  - Detail može da radi `apply` samostalno.
  - Master i detail nisu "prisilno" u istoj `apply` celini.
  - Korisno kad detail ima svoj životni ciklus, ali nosi veći rizik nekonzistentnog
    redosleda snimanja ako zavise jedan od drugog.

### Šta se dešava sa Details kada obrišeš Master bez FK u bazi?

- Ako ideš kroz standardni Jam.py tok `delete_record` pa `apply`, Master zapis ide
  na `soft delete` (`DELETED = 1`) ako je `soft_delete` uključen.  

- Framework zatim prolazi kroz povezane detail skupove i njih označava sa `DELETED
  = 1`. To radi i rekurzivno za dublje nivoe detalja.
  
- Ako `soft_delete` nije uključen, radi hard delete i za master i za odgovarajuće detail redove.

Znači, u standardnom toku ne ostavlja "žive" detalje uz obrisan master.

### Da li se DELETED automatski filtrira kod lookup dohvatanja?

- Za normalni `open` nad item-om: da, automatski se dodaje uslov `DELETED = 0`, osim
  ako eksplicitno filtriraš po DELETED.
- Za lookup listu (typeahead/select): lookup item se otvara kao regularan dataset,
  pa u praksi takođe dobija `DELETED = 0` i obrisani redovi ne nude se za novi izbor.
  
  Bitna nijansa: kod `expanded` join prikaza lookup vrednosti u gridu, join sam po sebi ne dodaje automatski filter nad lookup tabelom na `DELETED = 0`. To znači:
  
  - Stari zapis može i dalje imati prikazanu lookup vrednost iako je lookup red
    soft-obrisan.
  
  - Novi izbor kroz lookup listu obično taj red više ne nudi.
  
  - Ovo je verovatno namerno ponašanje zbog kompatibilnosti i "istorijskog" prikaza
    podataka.

Za tvoju ideju da `master_applies` držiš stalno uključeno na Details: Za dokumentni model (master + details) to je najbezbedniji izbor i slažem se sa tim. Time dobijaš jednoobrazan `apply` tok i manje parcijalnih stanja.
