[Table of content](index.md) | [Getting started >](getting_started.md)

# The overview

The Jam.py framework is an open source, event driven framework for developing 
web database applications. 

**It uses**:

* `Python` and `Werkzeug` network library on the server side,
* `Javascript`, `JQuery` and `Bootstrap` on the client side,
* `Application Builder` for graphical development on the server and client side,
* a dozen predefined css themes,
* `SQLite`, `MySQL`, `PostgrSQL`, `Firebird`, `MS SQL Server` and `Oracle` databases,
  for work in a new or existing database schema,
* `LibreOffice` templates for create reports, in the `pdf`, `excel` or `csv` formats,
* `data-aware` controls.

**We can create**:

* dashboards as well as big applications with complex business logic,
* filters for filtering data by one or more selected conditions,
* search criteria for data by selected conditions, 
* user actions associated with predefined javascript buttons,
* cascade-linked combo-boxes with incremental search capabilities,
* pages for the authentication and authorization actions,
* help systems associated with any of controls in a from,
* applications, with foreign python and javascript libraries.

## Jam.py has powerful library

### Creating form of item in the console of browser

Suppose we created a `customers` item in The Application Builder.

Now we can create `customers` form `modal`:

```javascript
  task.customers.view()
```

or, `modeles`:

```javascript
  task.customers.view($('#content'))
```

**Note**: This code we can get run in the web javascript console of our browsers.

### Work with choosen records of item

We have `selection` checkboxes for work with choosen records:

* Turn on selection checkboxes:
  
  ```javascript 
  
    task.customers.selections = []
  ```

* After changing their value in the customer form, we check their values

  ```javascript
    task.customers.selections
  ```

* Turn off it:
  
  ```javascript
    task.customers.selections = undefined
  ```

### Play with records of dataset:

```javascript
  // Go to the next record in the current dataset
  task.customers.next()
  
  // Get value of the lastname field of the current dataset record
  task.customers.lastname.value
  
  // Get edit form for the current dataset record
  task.customers.create_edit_form()
  
  // Go to the next record in the dataset
  task.customers.next()
  
  // Let we insert a blanc record in the dataset, 
  // and get edit form for the current dataset record
  task.customers.insert_record()
  
  // Let we set value of the lastname field of the current dataset record
  task.customers.lastname.value = 'Yushev'
  
  // Let we save value of the current record in the database
  task.customers.apply_record()
```

## Event-driven Jam.py nature  

Let we write the `on_after_insert` event handler of the item, with the `unitprice` field:
  
```javascript
  item.unitprice.value = 1;
```  
    
From now on, each newly added record will have predefined value of the `unitprice` field equally 1.0.

> [!Note]
> The `on_after_insert` is right place for set up `default value` for any of user item defined fields.


## Conclusion

The Jam.py is an `All in the browser` framework. Complete development of the application is possible within a browser.

It developed and put to use under the terms of a modified BSD license, by Andrew Yushev, from Tula, Russia.

See all this and many other on http://jam-py.com/.

[Table of content](index.md) | [Getting started >](getting_started.md)