
# Understanding the Excellence of the Jam.py Framework

## Introduction

Jam.py's official documentation does not explain, in any practical or sufficiently detailed way, how the framework was designed and how it actually works internally. For that reason, I chose to focus on the most important file in the system: the `admin.sqlite` database. Even so, that file alone does not reveal the whole story.

I decided to contribute to the Jam.py documentation because I believe that Jam.py deserves far more attention than it usually receives, and that `admin.sqlite` lies at the center of the framework and is key to understanding what makes Jam.py different from other Python web frameworks.

## Basics of the Jam.py framework

Jam.py is:

- web based,
- application-oriented,
- object-oriented,
- multi-database capable,
- capable of report generation,
- event-driven,
- designed for building business applications

framework.

There are many frameworks like this, one might argue.

What distinguishes Jam.py from the vast majority of Python frameworks, and arguably from almost all of them, are the following characteristics:

- metadata-driven architecture,
- a hierarchically organized set of objects in an application task tree,
- RPC-based communication between the front-end and back-end,
- low-code development model.

### Metadata-driven framework

By definition, a metadata-driven framework does not require the developer to write code for every part of the application model and state. Instead, it relies on auxiliary tools that define application metadata, usually stored in a database or a JSON file.

> [!Note]
> `admin.sqlite` is the place where the auxiliary Jam.py application, `AppBuilder`, stores information about application objects. That metadata is later interpreted by the runtime components of Jam.py and used to dynamically construct application elements.

This is the core idea: “data about application objects.” That is what metadata is today, and this is precisely why Jam.py deserves the label “metadata-driven.”

In Jam.py, there are no `Model` or `View` classes as a standard framework requirement, neither written by the user nor generated automatically.

There is, however, a specific component that can be described as a metadata deserializer. It reads metadata from the `admin.sqlite` database and turns it into runtime objects that describe both the GUI and the database where the application’s working data is stored.

Everything else is similar to other frameworks, except for the part that, in many other frameworks, would be called the Controller. In Jam.py, this responsibility is placed in event handlers on both the server and the client, and it is embedded in the following framework feature.

### Hierarchically organized set of objects in a task tree

This second characteristic may distinguish Jam.py even more sharply from other frameworks, because it defines the behavior of application objects built from metadata stored in the `admin.sqlite` database.

The behavior of the application can be extended in different ways, as in the `Behavior` pattern. The difference is that, when event handlers are arranged hierarchically over a tree of application objects, and Jam.py uses the following basic structure:

- task
  - group
    - item

then the developer naturally gains the ability to chain event handlers in one direction or the other, depending on the need.

This is the second major Jam.py idea: by placing the default code in the `task` object, the application can already function. Then, by making minor adjustments in the event handlers of individual items, the developer builds a powerful client-server application.

This is also possible with the `Behavior` pattern, but then the programmer must deal with inheritance and manually configure behavior from code, which can become cumbersome and error-prone.

### RPC principle for connecting the front-end and back-end

After the previous chapter, the first natural question is:

“How are client-side event handlers connected to server-side code?”

The answer is RPC. The framework provides only a small number of methods—just four—for communication between the front-end and back-end. In this model, the client sends a request to the server, the server executes the relevant Python method, and then returns the result or an error to the client.

### Low-code framework

Taken together, these characteristics lead to the conclusion that Jam.py is a no-code or low-code framework.

I have personally built an application for managing non-trivial records of technical devices, with a database of 39 tables and only 15 lines of client-side code.

## Structure of the `admin.sqlite` database

### Basic structure

`admin.sqlite` is a classic SQLite file with the following basic data (for the database from the Demo application):

```sql
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
```

The basic SQLite behavior setting is:

```sql
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
```

## Database schema - `admin.sqlite`

### Introduction to the `admin.sqlite` schema

The database contains the following tables:

- SYS_COUNTRIES
- SYS_FIELD_LOOKUPS
- SYS_FILTERS
- SYS_ITEMS
- SYS_LANGUAGES
- SYS_PARAMS
- SYS_REPORT_PARAMS
- SYS_TASKS
- SYS_FIELDS
- SYS_FIELD_PRIVILEGES
- SYS_INDICES
- SYS_LANGS
- SYS_LOOKUP_LISTS
- SYS_PRIVILEGES
- SYS_ROLES
- SYS_USERS

Here comes the story of some basic design decisions in building the framework. These will also be respected in the main database when building tables related to the application task tree objects:

- There are no foreign keys at the framework level.
- The primary key of a table is a single integer field named `ID`.
- Each table contains an integer field named `DELETED`.
- In most tables, the presence of these two fields is noticeable:
  
  - `OWNER_ID`
  - `OWNER_RECORD_ID`
  
- Field names are either simple or have an `F_` prefix.

#### No foreign keys

This is the first surprise. Jam.py is in a conflict with foreign keys; all tables are without referential integrity protection. Consequently, the exact `admin.sqlite` schema is questionable.

Even worse, if you generate a project database schema from Jam.py, it will not have foreign keys.

Support for foreign keys existed in v5. Apparently, the code change in v7 to support different states and relationships between application tree objects caused the removal of support for foreign keys.

I cannot regret that. In my opinion:

“The foreign key is a civilizational achievement in the development of databases.”

But what can we do? This is the situation.

With the `DELETED` field, which has a value of `1` or `0`, we get a small advantage over classic referential integrity:

- We can perform schema import/export and schema upgrade operations without the problems caused by referential integrity. This is especially relevant for databases where RI is not declarative.
- Even though the reference has been deleted, the record remains there, only marked as `DELETED`. If the code that handles the relation ignores this, lookup values can still remain accessible.

#### Simple PK with surrogate values

I also consider this a civilizational achievement. This is simply the best option for choosing a PK type, even though it can introduce additional requirements for `UNIQUE` fields.

I recently participated in a discussion on the Jam.py Google group, in which a user declared a field as a PK of type `KEYS`. To be honest, I did not fully understand everything, but it was supposed to serve an internal small `m:n` relation, implemented not by a simple join but by code and the `CONTAINS` operator.

That makes sense, but not for `KEYS` to be the primary key. It should simply be moved to another field, which would be of type `KEYS`, and the relation should be created through that field rather than through the PK.

#### The `DELETED` field

This field marks a record as deleted, and all subsequent SQL statements should ignore it using the appropriate filter.

A little more work, but several advantages:

- Help with RI management, as described above.
- Records remain there; we simply pretend they are not there.
- Easier cleanup later (deleted records can be permanently removed more easily).

The only thing I am suspicious about is the audit trail.

#### The `OWNER_ID` and `OWNER_REC_ID` fields

This is another difference in Jam.py.

Both fields were introduced to support a specific way of creating accounting documents:

- One table can be a `Master` for several others, which are called `Details` and are attached by simple “substitution”.
- One table `Detail` can be “subsumed” under several different `Master` tables.

All of this without writing a single line of code.

In doing this, Jam.py sets these two fields based on the position of `Details` relative to `Master`, and vice versa.

This means that one `Details` table may contain records related to different `Master` tables, and obviously the resolution requires `OWNER_ID`. `OWNER_REC_ID` is evidently a foreign key for the connection to the primary key of the `Master` table record.

And vice versa, one `Master` table may contain records related to records in multiple `Details` tables.

With version v7, we moved to somewhat more common types of relations, which we have to “tame”, i.e. it is not enough to simply underline them; some code must be added independently. At minimum, the definition of `owner_rec_id` as a foreign key must be set, as in any other framework.

With the code from v5, it was quite difficult to achieve a simple large `m:n` relation with an associative table in the middle. The reason is simple: v5 treated relations between records as pure, while an associative table assumes the presence of two foreign keys in a single record.

And so, because of all this, it is a good thing that they got Jam.py v7.

Of course, the question is: “How do I get the behavior from v5?” Well, by simply defining a foreign key in multiple `Details` tables to one `Master` table, in other words, by defining the `OWNER_REC_ID` field.

OK, but there is a catch-22 here.

“How will Jam.py know between which tables the relation is established if there are no foreign keys?” Here is our good friend `OWNER_ID`. We simply point it where it should go and that is it.

And where is “where it should go”? Now we come to another Jam.py-specific feature that eliminates the need for a foreign key.

> [!Note]
> The `OWNER_ID` field, which is a foreign key (Integer), points to the `SYS_ITEMS` table and its primary key, i.e. to the record defining the project task tree object, which in our case is the `MASTER` table.

#### Field name only or with the `F_` prefix

Here we come to a very important part, and we make an assumption. I really have not read or discussed this before.

In my opinion, the variety in the field names can be explained as follows:

- Primary key and foreign key fields (loosely speaking) are named without the `F_` prefix.
- All other fields (regular fields) have the `F_` prefix.

## Detailed schema of `admin.sqlite`

This story about the structure would not be complete without its detailed representation.

To do that, we divide it as follows:

- `admin.sqlite` core subsystem
  - SYS_FIELDS
  - SYS_FIELD_LOOKUPS
  - SYS_FILTERS
  - SYS_INDICES
  - SYS_ITEMS
  - SYS_LOOKUP_LISTS
  - SYS_PARAMS
  - SYS_REPORT_PARAMS
  - SYS_TASKS

- `admin.sqlite` language subsystem
  - SYS_COUNTRIES
  - SYS_LANGUAGES
  - SYS_LANGS

- `admin.sqlite` privileges subsystem
  - SYS_FIELD_PRIVILEGES
  - SYS_PRIVILEGES
  - SYS_ROLES
  - SYS_USERS

I do not think we will discuss *the language subsystem* at all, because this part of the database is no longer used; it has all been moved and developed in `lang.sqlite`.

As for *the privileges subsystem*, maybe that will be in a future sequel.

## Database schema notes - core subsystem

### `SYS_PARAMS` table

We can simplify the analysis further: the `SYS_PARAMS` table has three special fields:

```sql
"ID" INTEGER PRIMARY KEY, # primary key of the table
"DELETED" INTEGER,        # flag indicating whether the record is deleted
"TASK_ID" INTEGER,        # link to the SYS_TASKS table or to the task definition in the SYS_ITEMS table. 
```

And then there are many ordinary fields related to the configuration of the Jam.py application. These are, of course, very important, but since 100% of the screenshots come from the `Parameters` modal window, we will not deal with them further here.

> [!Note]
> The inclusion of the `TASK_ID` lookup field indicates that the author was thinking about the possibility of building a `multi_task` application.

The `TASK_ID` field determines which task or application is in question.

### `SYS_TASK` table

If we look at the structure of the `SYS_TASK` table:

```sql
"ID" INTEGER PRIMARY KEY,
"DELETED" INTEGER,
"TASK_ID" INTEGER,
"F_MANUAL_UPDATE" INTEGER, 
"F_DB_TYPE" INTEGER,
"F_DB_NAME" TEXT,
"F_ALIAS" TEXT,
"F_LOGIN" TEXT,
"F_PASSWORD" TEXT,
"F_ENCODING" TEXT,
"F_HOST" TEXT,
"F_PORT" TEXT,
F_NAME TEXT,
F_ITEM_NAME TEXT,
F_LANGUAGE INTEGER,
F_SERVER TEXT,
F_CUSTOM_CONNECTION INTEGER,
F_PYTHON_LIBRARY INTEGER,
F_DSN TEXT
```

We see many signs that the framework is ready and equipped for multi-tasking possibilities.

The `TASK_ID` field should probably not be there, but if you want to keep track of multiple tasks, all of which except the root have as their owner one of the tasks already present, then this is the way to represent it.

This is a way to create an internal connection, or recursion, in relational databases. However, since Jam.py usually works with only one task, meaning `TASK_ID = 1` always, this is not worth discussing.

All the other fields indicate, as already noted, that there are elements intended to define other tasks, which could be deployed on other hosts and even on other databases.

> [!Note]
> Here I am thinking and writing about centralized management with multiple tasks, not about managing multiple links to different domains, databases, and applications.

### `SYS_FIELD_LOOKUPS` table

This table comes from the beginning of Jam.py. I do not think the code uses it anymore. The data in it resembles the data in the `SYS_ROLES` table.

### `SYS_LOOKUP_LIST` table

A table used to store the programmer’s lookup-list definitions:

```sql
ID INTEGER PRIMARY KEY,
DELETED INTEGER,
F_NAME TEXT,                # field storing the lookup list name
F_LOOKUP_VALUES_TEXT BLOB   # stores a list of {number: string} values representing the lookup list
```

Interestingly, the last field is of type `BLOB`, which means it can accept a list of significant length.

### `SYS_INDICES` table

A table that stores the definition of indexes defined through AppBuilder.

If Jam.py relies on a legacy system, there may be differences between what is in the database and what this table contains.

```sql
"ID" INTEGER PRIMARY KEY,
"DELETED" INTEGER,
"OWNER_ID" INTEGER,
"OWNER_REC_ID" INTEGER,
"TASK_ID" INTEGER,
"F_INDEX_NAME" TEXT,          # index name
"DESCENDING" INTEGER,         # sort direction for the index
"F_FIELDS" BLOB,              # list of fields included in the index
"F_FOREIGN_INDEX" INTEGER,    # leftover from v5
"F_FOREIGN_FIELD" INTEGER,    # leftover from v5
F_UNIQUE_INDEX INTEGER,       # whether the index is UNIQUE
F_FIELDS_LIST TEXT            # another list of fields included in the index
```

Here we see only one new special field, `DESCENDING`, which instructs the database how to create the index, in ascending or descending order.

All other fields are there to help Jam.py create the DDL SQL statement needed to create the index.

The field names show that foreign key support existed at some point.

The `OWNER_ID` field tells us which table in the project database it refers to, via a lookup to the corresponding record in `SYS_ITEMS`.

The `OWNER_REC_ID` field tells us which field to look up if the index is simple. This applies to simple indexes. When the indexes are complex, then one of `F_FIELDS` or `F_FIELDS_LIST` must be used.

The names of the other fields tell you what they do in this table.

### `SYS_FILTERS` table

This is the table where Jam.py stores the filter definition for each item.

```sql
"ID" INTEGER PRIMARY KEY,
"DELETED" INTEGER,
"OWNER_ID" INTEGER,
"OWNER_REC_ID" INTEGER,
"TASK_ID" INTEGER,
"F_INDEX" INTEGER,            # which index is used
"F_FIELD" INTEGER,            # which field enters the filter expression
"F_NAME" TEXT,                # field name
"F_FILTER_NAME" TEXT,         # filter name
"F_DATA_TYPE" INTEGER,        # type of filter field
"F_TYPE" INTEGER,             # type of link operator (probably)
"F_VISIBLE" INTEGER,          # whether it is visible, which allows different display options
                              # for default data presentation
F_HELP BLOB,                  # help text for the filter field
F_PLACEHOLDER TEXT,           # content displayed when the filter field is empty
F_MULTI_SELECT_ALL INTEGER    # ???
```

### `SYS_REPORT_PARAMS` table

Another definition table, this time for creating objects of type `Report Parmeters`.

```sql
"ID" INTEGER PRIMARY KEY,
"DELETED" INTEGER,
"OWNER_ID" INTEGER,
"OWNER_REC_ID" INTEGER,
"TASK_ID" INTEGER,
"F_INDEX" INTEGER,
"F_NAME" TEXT,
"F_PARAM_NAME" TEXT,
"F_DATA_TYPE" INTEGER,
"F_SIZE" INTEGER,
"F_OBJECT" INTEGER,
"F_OBJECT_FIELD" INTEGER,
"F_REQUIRED" INTEGER,
"F_VISIBLE" INTEGER,
"F_ALIGNMENT" INTEGER,
F_ENABLE_TYPEHEAD INTEGER,
F_LOOKUP_VALUES INTEGER,
F_MASTER_FIELD INTEGER,
F_HELP BLOB,
F_PLACEHOLDER TEXT,
F_OBJECT_FIELD1 INTEGER,
F_OBJECT_FIELD2 INTEGER,
F_MULTI_SELECT INTEGER,
F_MULTI_SELECT_ALL INTEGER,
F_CALC_ITEM INTEGER,
F_CALC_FIELD INTEGER,
F_CALC_OP INTEGER,
F_READ_ONLY INTEGER,
F_NOT_NULL INTEGER,
F_CHECK_BEFORE_DELETING INTEGER
```

Although awkward, this table contains details about report parameters. These are the definitions of parameters that can appear in the fields of the `Report Parameters` modal window.

Since it is very similar to the following `SYS_FIELDS` table, we will not explain it separately.

### `SYS_FIELDS` table

Now we are getting to a slippery area. Since this database was built with the assumption of dynamic relationships, to be 100% sure about the relationships that appear here, we would need to search the entire code.

Of course we are not going to do that, but errors in my assumptions are possible.

```sql
"ID" INTEGER PRIMARY KEY,
"DELETED" INTEGER,
"OWNER_ID" INTEGER,
"OWNER_REC_ID" INTEGER,
"TASK_ID" INTEGER,
"F_NAME" TEXT # field name,
"F_FIELD_NAME" TEXT # field name in code,
"F_DATA_TYPE" INTEGER # field type,
"F_SIZE" INTEGER  # field width for text type,
"F_OBJECT" INTEGER # lookup object,
"F_OBJECT_FIELD" INTEGER # lookup field,
"F_EDIT_FIELD" INTEGER # editor for the field; may be multiline,
"F_MASTER_FIELD" INTEGER # the master field number,
"F_REQUIRED" INTEGER # whether the field is required,
"F_CALCULATED" INTEGER # whether the field is calculated,
"F_DEFAULT" INTEGER # whether the field has a default value,
"F_READ_ONLY" INTEGER # whether it is read-only,
"F_ALIGNMENT" INTEGER # alignment setting for the field content,
F_ENABLE_TYPEHEAD INTEGER # whether incremental search is enabled,
F_LOOKUP_VALUES INTEGER # ???
F_LOOKUP_VALUES_TEXT BLOB # ???
F_DEFAULT_VALUE TEXT # default value in the field,
F_HELP BLOB # help text for the field,
F_PLACEHOLDER TEXT # default content placed in empty field space,
F_OBJECT_FIELD1 INTEGER # lookup object of the lookup object,
F_OBJECT_FIELD2 INTEGER # lookup object of the lookup object lookup object,
F_MULTI_SELECT INTEGER,
F_MULTI_SELECT_ALL INTEGER,
F_DB_FIELD_NAME TEXT # database field name,
F_MASK TEXT # forced formatting of field content,
F_DEFAULT_LOOKUP_VALUE INTEGER # default lookup value of the field,
F_IMAGE_EDIT_WIDTH INTEGER, # image field geometry definition
F_IMAGE_EDIT_HEIGHT INTEGER,
F_IMAGE_VIEW_WIDTH INTEGER,
F_IMAGE_VIEW_HEIGHT INTEGER,
F_IMAGE_PLACEHOLDER TEXT, # content placed in empty image space
F_FILE_DOWNLOAD_BTN INTEGER, # definitions related to file fields
F_FILE_OPEN_BTN INTEGER,
F_FILE_ACCEPT TEXT,
F_IMAGE_CAMERA INTEGER,
F_CALC_ITEM INTEGER,  # detail object for calculated field
F_CALC_FIELD INTEGER, # calculated field
F_CALC_OP INTEGER,    # aggregate function for the calculated field
F_NOT_NULL INTEGER # whether the field can be NOT NULL in the database
F_TEXTAREA INTEGER # the field has a multiline input element,
F_DO_NOT_SANITIZE INTEGER # whether sanitization is applied to the field,
F_CHECK_BEFORE_DELETING INTEGER # check whether it is referenced before deletion,
F_COPY_OF INTEGER # whether it was created by copy,
F_CALC_LOOKUP_FIELD INTEGER # lookup field of the detail object of the calculated field.
```

### `SYS_ITEMS` table

At the very end, I leave the queen of all tables in `admin.sqlite`, which is `SYS_ITEMS`.

Probably next to `SYS_FIELDS`, this is the most important meta table in the Jam.py framework, where many relationships and values important for the functioning of this framework are defined.

This is the complete definition of an object, item, or node in the application task tree.

First of all, during the development of the framework, the need arose to deepen the internal structure, so the author made a joke with the values of the primary key in this table, reducing it to `-1`, `-2`, `-3`, `-4`, as needed.

However, we can observe the following structure:

- Project
  - Users
  - Roles
  - Task
    - Demo
      - Catalogs
        - Artist
        - Album
        - Genres
        - Tracks
      - Journals
        - Invoices
          - Invoice Tables
      - Details
        - Invoice_Tables
      - Reports
        - Print Invoices
        - Customer Purchases
        - Customer List

This entire structure is regulated by the fields `ID`, `PARENT`, and `HAS_CHILDREN`.

The `TYPE_ID` field stores different types of items depending on their position within the task, while `TABLE_ID` is a classifier of table type. Both of these classifiers serve the framework’s internal needs.

```sql
"ID" INTEGER PRIMARY KEY, # all other lookup fields from `SYS_ITEMS` point here
"DELETED" INTEGER,
"PARENT" INTEGER, # if there is a parent, this is the parent record `ID`
"TASK_ID" INTEGER, # `ID` of the task to which it belongs
"TYPE_ID" INTEGER,  # different values for task, group, item, report group, etc.
"TABLE_ID" INTEGER,
"HAS_CHILDREN" INTEGER,
"F_NAME" TEXT # item name,
"F_ITEM_NAME" TEXT # item name in code,
"F_TABLE_NAME" TEXT # name of the corresponding table in the database,
"F_VIEW_TEMPLATE" TEXT # name of the item’s view template,
"F_EDIT_TEMPLATE" TEXT # name of the item’s edit template,
"F_FILTER_TEMPLATE" TEXT # name of the item’s filter template,
"F_VISIBLE" INTEGER # whether the item is visible,
"F_CLIENT_MODULE" BLOB # server-side Python code,
"F_SERVER_MODULE" BLOB # server-side Python code,
"F_INFO" BLOB # information about the item,
"F_WEB_CLIENT_MODULE" BLOB # client-side JS code for the item,
"F_SOFT_DELETE" INTEGER # whether soft delete is enabled,
"F_INDEX" INTEGER # ?,
"F_EXTERNAL" INTEGER # ?,
F_VIRTUAL_TABLE INTEGER # ? whether the item is a virtual table,
F_JS_EXTERNAL INTEGER # name of the external JS file to import,
F_JS_FILENAME TEXT # name of the local JS file to import,
F_PRIMARY_KEY INTEGER # whether the item has a primary key,
F_DELETED_FLAG INTEGER # whether it has a deleted field,
F_MASTER_ID INTEGER # master ID linking the master table,
F_MASTER_REC_ID INTEGER # `master_rec_id` is the internal link to the master table,
F_JS_FUNCS BLOB # JS code for the item,
F_EDIT_LOCK INTEGER # whether item hard-lock is applied on record edit,
F_GEN_NAME TEXT # generic name of the item,
F_KEEP_HISTORY INTEGER # whether history of changes is tracked for item data,
SYS_ID INTEGER # ???,
F_SELECT_ALL INTEGER # probably for building copy tables,
F_RECORD_VERSION INTEGER # whether optimistic locking is used during editing,
F_MASTER_FIELD INTEGER # which master field the item uses,
F_COPY_OF INTEGER # copy of which table; internal relation,
F_MASTER_APPLIES INTEGER # ???.
```

## Conclusion

The fact that Jam.py is built on a different conceptual foundation than most of the better-known Python frameworks today already places it among the exceptional ones.

In Jam.py, there is no real need for:

- writing model code,
- writing view code,
- writing raw SQL code, or doing so only rarely as in reports,
- routing, because login and user-flow scenarios can be handled without it,
- writing huge amounts of repetitive code.

The framework is both robust and efficient.

The code that still needs to be written is usually limited to event handlers at the task, group, or item level, on either the client or the server side.

When building project task tree objects, it is important to keep in mind that these objects—and their methods or attributes—can be called from anywhere in the task tree.

In practice, this is one of the rare ways to extend the framework from the inside: by creating and adding commonly used items to the project task tree, such as email functionality or similar services, placing them in the skeleton task tree by default, and then reusing them in the application-specific task tree code.

Every other modification requires changes in AppBuilder, which is not easy, largely because of the framework’s underlying architecture based on Remote Procedure Calls.

Fortunately, the framework already includes the necessary administrative components, so the remaining task is simply to use them well.
