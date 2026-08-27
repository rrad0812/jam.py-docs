
# Jam.py FAQ

## 01 What is the difference between catalogs and journals

When a new project is created, its task tree has the following groups: `Catalogs`, `Journals` and `Reports`.

`Catalogs` and `Journals` belong to the `Item Group` type and have the same functional purpose.

We created them to distinguish between two types of data items:

- data items that contain information of catalog type such as customers, organizations, tracks, etc. - Catalogs

- data items that store information about events recorded in some documents, such as invoices, purchase orders, etc. - Journals

`System` group is created automatically with `History` item creation.

## 02 How to upgrade an already created project to a new version of jampy

To upgrade an existing V7 project to a new package you must update the package.

You can do it using `pip`.

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

This makes `the Copy` feature useful when multiple database tables share the same structure. A single source `Item` defines the fields, while each copied Item can work with a different table.

It is possible to detach the table from the source.

**Note**:

Think of a copied `Item` as sharing its schema definition with the source Item, while maintaining its own database table and data.

## 06 What are foreign keys used for

Foreign keys that you can create in the Jam.py V5 Application Builder, prevent deletion of a record in the lookup table if a reference to it is stored in the lookup field.

For example, when a foreign key is created on the “Customer” field for “Invoices” item, user won’t be able to delete a customer in “Customers” catalog if a reference to it is stored in “Invoices”.

The soft delete attribute of the lookup item must be set to false (see Item Editor Dialog ) for the lookup field to appear in the Foreign Keys Dialog

The creation/deleting support for foreign keys is dropped in Jam.py V7.
