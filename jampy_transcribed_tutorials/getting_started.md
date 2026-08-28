[< Overview](overview.md) | [Table of content](index.md) | [Task tree >](task_tree.md)

# The getting started

### Installation

##### Python installation

Jam.py requires *Python*. If it is not installed you can get the latest version 
of Python at [Python 3.7.7](https://www.python.org/downloads/release/python-377) 
From now on, we will only consider working with version 3 of the Python interpreter.

##### Virtual enviroment installation

In a working directory we should create virtual environment:

    > mkdir myenv
    > cd my myenv
    > py -3.7 -m venv .
  
Now, we activating our virtual enviroment:

    >> Scripts\activate.bat
  
##### Jam.py installation

Now, we can installing Jam.py package

    > (myenv) > pip install jam.py

### Creating a Jam.py project

We want to make a simple CRM application. 

First we will create a folder for the new project, and in this folder we will 
run the `jam-project.py` script to create the project structures:

    > (myenv) > mkdir crm
    > (myenv) > cd crm
    > (myenv) > python <where is myenv dir>\bin\jam-project.py


Then run the `server.py` script created by `jam-project.py`.

    > (myenv) > python server.py

To complete the creation of the project we'll start [appbuilder page](http://localhost:8080/builder.html) in any browser:

* First we are selecting the `project language`, 
* After that, the `project parameters` dialog will appear. Now we specify 
  * The `caption` and 
  * The `name` of the project.
  * We will use `SQLITE` database. 
    * Let's name it `crm.sqlite`.
    
We completed the project creation. 

##### The application home page

Let's go to the project - [application home page](http://localhost:8080/),
and refresh it ( F5 ).

Here is now an empty menu. 

##### Creating a catalog's item

* Let's go back to the [appbuilder page](http://localhost:8080/builder.html).

* Go to the `Catalogs` group and create `Customers` item.

* Now we specify its attributes: 
  * `Caption` (that a user can see) 
  * `Name` (used in the programming code) 
  * `Database table name`

* And three text fields 
    
  * `Firstname`, 
  * `Lastname` and 
  * `Phone`.

  As can be seen, while saving the `Customers` item, the server created a 
  corresponding table in the project database.

* Let's go to the [application home page](http://localhost:8080/) and refresh it ( F5 ).

* After that, in the [appbuilder page](http://localhost:8080/builder.html), we open the `Customers` catalog again and for Lastname field specify:
  
  * the `required` attribut,
  * the `default` atribut
  
  For `Catalog` item fields, whose default attribute is set, the default code 
  creates a search functionality. 
    
  > [!Note]
  > This behaviour is changed, all viewed fields are searchable!

##### Added some customers

Let's add a customer.

    John
    Gordon
    999-111-2222

##### Creating a journal's item

Now in the Journals group we create `Contacts` item, and its fields

* `Date`, datetime field, 
* `Customer`, lookup field, that will store a reference to a record in the `Customers` table,
* `Firstname` and `Phone`, lookup fields, but for this fields we specify the `Customer` field as a `master_field` attribute. In this case the Jam.py won't create fields in the database `Contacts` table.
* `Notes`, text field, lenght = 100

Let's open the `Contacts` item in the application home page.

##### Creating a lookup list

Now, for convenience, we duplicate [appbuilder page](http://localhost:8080/builder.html), then select the `Task` and create a `lookup list` `Status`, with next `key`, `value` pairs: 

* 1: Contact
* 2: Meeting
* 3: Proposal
* 4: Close

##### Creating a journal's item - continue

* Let's open `Contacts` item again and add a `lookup field Status` by specifying 
  the `Status lookup list`.
* Let's specify the `Customer` field attributes: 
  
  * `is required`
  * `typeahead`

##### Now we will write some code

Let's create the `on_after_append` event handler, will be triggered when a new 
record  is added to the `Contacts` journal.

```javascript
  function on_after_append(item) {
    item.date.value = new Date();
    item.status.value = 1;
  }
```

and will set:

* `Date` field to the current date and 
* `Status` field to the `Contact` lookup value.

Let's write the `on_field_get_text` event handler

```javascript
  function on_field_get_text(field) {
    if (field.field_name === 'customer' && field.value) {
      return field.owner.firstname.lookup_value + ' ' + field.lookup_value;
    }
  }
```

that will display in the `Customer` field as the customer's `first` and `last name`.

##### View and edit form set up

Now let's change the display order of the fields when viewing the journal, 
and when editing. We can do this in the [appbuilder page](http://localhost:8080/builder.html) by clicking on the `View form` and `Edit form` buttons, respectively.

Let's change the default code. We delete the dynamic menu, created in the `on_page_loaded` event handler and instead, we will open to view form of the `Contacts` journal.

```javascript
  task.contacts.view($('#content'));
```

For journals the default code allows to use `filters`. 

Let's create filters for journal fields.

To demonstrate the `filters` we'll add another `customer`. We add a new `customer`:

    Kathy
    Chase
    111-22-3333

##### Demonstrate working with html templates

By default, `html template` with class `default-view` is used to creating a `view form` for all `journals` and `catalogs`, for which their own view form template is not defined.

Let's create a view form template for `Contacts` journal, by copying `default-view` template. We define its class as `contacts-view`. And now we add `Customers` button.

```html
  <div class="contacts-view">
    <div class="form-body">
      <div class="view-table"></div>
        <div class="view-detail"></div>
      </div>
    <div class="form-footer">
      <button id="delete-btn" class="btn expanded-btn pull-left" type="button">
        <i class="icon-trash"></i> Delete<small class="muted">&nbsp;[Ctrl+Del]</small>
      </button>
      <div id="report-btn" class="btn-group dropup">
        <a class="btn expanded-btn dropdown-toggle" data-toggle="dropdown" href="#">
        <i class="icon-print"></i> Reports
          <span class="caret"></span>
        </a>
        <ul class="dropdown-menu bottom-up">
        </ul>
      </div>
      <button id="customers-btn" class="btn expanded-btn" type="button">
        Customers
      </button>
      <button id="edit-btn" class="btn expanded-btn" type="button">
        <i class="icon-edit"></i> Edit
      </button>
      <button id="new-btn" class="btn expanded-btn" type="button">
        <i class="icon-plus"></i> New <small class="muted">&nbsp;[Ctrl+Ins]</small>
      </button>
    </div>
  </div>
```

Let's create a event handler for `the on_view_form_created` event.

```javascript
  function on_view_form_created(item) {
      item.view_form.find('#customers-btn').click(function() {
        task.customers.view();
      });
  }
```

So that click on the `Customers` button will open view form for `Customers` item.

We can define the display order of `Contact` journal records and create corresponding index for its database table.

Now we'll open the `project.css` file and change a bit the appearance of the 
application using css.

```css
  .dbtable {
    border: 2px solid blue;
    border-radius: 6px;
  }
  
  .dbtable td.customers {
    color: black;
    font-weight: bold;
  }
```
[< Overview](overview.md) | [Table of content](index.md) | [Task tree >](task_tree.md)