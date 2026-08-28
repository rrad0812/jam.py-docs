
# Jampy FAQ

## 01 What is the difference between catalogs and journals

From version V7 Jampy makes almost no distinction between `Catalogs` and `Journals` groups:

- Both have the same common fields,
- the same attributes can be set on both,
- both contain objects that have a coresponding table in the database.
- The only thing to note is that Catalogs are intended for storing master data for the business factors of the application, while Journals are intended for storing transaction data for the business factors of the application.

In addition to these two, there is also a `Reports` group that stores defined objects of reports on the business factors of the application, which by their nature are not connected to tables in the database. Here it is necessary to distinguish cases when a Report builds its own table to produce output, as is already common in OLAP systems.

`System` group is created automatically with `History` item creation.

## 02 How to upgrade an already created project to a new version of jampy

If you’re using Linux, Mac OS X or some other flavour of Unix, enter the command:

```sh
sudo pip install --upgrade jam.py-v7
```

If you’re using Windows, start a command shell with administrator privileges and run the command

```sh
pip install --upgrade jam.py-v7
```

## 03 Can I use other libraries in my application

You can add javascript libraries to use them for programming on the client side.

It is better to place them in the `js` folders of the `static` directory of the project. And refer to them using the src attribute in the `<script>` tag of the `Index.html` file.

For example, Demo project uses `Chart.js` library to create a dashboard:

```html
<script src="/static/js/Chart.min.js"></script>
```

On the server side you can import python libraries to your modules.

For example the mail item server module import smtplib library to send emails:

```py
import smtplib
```

## 04 When printing a report I get an ods file instead of pdf

When a report is generated the server application first creates an `ods` file.

If extension attribute of the report is set to ‘pdf’ or any other format except ‘ods’, the application first creates an `ods` file and then uses LibreOffice in “headless” mode to convert the ods file to that format.

If LibreOffice is currently running on the server this conversion may not happen. You must close LibreOffice on the server for the conversion to take place.

## 05 What Is a Copied Item

When creating a new `Item`, you can use the `Copy` feature.

A copied `Item` is not an independent copy. Instead, it remains linked to the source `Item`.

The copied `Item` inherits selected fields from the source `Item` and can add any fields that are later added to the source. Because of this relationship, the copied `Item` cannot define its own fields - the source `Item` remains the single place where the field definition is maintained.

During the copy process, you may choose a different database table for the copied `Item`. The copied Item performs all CRUD operations on its own table while continuing to use the field definitions inherited from the source `Item`.

This makes the `Copy` feature useful when multiple database tables share the same structure. A single source `Item` defines the fields, while each copied Item can work with a different table.

It is possible to detach the table from the source.

**Note**:

Think of a copied `Item` as sharing its schema definition with the source Item, while maintaining its own database table and data.

## 06 What are foreign keys used for

Definition:

> [!Note]
>
> - A foreign key is primarily a database mechanism that protects so-called referential integrity.  
> - Referential integrity refers to the integrity of records in a referenced database table.  
> - In addition, a foreign key determines the rules of behavior of the referenced and current table records during update or deletion operations in the referenced table.  
> - In short, referential integrity is a tool for maintaining stable and predictable behavior of relationships between tables in a database.

In version V5, Jampy fully supported referential integrity, i.e. foreign keys for the case of RESTRICT update/delete operations on a referenced record.

Since version V7, Jampy does not support foreign keys, or referential integrity, but it is still possible to define referential integrity directly in the database.

As a replacement, Jampy offers DELETED fields, which can achieve similar effects at the application level, with this approach reducing referential integrity problems when maintaining the database.

We also have an `f_check_before_deleting` field attribute with which, when set, we get very similar behavior as if we had a foreign key in RESTRICT mode.

> [!Note]
>
> It should always be kept in mind that Jampy's CRUD system is a document type, meaning that either everything or nothing happens on tables connected by relations during update/delete operations, which corresponds to the CASCADE foreign key model for the case of updating/deleting a referenced record.
>
> But here too there is a difference in behavior for the case where `f_master_applies` of the object (item) attribute is set. In that case Jampy CRUD treats each table in the relationship separately.
