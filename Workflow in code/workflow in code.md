
# `Jam.py` projekat iznutra

## JS server() metod

Dakle, JS lanac je:

- `task.server(...)` →
- `abstr_item.server()` →
- `send_request('server', ...)` →
- `task.process_request()` →
- `POST /api`.

## Demistifikacija amin.sqlite

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
- `create_groups/create_items` mapiraju redove na Group/Item instance i pune server_code iz `f_server_module`: task.py:109, task.py:138, task.py:158.
- Posle toga `bind_items` i `compile_all` povezuju i kompajliraju server kod: task.py:339.
- Kompajliranje izvršava kod iz `server_code` i kači funkcije na `item/task`: items.py:665.

### Izbor glavne projektne baze

- Admin builder čita `sys_tasks` i postavlja `task_db_info/task_db_type`: builder.py:96.
- `load_task` preuzima to u runtime task i kreira `DB adapter/pool`: task.py:342.

### Korisnik bira objekat iz task stabla projekta

- HTTP ulaz ide kroz `App.call` i rutira se na `index/api`: wsgi.py:206, wsgi.py:241, wsgi.py:300.
- Frontend inicijalno šalje `load` preko `/api`: task.js:330, task.js:110.
- Backend `on_api` parsira JSON paket i zove `get_response`: wsgi.py:667, wsgi.py:726.
- Za `load` vraća `task.get_info` + `templates/settings/locale/language`: wsgi.py:520.

### Kada korisnik otvori konkretan objekat

- Frontend šalje `open`: item.js:1778.
- Backend `get_response` za `open` zove `select_records`: wsgi.py:727, items.py:232.
- `select_records` radi hookove i SQL read kroz `execute_open`: items.py:240, items.py:178.

### Kada frontend zove task.server("ime_funkcije")

- JS metoda `server` je ovde: abstr_item.js:373.
- Ona šalje `server` request: abstr_item.js:389.
- Backend to rutira na `server_func/getattr` i izvrši zahtevanu funkciju po imenu: wsgi.py:731,
  wsgi.py:743.

**Suština**:

- `admin.sqlite` je metadata baza podataka iz koje framework sastavlja Task/Item objekte i njihov server kod; posle toga klijent radi preko `/api`, a za konkretan objekat ide - `open` ili `server` poziv kroz `item_id` i `ime funkcije`.

## /api endpoint

Kroz wsgi.py izložena je praktično jedna API ulazna tačka: putanja `/api`.

Šta tačno prima:

- POST JSON paket u formatu [method, task_id, item_id, params, modification]
- Parsiranje je u wsgi.py

Šta je izloženo kao method:

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
   - logika u wsgi.py

2. `item_id` bira objekat:
   - `task.item_by_ID(item_id)`
   - wsgi.py

3. method bira operaciju:
   - `open` -> `item.select_records(...)`
   - `apply` -> `item.apply_changes(...)`
   - `server` -> `server_func(item, func_name, params)`
   - `print` -> `item.print_report(...)`
   - `load` -> `init_client(...)`
   - wsgi.py

Server poziv po imenu:

- `server_func` radi `getattr(obj, func_name)` pa poziva funkciju
- wsgi.py

Frontend strana koja gađa `/api`:

- JSON POST ide iz `process_request` u task.js
- URL je eksplicitno api u task.js
- `task.server(...)` pakuje server method poziv u abstr_item.js

Van `/api` su odvojeni endpointi:

- `upload` na `/upload`: wsgi.py
- `ext` na `/ext`: wsgi.py
- `index/login/logout` routing u wsgi.py

Ako želiš, sledeće mogu da ti dam konkretan primer jednog realnog request tela za `load`, `open` i `server`, pa da vidiš 1:1 kako izgleda payload i response.

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
    "status": 1,
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
    "__filters": [["id", "=", 10, 0]],
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
    "status": 1,
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

- server => server_func: wsgi.py
- getattr i poziv funkcije: wsgi.py

Tipičan response:

```json
{
  "result": {
    "status": 1,
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

**Kratko**:

`post` = lokalno zatvaranje edit-a i priprema delta.

`apply` = slanje delta na server + DB commit + povrat update-a.

## Master detail odnosi

**Kratak odgovor**: Pri `open` za master, framework ne učitava detail zapise u istom API pozivu. Details se otvaraju posebno, klasično, kada UI zatreba.

Kako to vidiš u kodu:

- `open` request učitava dataset samo za trenutni item
  - Front šalje `open`: item.js:1778
  - Back to obrađuje kao `select_records` za taj jedan item: wsgi.py:731, items.py:232
  - Posle open, `_do_after_load` popunjava samo `this._dataset` tog item-a
  - Nema automatskog loop-a koji otvara detail-e u tom trenutku: item.js:1787  
    Details se otvaraju naknadno, na UI događaj
  - Kada se promeni master red (`on_after_scroll`), aktivni detail radi `detail.open(true)`: item.
    js:3664
  - Pri izboru detail taba takođe ide `detail.open(true)`: item.js:3634
  - `__expanded` nije “učitaj details”, nego “expanded prikaz/lookup polja”  
    Parametar ide kroz QueryData.expanded: common.py:435  
    DB ga koristi u grananju za lookup/order/group SQL, ne za master-detail eager load: db.py:762, db.py:801, db.py:861

Zaključak:

- Master i details nisu jedan “joined payload” pri običnom `open`.
- Master se otvara prvo.
- Details se otvaraju odvojeno (lazy, po potrebi u UI toku).

## COPY item

Imam ključni signal: COPY ne pravi novu tabelu nego označava item kao kopiju (`f_copy_of`) i tada backend namerno preskače DDL; proveravam još par mesta gde se to eksplicitno koristi da potvrdimo namenu.

Odlično si to primetio, i analiza koda potvrđuje da je to uglavnom namerno ponašanje COPY mehanizma.

Šta COPY zapravo radi u v7:

1. COPY u UI pravi novi zapis u `sys_items` kao kopiju postojećeg item-a

   - klik na copy dugme: admin.js  
   - postavlja `f_copy_of` na originalni item: admin.js  
   - odmah pokreće biranje/punjenje polja iz izvora: admin.js

2. Novi copy item preuzima tehničke atribute izvora, uključujući tabelu  

   - pri copy kliku kopiraju se gotovo sva polja item-a (osim par sistemskih referenci): admin.js  
   - to uključuje i `f_table_name/f_gen_name`, pa copy pokazuje na istu fizičku tabelu kao original.

3. Kod čuvanja, backend namerno gasi DDL za copy item  

   - ključna linija: `manual_update = params[manual_update] or delta.f_copy_of.value` u builder.py  
   - kada je `f_copy_of` postavljen, `manual_update` postaje `true`.

4. Kada je `manual_update` `true`, ne pravi se/menja tabela u bazi  

   - `insert_item_query` radi `create_table` samo ako `not manual_update`: admin.py  
   - `update_item_query` radi `alter/recreate` samo ako `not manual_update`: admin.py  
   - `delete_item_query` radi `drop table` samo ako `not manual_update`: admin.py

5. Ipak se metadata snima normalno  

   - `apply_delta` i `commit` i dalje rade: builder.py, builder.py  
   - znači `sys_items/sys_fields` definicije kopije se čuvaju, ali fizička tabela ostaje ista.

Zašto ti izgleda da sve ide u postojeći item/tabelu:

- zato što copy item i original dele isti `f_table_name`, a copy režim zabranjuje DDL promene nad šemom.  
- praktično, COPY je više “alternativna definicija/interfejs nad istom tabelom”, ne “clone sa novom tabelom”.

Dodatni signali u kodu da je to koncept:

- u više lookup i detail mesta `copy` koristi `owner_id = f_copy_of (original)` umesto svog `id`: admin.js, admin.js, admin.js  

- postoji i zaštita “used in copy definitions” pri brisanju polja: builder.py

Zaključak:

- ponašanje koje si opisao nije slučajno; COPY u AppBuilder-u je copy definicije nad postojećom tabelom, ne automatsko kreiranje nove fizičke tabele.  

- “obično ponašanje” (create/alter/drop table) vraća se tek kada item nije copy, tj. nema `f_copy_of` i `manual_update` je `false`.
