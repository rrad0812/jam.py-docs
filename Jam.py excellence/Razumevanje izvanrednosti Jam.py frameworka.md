
# Razumevanje izvrsnosti Jam.py okvira

## Uvod

Zvanična dokumentacija Jam.py-ja ne objašnjava, ni na koji praktičan niti dovoljno detaljan način, kako je frejmvork dizajniran i kako on zapravo funkcioniše interno. Iz tog razloga, odlučio sam da se fokusiram na najvažniju datoteku u sistemu: `admin.sqlite` bazu podataka. Čak i tako, ta datoteka sama po sebi ne otkriva celu priču.

Odlučio sam da doprinesem Jam.py dokumentaciji jer verujem da Jam.py zaslužuje mnogo više pažnje nego što obično dobija, a `admin.sqlite` je u srži frejmvorka i ključna je za razumevanje šta Jam.py razlikuje od drugih Python veb frejmvorka.

## Osnove Jam.py frejmvorka

Jam.py je:

- veb baziran,
- orijentisan na aplikativnu primenu,
- objektno orijentisan,
- sposoban za više baza podataka,
- sposoban za generisanje izveštaja,
- vođen događajima,
- dizajniran za izgradnju poslovnih aplikacija

radni okvir.

Moglo bi se reći da postoji mnogo takvih okvira.

Ono što razlikuje Jam.py od velike većine Pajton frejmvorka, a verovatno i od skoro svih njih, jesu sledeće karakteristike:

- arhitektura vođena metapodacima,
- hijerarhijski organizovan skup objekata u stablu zadataka aplikacije,
- RPC-bazirana komunikacija između front-enda i bek-enda,
- model razvoja sa niskim kodom.

### Izvrsnost broj 1 - Frejmvork zasnovan na metapodacima

Po definiciji, okvir vođen metapodacima ne zahteva od programera da piše kod za svaki deo modela i stanja aplikacije. Umesto toga, oslanja se na pomoćne alate koji definišu metapodatke aplikacije, obično sačuvane u bazi podataka ili JSON datoteci.

> [!Note]
>
> `admin.sqlite` je mesto gde pomoćna Jam.py aplikacija, AppBuilder, čuva informacije o objektima aplikacije. Te metapodatke kasnije interpretiraju komponente Jam.py-ja tokom izvršavanja i koriste se za dinamičko konstruisanje elemenata aplikacije.

Ovo je osnovna ideja: "podaci o objektima aplikacije". To je ono što su metapodaci danas, i upravo zato Jam.py zaslužuje oznaku "vođen metapodacima".

U Jam.py, ne postoje klase Model ili View kao standardni zahtev frejmvorka, niti ih je napisao korisnik, niti su automatski generisane.

Međutim, postoji specifična komponenta koja se može opisati kao `deserijalizator metapodataka`. Ona čita metapodatke iz `admin.sqlite` baze podataka i pretvara ih u objekte za vreme izvršavanja koji opisuju i grafički korisnički interfejs i bazu podataka u kojoj su smešteni radni podaci aplikacije.

Sve ostalo je slično drugim frejmvorcima, osim dela koji bi se u mnogim drugim frejmvorcima nazivao Controller. U Jam.py-u, ova odgovornost je smeštena na obrađivače događaja i na serveru i na klijentu, i ugrađena je u sledeću funkciju frejmvorka.

### Izvrsnost broj 2 - Hijerarhijski organizovan skup objekata u stablo taska projekta

Ova druga karakteristika možda još više razlikuje Jam.py od drugih frejmvorka, jer definiše ponašanje objekata aplikacije izgrađenih od metapodataka sačuvanih u `admin.sqlite` bazi podataka.

Ponašanje aplikacije može se proširiti na različite načine, kao u `Behavior` obrascu. Razlika je u tome što su ovde obrađivači događaja hijerarhijski raspoređeni preko stabla objekata aplikacije. Jam.py koristi sledeću osnovnu strukturu:

- task
  - grupa
    - objekat

tada programer prirodno stiče mogućnost da lančano povezuje obrađivače događaja u jednom ili drugom smeru, u zavisnosti od potrebe.

Ovo je druga glavna ideja Jam.py-a: postavljanjem podrazumevanog koda u objekat `task`, aplikacija već može da funkcioniše. Zatim, pravljenjem manjih podešavanja u obrađivačima događaja pojedinačnih objekata task stabla projekta, programer gradi moćnu klijent-server aplikaciju.

Ovo je takođe moguće sa `Behavior` šablonom, ali tada programer mora da se bavi nasleđivanjem i ručnim konfigurisanjem ponašanja iz koda, što može postati glomazno i sklono greškama.

### Izvrsnost broj 3 - RPC princip za povezivanje front-enda i bek-enda

Nakon prethodnog poglavlja, prvo prirodno pitanje je:

"Kako su obrađivači događaja na strani klijenta povezani sa kodom na strani servera?"

Odgovor je RPC. Frejmvork pruža mali broj metoda — samo četiri — za komunikaciju između front-enda i bek-enda. U ovom modelu, klijent šalje zahtev serveru, server izvršava relevantnu Pajton metodu, a zatim vraća rezultat ili grešku klijentu.

### Izvrsnost broj 4 - Frejmvork sa malo koda

Uzete zajedno, ove karakteristike dovode do zaključka da je Jam.py frejmvork sa malo ili bez koda.

Lično sam napravio aplikaciju za upravljanje netrivijalnim zapisima tehničkih uređaja, sa bazom podataka od 39 tabela i samo 15 linija klijentskog koda.

## Struktura admin.sqlite baze podataka

### Osnovna struktura

`admin.sqlite` je klasična SQLite datoteka sa sledećim osnovnim podacima (za bazu podataka iz demo aplikacije):

SQLite version 3.53.4 2026-07-24 19:02:57
database page size:  1024
write format:        1
read format:         1
reserved bytes:      0
file change counter: 17458
database page count: 141
freelist page count: 9
schema cookie:       1124
schema format:       4
default cache size:  0
autovacuum top root: 0
incremental vacuum:  0
text encoding:       1 (utf8)
user version:        0
application id:      0
software version:    3046001
number of tables:    16
number of indexes:   0
number of triggers:  0
number of views:     0
schema size:         6528
data version         2

Osnovno podešavanje ponašanja SQLite-a je:

      attach_create on
       attach_write on
           comments on
          defensive on
            dqs_ddl off
            dqs_dml off
        enable_fkey off
        enable_qpsg off
     enable_trigger on
        enable_view on
     fts3_tokenizer off
          fp_digits 17
 legacy_alter_table off
 legacy_file_format off
     load_extension on
   no_ckpt_on_close off
     reset_database off
  reverse_scanorder off
    stmt_scanstatus off
        trigger_eqp off
     trusted_schema off
    writable_schema off

## Šema baze podataka admin.sqlite

### Uvod u admin.sqlitešemu

Baza podataka sadrži sledeće tabele:

- SYS_COUNTRIES
- SYS_FIELD_LOOKUPS
- SYS_FILTERS
- SYS_ITEMS
- SYS_LANGUAGES
- SYS_PARAMS
- SYS_REPORT_PARAMS
- SYS_TAKS
- SYS_FIELDS
- SYS_FIELDS_PRIVILEGES
- SYS_INDICES
- SYS_LANGS
- SYS_LOOKUP_LISTS
- SYS_PRIVILEGES
- SYS_ROLES
- SYS_USERS

Evo priče o nekim osnovnim dizajnerskim odlukama pri izgradnji okvira. One će takođe biti poštovane u glavnoj bazi podataka prilikom izgradnje tabela povezanih sa objektima stabla zadataka aplikacije:

- Ne postoje strani ključevi na nivou frejmvorka.

- Primarni ključ tabele je jedno celobrojno polje pod nazivom ID.

- Svaka tabela sadrži celobrojno polje pod nazivom DELETED.

- U većini tabela, primetno je prisustvo ova dva polja:

  - OWNER_ID
  - OWNER_RECORD_ID

- Imena polja su ili jednostavna ili imaju F_prefiks.

#### Bez stranih ključeva

Ovo je prvo iznenađenje. Jam.py je u sukobu sa stranim ključevima; sve tabele su bez zaštite referencijalnog integriteta. Shodno tome, tačna `admin.sqlite` šema je upitna.

Još gore, ako generišete šemu baze podataka projekta iz Jam.py, ona neće imati strane ključeve.

Podrška za strane ključeve postojala je u verziji 5. Izgleda da je promena koda u verziji 7 radi podrške različitih stanja i odnosa između objekata stabla aplikacije dovela do uklanjanja podrške za strane ključeve.

Ne mogu da žalim zbog toga. Po mom mišljenju:

"Strani ključ je civilizacijsko dostignuće u razvoju baza podataka."

Ali šta možemo da uradimo? Takva je situacija.

Sa DELETED poljem, koje ima vrednost 1ili 0, dobijamo malu prednost u odnosu na klasični referencijalni integritet:

- Možemo da izvršimo operacije uvoza/izvoza šeme i nadogradnje šeme bez problema izazvanih referencijalnim integritetom. Ovo je posebno relevantno za baze podataka
  gde RI nije deklarativna.
- Iako je referenca obrisana, zapis ostaje tamo, samo označen kao DELETED. Ako kod koji obrađuje relaciju ignoriše ovo, vrednosti pretrage i dalje mogu ostati
  dostupne.

#### Jednostavan PK sa surogatnim vrednostima

Takođe smatram da je ovo civilizacijsko dostignuće. Ovo je jednostavno najbolja opcija za izbor tipa PK, iako može uvesti dodatne zahteve za UNIQUE polja.

Nedavno sam učestvovao u diskusiji na Google grupi Jam.py, u kojoj je korisnik deklarisao polje kao PK tipa KEYS. Iskreno, nisam sve u potpunosti razumeo, ali je trebalo da služi internoj maloj `m:n` relaciji, implementiranoj ne jednostavnim spajanjem već kodom i `CONTAINS` operatorom.

To ima smisla, ali ne da KEYS bude primarni ključ. Trebalo bi jednostavno da se premesti u drugo polje, koje bi bilo tipa KEYS, a relacija bi trebalo da se kreira preko tog polja, a ne preko PK.

#### Polje DELETED​

Ovo polje označava zapis kao obrisan i svi naredni SQL izrazi treba da ga ignorišu koristeći odgovarajući filter.

Malo više posla, ali nekoliko prednosti:

- Pomoć u upravljanju RI, kao što je gore opisano.
- Zapisi ostaju tamo; mi se jednostavno pravimo da ih nema.
- Lakše čišćenje kasnije (izbrisani zapisi se mogu trajno lakše ukloniti).

Jedino što mi je sumnjivo je revizorski trag.

#### Polja OWNER_ID i OWNER_REC_ID​

Ovo je još jedna razlika u Jam.py.

Oba polja su uvedena da bi podržala specifičan način kreiranja računovodstvenih dokumenata:

- Jedna tabela može biti `Master` za nekoliko drugih, koje se pozivaju `Details` i pridružuju jednostavnim "podmetanjem".
- Jedna tabela `Detail` može biti "podmetnuta" pod nekoliko različitih `Master` tabela.

Sve ovo bez pisanja i jedne linije koda.

Pri tome, Jam.py postavlja ova dva polja na osnovu položaja `Details` u odnosu na Master, i obrnuto.

To znači da jedna `Details` tabela može da sadrži zapise koji se odnose na različite `Master` tabele, i očigledno je da razrešenje zahteva `OWNER_ID`. `OWNER_REC_ID` je očigledno strani ključ za vezu sa primarnim ključem `Master` zapisa tabele.

I obrnuto, jedna `Master` tabela može da sadrži zapise povezane sa zapisima u više različitih `Details` tabela.

Sa verzijom v7, prešli smo na nešto uobičajenije tipove relacija, koje moramo da definišemo, tj. nije dovoljno samo ih podvući; neki kod se mora dodati nezavisno. Minimum, `OWNER_REC_ID` se mora podesiti kao definicija stranog ključa, kao i u bilo kom drugom okviru.

Sa kodom iz verzije 5, bilo je prilično teško postići jednostavnu veliku `m:nr` elaciju sa asocijativnom tabelom u sredini. Razlog je jednostavan: verzija 5 je tretirala relacije između zapisa kao čiste, dok asocijativna tabela pretpostavlja prisustvo dva strana ključa u jednom zapisu.

I zato je, zbog svega ovoga, dobro šta smo dobili u Jam.py v7.

Naravno, pitanje je: "Kako da dobijem ponašanje iz v5?" Pa, jednostavnim definisanjem stranog ključa u više `Details` tabela za jednu `Master` tabelu, drugim rečima, definisanjem `OWNER_REC_ID` polja.

U redu, ali ovde postoji začkoljica.

"Kako će Jam.py znati između kojih tabela je uspostavljena relacija ako nema stranih ključeva?" Evo našeg dobrog prijatelja `OWNER_ID`. Jednostavno ga usmerimo gde treba da ide i to je to. A gde je to "Gde bi trebalo da ide“? Sada dolazimo do još jedne Jam.py specifične karakteristike koja eliminiše potrebu za stranim ključem.

> [!Note]
>
> Polje `OWNER_ID`, koje je strani ključ (Integer), ukazuje na `SYS_ITEMS` tabelu i njen primarni ključ, odnosno na zapis koji definiše objekat stabla taska projekta, što je u našem slučaju tabela `MASTER`.

#### Samo naziv polja ili sa F_prefiksom

Ovde dolazimo do veoma važnog dela i pravimo jednu pretpostavku. Zaista nisam ovo ranije čitao niti o tome raspravljao.

Po mom mišljenju, raznovrsnost u nazivima polja može se objasniti na sledeći način:

- Polja primarnog ključa i stranog ključa (uslovno govoreći) se imenuju bez F_ prefiksa.
- Sva ostala polja (redovna polja) imaju F_prefiks.

## Detaljna šema admin.sqlite

Ova priča o strukturi ne bi bila potpuna bez njenog detaljnog prikaza.

Da bismo to uradili, podelimo ga na sledeći način:

### admin.sqlite osnovni podsistem

- SYS_FIELDS
- SYS_FIELD_LOOKUPS
- SYS_FILTERS
- SYS_INDICES
- SYS_ITEMS
- SYS_LOOKUP_LISTS
- SYS_PARAMS
- SYS_REPORT_PARAMS
- SYS_TASKS

### admin.sqlite jezički podsistem

- SYS_COUNTRIES
- SYS_LANGUAGES
- SYS_LANGS

### admin.sqlitepodsistem privilegija

- SYS_FIELDS_PRIVILEGES
- SYS_PRIVILEGES
- SYS_ROLES
- SYS_USERS

Mislim da ovde uopšte nećemo raspravljati o jezičkom podsistemu, jer se ovaj deo baze podataka više ne koristi; sve je premešteno i razvijeno u `langs.sqlite`.

Što se tiče podsistema privilegija, možda će to biti u nekom budućem nastavku.

## Beleške o šemi baze podataka - osnovni podsistem

### SYS_PARAMS tabela

Analizu možemo dodatno pojednostaviti:

`SYS_PARAMS` tabela ima tri posebna polja:

- "ID" INTEGER PRIMARY KEY, # primary key of the table
- "DELETED" INTEGER,        # flag indicating whether the record is deleted
- "TASK_ID" INTEGER,        # link to the SYS_TASKS table or to the task definition in the SYS_ITEMS table.

A onda postoje mnoga obična polja vezana za konfiguraciju aplikacije Jam.py. Ona su, naravno, veoma važna, ali pošto 100% preslikna iz `Parameters` modalnog prozora, projekta, nećemo se njima ovde dalje baviti.

> [!Note]
>
> Uključivanje `TASK_ID` lookup polja ukazuje na to da je autor razmišljao o mogućnosti izrade multi_task aplikacije.

Polje `TASK_ID` određuje koji je task u pitanju.

### SYS_TASK tabela

Ako pogledamo strukturu tabele `SYS_TASK`:

- "ID" INTEGER PRIMARY KEY,
- "DELETED" INTEGER,
- "TASK_ID" INTEGER,
- "F_MANUAL_UPDATE" INTEGER,
- "F_DB_TYPE" INTEGER,
- "F_DB_NAME" TEXT,
- "F_ALIAS" TEXT,
- "F_LOGIN" TEXT,
- "F_PASSWORD" TEXT,
- "F_ENCODING" TEXT,
- "F_HOST" TEXT,
- "F_PORT" TEXT,
- F_NAME TEXT,
- F_ITEM_NAME TEXT,
- F_LANGUAGE INTEGER,
- F_SERVER TEXT,
- F_CUSTOM_CONNECTION INTEGER,
- F_PYTHON_LIBRARY INTEGER,
- F_DSN TEXT

Vidimo mnogo znakova da je okvir spreman i opreman za mogućnosti obavljanja više taskova.

Polje `TASK_ID` verovatno ne bi trebalo da bude tamo, ali ako želite da pratite više taskova, od kojih svi osim korena imaju za vlasnika jedan od već prisutnih, tada je ovo način da ga predstavite.

Ovo je način kreiranja interne veze, ili rekurzije, u relacionim bazama podataka. Međutim, pošto Jam.py radi samo sa jednim projektnim taskom, što znači `TASK_ID = 1` uvek, ovo nije vredno rasprave.

Sva ostala polja ukazuju, kao što je već napomenuto, da postoje elementi namenjeni definisanju drugih taskova, koji bi mogli biti raspoređeni na drugim hostovima, pa čak i na drugim bazama podataka.

> [!Note]
>
> Ovde razmišljam i pišem o centralizovanom upravljanju sa više taskova, a ne o upravljanju višestrukim vezama ka različitim domenima, bazama podataka i aplikacijama.

### SYS_FIELD_LOOKUPS tabela

Ova tabela potiče sa početka Jam.py. Mislim da je kod više ne koristi. Podaci u njoj podsećaju na podatke u `SYS_ROLES` tabeli.

### SYS_LOOKUP_LIST tabela

Tabela koja se koristi za čuvanje definicija lookup liste programera:

- ID INTEGER PRIMARY KEY,
- DELETED INTEGER,
- F_NAME TEXT,                # field storing the lookup list name
- F_LOOKUP_VALUES_TEXT BLOB   # stores a list of {number: string} values representing the lookup list

Zanimljivo je da je poslednje polje tipa `BLOB`, što znači da može da prihvati listu značajne dužine.

### SYS_INDICES tabela

Tabela koja čuva definiciju indeksa definisanih pomoću AppBuilder-a.

Ako se Jam.py oslanja na zastareli sistem, mogu postojati razlike između onoga što se nalazi u bazi podataka i onoga što ova tabela sadrži.

- "ID" INTEGER PRIMARY KEY,
- "DELETED" INTEGER,
- "OWNER_ID" INTEGER,
- "OWNER_REC_ID" INTEGER,
- "TASK_ID" INTEGER,
- "F_INDEX_NAME" TEXT,          # index name
- "DESCENDING" INTEGER,         # sort direction for the index
- "F_FIELDS" BLOB,              # list of fields included in the index
- "F_FOREIGN_INDEX" INTEGER,    # leftover from v5
- "F_FOREIGN_FIELD" INTEGER,    # leftover from v5
- F_UNIQUE_INDEX INTEGER,       # whether the index is UNIQUE
- F_FIELDS_LIST TEXT            # another list of fields included in the index

Ovde vidimo samo jedno novo specijalno polje, DESCENDING, koje upućuje bazu podataka kako da kreira indeks, u rastućem ili opadajućem redosledu.

Sva ostala polja su tu da pomognu Jam.py-u da kreira DDL SQL izraz potreban za kreiranje indeksa.

Imena polja pokazuju da je podrška za strane ključeve postojala u nekom trenutku.

Polje `OWNER_ID` nam govori na koju tabelu u bazi podataka projekta se odnosi, putem lookupa odgovarajućeg zapisa u `SYS_ITEMS`.

Polje `OWNER_REC_ID` nam govori koje lookup je u pitanju ako je indeks jednostavan. Ovo se odnosi na jednostavne indekse. Kada su indeksi složeni, onda se mora koristiti jedan od F_FIELDS ili F_FIELDS_LIST.

Imena ostalih polja vam govore šta rade u ovoj tabeli.

### SYS_FILTERS tabela

Ovo je tabela u kojoj Jam.py čuva definiciju filtera ( ili više njih ) za svaki objekat u task stablu projekta.

- "ID" INTEGER PRIMARY KEY,
- "DELETED" INTEGER,
- "OWNER_ID" INTEGER,
- "OWNER_REC_ID" INTEGER,
- "TASK_ID" INTEGER,
- "F_INDEX" INTEGER,            # which index is used
- "F_FIELD" INTEGER,            # which field enters the filter expression
- "F_NAME" TEXT,                # field name
- "F_FILTER_NAME" TEXT,         # filter name
- "F_DATA_TYPE" INTEGER,        # type of filter field
- "F_TYPE" INTEGER,             # type of link operator (probably)
- "F_VISIBLE" INTEGER,          # whether it is visible, which allows different display options for default data presentation
- F_HELP BLOB,                  # help text for the filter field
- F_PLACEHOLDER TEXT,           # content displayed when the filter field is empty
- F_MULTI_SELECT_ALL INTEGER    # ???

### SYS_REPORT_PARAMS tabela

Još jedna tabela definicija, ovog puta za kreiranje objekata tipa Report Parmeters.

- "ID" INTEGER PRIMARY KEY,
- "DELETED" INTEGER,
- "OWNER_ID" INTEGER,
- "OWNER_REC_ID" INTEGER,
- "TASK_ID" INTEGER,
- "F_INDEX" INTEGER,
- "F_NAME" TEXT,
- "F_PARAM_NAME" TEXT,
- "F_DATA_TYPE" INTEGER,
- "F_SIZE" INTEGER,
- "F_OBJECT" INTEGER,
- "F_OBJECT_FIELD" INTEGER,
- "F_REQUIRED" INTEGER,
- "F_VISIBLE" INTEGER,
- "F_ALIGNMENT" INTEGER,
- F_ENABLE_TYPEHEAD INTEGER,
- F_LOOKUP_VALUES INTEGER,
- F_MASTER_FIELD INTEGER,
- F_HELP BLOB,
- F_PLACEHOLDER TEXT,
- F_OBJECT_FIELD1 INTEGER,
- F_OBJECT_FIELD2 INTEGER,
- F_MULTI_SELECT INTEGER,
- F_MULTI_SELECT_ALL INTEGER,
- F_CALC_ITEM INTEGER,
- F_CALC_FIELD INTEGER,
- F_CALC_OP INTEGER,
- F_READ_ONLY INTEGER,
- F_NOT_NULL INTEGER,
- F_CHECK_BEFORE_DELETING INTEGER

Iako nezgrapna, ova tabela sadrži detalje o parametrima izveštaja. To su definicije parametara koji se mogu pojaviti u poljima modalnog `Report Parameters` prozora.

Pošto je veoma slična sledećoj `SYS_FIELDS` tabeli, nećemo je posebno objašnjavati.

### SYS_FIELDS tabela

Sada dolazimo do klizavog terena. Pošto je ova baza podataka izgrađena uz pretpostavku dinamičkih veza, da bismo bili 100% sigurni u veze koji se ovde pojavljuju, morali bismo da pretražimo ceo kod.

Naravno da to nećemo uraditi, ali greške u mojim pretpostavkama su moguće.

- "ID" INTEGER PRIMARY KEY,
- "DELETED" INTEGER,
- "OWNER_ID" INTEGER,
- "OWNER_REC_ID" INTEGER,
- "TASK_ID" INTEGER,
- "F_NAME" TEXT # field name,
- "F_FIELD_NAME" TEXT # field name in code,
- "F_DATA_TYPE" INTEGER # field type,
- "F_SIZE" INTEGER  # field width for text type,
- "F_OBJECT" INTEGER # lookup object,
- "F_OBJECT_FIELD" INTEGER # lookup field,
- "F_EDIT_FIELD" INTEGER # editor for the field; may be multiline,
- "F_MASTER_FIELD" INTEGER # the master field number,
- "F_REQUIRED" INTEGER # whether the field is required,
- "F_CALCULATED" INTEGER # whether the field is calculated,
- "F_DEFAULT" INTEGER # whether the field has a default value,
- "F_READ_ONLY" INTEGER # whether it is read-only,
- "F_ALIGNMENT" INTEGER # alignment setting for the field content,
- F_ENABLE_TYPEHEAD INTEGER # whether incremental search is enabled,
- F_LOOKUP_VALUES INTEGER # ???
- F_LOOKUP_VALUES_TEXT BLOB # ???
- F_DEFAULT_VALUE TEXT # default value in the field,
- F_HELP BLOB # help text for the field,
- F_PLACEHOLDER TEXT # default content placed in empty field space,
- F_OBJECT_FIELD1 INTEGER # lookup object of the lookup object,
- F_OBJECT_FIELD2 INTEGER # lookup object of the lookup object lookup object,
- F_MULTI_SELECT INTEGER,
- F_MULTI_SELECT_ALL INTEGER,
- F_DB_FIELD_NAME TEXT # database field name,
- F_MASK TEXT # forced formatting of field content,
- F_DEFAULT_LOOKUP_VALUE INTEGER # default lookup value of the field,
- F_IMAGE_EDIT_WIDTH INTEGER, # image field geometry definition
- F_IMAGE_EDIT_HEIGHT INTEGER,
- F_IMAGE_VIEW_WIDTH INTEGER,
- F_IMAGE_VIEW_HEIGHT INTEGER,
- F_IMAGE_PLACEHOLDER TEXT, # content placed in empty image space
- F_FILE_DOWNLOAD_BTN INTEGER, # definitions related to file fields
- F_FILE_OPEN_BTN INTEGER,
- F_FILE_ACCEPT TEXT,
- F_IMAGE_CAMERA INTEGER,
- F_CALC_ITEM INTEGER,  # detail object for calculated field
- F_CALC_FIELD INTEGER, # calculated field
- F_CALC_OP INTEGER,    # aggregate function for the calculated field
- F_NOT_NULL INTEGER # whether the field can be NOT NULL in the database
- F_TEXTAREA INTEGER # the field has a multiline input element,
- F_DO_NOT_SANITIZE INTEGER # whether sanitization is applied to the field,
- F_CHECK_BEFORE_DELETING INTEGER # check whether it is referenced before deletion,
- F_COPY_OF INTEGER # whether it was created by copy,
- F_CALC_LOOKUP_FIELD INTEGER # lookup field of the detail object of the calculated field.

### SYS_ITEMS tabela

Na samom kraju, ostavljam kraljicu svih metadata tabela u `admin.sqlite`, a to je `SYS_ITEMS`.

Verovatno pored `SYS_FIELDS`, ovo je najvažnija meta tabela u Jam.py okviru, gde su definisani mnogi odnosi i vrednosti važni za funkcionisanje ovog okvira.

Ovo je potpuna definicija objekta, stavke ili čvora u stablu zadataka aplikacije.

Pre svega, tokom razvoja okvira, javila se potreba za produbljivanjem unutrašnje strukture, pa se autor našalio sa vrednostima primarnog ključa u ovoj tabeli, proširujući ga u negativnu stranu, na -1, -2, -3, -4, po potrebi.

Međutim, možemo primetiti sledeću strukturu:

- Projekat                      # Jam.py projekat
  - Korisnici                   # Korisnici definisani u projektu
  - Uloge                       # Uloge definisane u projektu
  - Task                        # Aplikacija projekta
    - Demo                      # Naziv aplikacije projekta  
      - Katalozi                # Grupa katalozi - povezani sa tabela u bazi podataka projekta
        - Umetnik               # Objekat grupe katalozi
        - Album                 #         - II -
        - Žanrovi               #         - II -
        - Pesme                 #         - II -
      - Dnevnici                # Gripa Dnevnici
        - Fakture               # Objekat grupe dnevnici
          - Detalji fakture     #         - II -
      - Detalji                 # Grupa Detalji
        - Detalji fakture       # Objekat grupe detalji
      - Izveštaji               # Grupa izveštaji
        - Odštampaj fakture     # Objekat grupe izveštaji
        - Kupovine kupaca       #          - II -
        - Lista kupaca          #          - II -

Čitava ova struktura je regulisana poljima `ID`, `PARENT`, i `HAS_CHILDREN`.

Pored toga, polje `TYPE_ID` čuva različite tipove objekata u zavisnosti od njihove pozicije unutar taska, dok polje `TABLE_ID` je klasifikator tipa tabela. Oba ova klasifikatora služe internim potrebama frejmvorka.

- "ID" INTEGER PRIMARY KEY, # all other lookup fields from `SYS_ITEMS` point here
- "DELETED" INTEGER,
- "PARENT" INTEGER, # if there is a parent, this is the parent record `ID`
- "TASK_ID" INTEGER, # `ID` of the task to which it belongs
- "TYPE_ID" INTEGER,  # different values for task, group, item, report group, etc.
- "TABLE_ID" INTEGER,
- "HAS_CHILDREN" INTEGER,
- "F_NAME" TEXT # item name,
- "F_ITEM_NAME" TEXT # item name in code,
- "F_TABLE_NAME" TEXT # name of the corresponding table in the database,
- "F_VIEW_TEMPLATE" TEXT # name of the item’s view template,
- "F_EDIT_TEMPLATE" TEXT # name of the item’s edit template,
- "F_FILTER_TEMPLATE" TEXT # name of the item’s filter template,
- "F_VISIBLE" INTEGER # whether the item is visible,
- "F_CLIENT_MODULE" BLOB # server-side Python code,
- "F_SERVER_MODULE" BLOB # server-side Python code,
- "F_INFO" BLOB # information about the item,
- "F_WEB_CLIENT_MODULE" BLOB # client-side JS code for the item,
- "F_SOFT_DELETE" INTEGER # whether soft delete is enabled,
- "F_INDEX" INTEGER # ?,
- "F_EXTERNAL" INTEGER # ?,
- F_VIRTUAL_TABLE INTEGER # ? whether the item is a virtual table,
- F_JS_EXTERNAL INTEGER # name of the external JS file to import,
- F_JS_FILENAME TEXT # name of the local JS file to import,
- F_PRIMARY_KEY INTEGER # whether the item has a primary key,
- F_DELETED_FLAG INTEGER # whether it has a deleted field,
- F_MASTER_ID INTEGER # master ID linking the master table,
- F_MASTER_REC_ID INTEGER # `master_rec_id` is the internal link to the master table,
- F_JS_FUNCS BLOB # JS code for the item,
- F_EDIT_LOCK INTEGER # whether item hard-lock is applied on record edit,
- F_GEN_NAME TEXT # generic name of the item,
- F_KEEP_HISTORY INTEGER # whether history of changes is tracked for item data,
- SYS_ID INTEGER # ???,
- F_SELECT_ALL INTEGER # probably for building copy tables,
- F_RECORD_VERSION INTEGER # whether optimistic locking is used during editing,
- F_MASTER_FIELD INTEGER # which master field the item uses,
- F_COPY_OF INTEGER # copy of which table; internal relation,
- F_MASTER_APPLIES INTEGER # ???.

## Zaključak

Činjenica da je Jam.py izgrađen na drugačijoj konceptualnoj osnovi od većine danas poznatijih Pajton frejmvorka već ga svrstava među izuzetne.

U Jam.py, nema stvarne potrebe za:

- pisanjem model koda,
- pisanjem view koda,
- pisanjem sirovog SQL koda, ili retko kao u izveštajima,
- rutiranjem, jer se scenariji toka prijave korisnika mogu obraditi bez njega,
- pisanjem ogromnih količina ponavljajućeg koda.

Okvir je i robustan i efikasan.

Kod koji još treba napisati obično je ograničen na obrađivače događaja na nivou taska, grupe ili objekta stabla taska projekta, bilo na strani klijenta ili servera.

Prilikom kreiranja objekata stabla taska projekta, važno je imati na umu da se ovi objekti - i njihove metode ili atributi - mogu pozivati sa bilo kog mesta u stablu taska.

U praksi, ovo je jedan od retkih načina za proširenje okvira iznutra: kreiranjem i dodavanjem često korišćenih objekata u stablo taska projekta, kao što su funkcionalnost e-pošte ili slične usluge, njihovim podrazumevanim postavljanjem u skelet stablo taska projekta, a zatim njihovom ponovnom upotrebom u kodu stabla taska specifičnog za aplikaciju.

Svaka druga modifikacija zahteva izmene u AppBuilder-u, što nije lako, uglavnom zbog osnovne arhitekture okvira zasnovane na udaljenim pozivima procedura.

Srećom, okvir već uključuje neophodne administrativne komponente, tako da je preostali zadatak jednostavno ga dobro koristiti.
