
# Virtual tables

## Populate virtual tables with data

### Populate with `on_after_open` event handler

The usual way to populate a virtual table is to use an event handler `on_after_open`. The `view` method opens items, and for virtual tables it opens them empty.

For example, the following code sets the item attribute `paginate` to `false` and then populates the item with records it retrieves from the server using the `get_records` function.

- **Client-side code**

  ```js
  function on_view_form_created(item) {
      item.paginate = false;
  }
  
  function on_after_open(item) {
      item.server('get_records', function(records) {
          records.forEach(function(rec) {
              item.append();
              item.product.value = rec.product;
              item.price.value = rec.price;
              item.post();
          });
      });
  }
  ```

- **Server-side code**

  ```py
  def get_records(item):
      return [
          {
              'product': 'Product1',
              'price': 10
          },
          {
              'product': 'Product2',
              'price': 20
          }
      ]
  ```

### Populate with `Jam.py` functions

In the following examples, we have Python code ( `show_data` ) that reads the Rest API and returns a list of values ​​( `data` ). This list will be displayed in the visualization tables.

Create a virtual table, including the fields it will display.

In the `on_view_form_created` client handler, include the following lines to disable `pagination` and prevent the record buttons from being displayed:

```js
function on_view_form_created(item) {
    item.paginate = false;
    item.view_form.find("#edit-btn").hide();
    item.view_form.find("#delete-btn").hide();
    item.view_form.find("#new-btn").hide();
}
```

In the `on_after_open` client handler, is called the Python routine `show_data` which returns a list ( `data` ) of records, and then include the records in the view form:

```js
function on_after_open(item) {
    item.server('show_data', function(data, error) {
        if (error) {
            item.alert_error(error);
        }
        else {
            data.forEach(function(rec) {
                item.append();
                item.id.value = rec.id;
                item.emp.value = rec.emp;
                item.data.value = rec.data_diaria;
                item.valor.value = rec.vl_diaria;
                item.uhs.value = rec.uhs;
                item.post();
            });
        }
    });
}
```

### Populate with DataTables Javascript library

<https://datatables.net>

Include the JS and CSS DataTables files in `index.html`:

```html
...
<script src="jam/js/jam.js"></script>

<!-- DataTables Lib Include -->
<link rel="stylesheet" type="text/css" href="/css/jquery.dataTables.css">
<script type="text/javascript" charset="utf8" src="/js/jquery.dataTables.js"></script>

Create a DIV “datatables-view” in index.htmlthe file:

<div class="datatables-view">
    <div class="row-fluid">
        <br>
        <table id="table_id" class="display"></table>
    </div>
</div>
```

Create an item with a virtual table called “datatables” (fields are not required).

In the client `on_view_form_created` event, is called the Python routine `show_data` which returns a list ( `data` ) with records and finally displays them in a DataTable object.

```js
function on_view_form_created(item) {
    item.server('show_data', [optview], function(data, error) {
        if (error) {
            item.alert_error(error);
        }
        else {
            var table=$('#table_id').DataTable( {
                language: {
                    url: '/js/datatables-pt-br.json',
                    decimal: '.',
                    thousands: ','
                },
                data: data,
                columns: [
                    {data: 'id', title: 'Id'},
                    {data: 'emp', title: 'Emp'},
                    {data: 'data_diaria', title: 'Data'},
                    {data: 'vl_diaria', title: 'Valor'},
                    {data: 'uhs', title: 'UHs'}
                ]
            });
        }
    });
}
```

### Virtual table and dynamic fields

The `apply` method doesn't work for virtual tables, you could save changes to the table on the server using some event handler, for example, `on_apply` and `server` method to send the data to the server and save it.

As for creation dynamic fields we have `dyn_fields` method for the `Item` class in the `jam.js` module.

You can find it by searching for the line:

```js
dyn_fields: function(fields) {
```

It needs to add dynamic fields to the item based on the parameter fields.

The parameter `fields` is a list of field definitions. They must be the same as those passed in `server` for item to create the fields on the client.

> [!Note]
>
> I tried the dyn_fields method for creating dynamic fields and it turned out to be outdated.
>
> I fixed it and published the changes in version 5.4.131.

It can be used in the following way:

```js
function on_view_form_created(item) {
    let d_fields = [
        {
            field_name: 'string_field',
            field_caption: 'String field',
            data_type: task.consts.TEXT,
            field_size: 100,
            required: true
        },
        {
            field_name: 'integer_field',
            field_caption: 'Integer field',
            data_type: task.consts.INTEGER
        },
        {
            field_name: 'lookup_field',
            field_caption: 'Lookup field',
            data_type: task.consts.INTEGER,
            lookup_item: task.customers,
            lookup_field: 'lastname'
        },
    ];
    item.dyn_fields(d_fields);

    item.view_options.fields = ['string_field', 'integer_field', 'lookup_field'];
    item.edit_options.fields = ['string_field', 'integer_field', 'lookup_field'];    

    item.open();
}
```

Dynamic fields are part of a virtual table.

### Getting data from tables into a virtual table

- First, you need to define everything for the VT table.

- Next, you need to create the server queries for the return, which is defined in the VT.

**Server code**:

```py
def get_rows(item):
    rows = None
    sql = """SELECT ID, User_ID, YearID, Year FROM INTEGRA_GRUSES"""
    #print(sql)
    rows = item.task.execute_select(sql)
    return rows

def get_uses(item):
    res = []
    rows = []

    rows = get_rows(item)
   
    for r in rows:
        res.append(
            {
                'id': r[0],
                'user_id': r[1],
                'yearid': r[2],
                'year': r[3]
            }
        )
       
    print(res)
    return res
```

**Client code**:

```js
function on_view_form_created(item) {
    item.paginate = false;
    item.table_options.new = false;
    item.view_form.find("#edit-btn").hide();
    item.view_form.find("#delete-btn").hide();
    item.view_form.find("#new-btn").hide();
    item.open(); // mislim da ovo ne treba
}

function on_after_open(item) {
    item.server('get_uses', function(records) {
        item.disable_controls();
        try {
            records.forEach(function(rec) {
                item.append();
                item.user_id.value = rec.user_id;
                item.yearid.value = rec.yearid;
                item.year.value = rec.year;
                item.post();
            });
        }
        finally {
            item.enable_controls();
        }
    });
}
```

### Getting data from views into a virtual table

For SQLite3, find the file: "jam/db/sqlite.py" code block with:

```py
def get_table_names(connection):
    cursor = connection.cursor()
    cursor.execute("SELECT * FROM sqlite_master WHERE type='table'")
```
  
and add only add `OR type='view'`:
  
```py
def get_table_names(connection):
    cursor = connection.cursor()
    cursor.execute(
        "SELECT * FROM sqlite_master WHERE type='table' OR type='view'"
    )
```

You will be able to import the view. Of course, this is not supported, so your export/import may not work...

So instead of using VT, you can now use DB view, which is almost instantaneous.

### Manual DB mode

"Manual Db mode" guarantees that a table created in the Jam Builder interface will not be created in the database.

When enabled, it allows importing tables or views in these examples above. Importing only creates information in the Jam.py Builder interface, since the table already exists in the database.

So, the way to access a database view in any database is to create a table of the same name in the Builder, with the required fields, while "Manual DB mode" is set.

### Huge performance boost with virtual tables or views

Virtual Tables (VT) have been pretty much under the radar because they were only used rarely. In the demo, VT is used to send emails to customers, which doesn't really show the full potential of VT.

VT can be used as a DB view. Let's say the DB view is just SQL SELECT statement.

So, we can create a server module with whatever SQL we need and pass it to the VT client module to display the table. However, this is exactly where the performance problem lies.

### View does not scroll

I am using this approach to populate VT. It works as expected. However, I noticed that the resulting preview form is not scrollable.

Here is my client module:

```py
function on_after_open(item) {
    item.disable_controls();
    let order_rows = item.server('get_order_rows');
    order_rows.forEach(function(v){
        item.append();
        item.id.value = v.id;
        item.master_rec_id.value = v.master_rec_id;
        item.sku.value = v.sku;
        item.title.value = v.title;
        item.colours.value = v.colours;
        item.post();
    });
    item.first();
    item.enable_controls();
}
```

And here is my server module:

```py
def get_order_rows(item):
    rows = []
    recs = item.task.execute_select('''
        SELECT a.id, a.master_rec_id, b.sku, b.title, a.colours
        FROM order_ordertable a
        INNER JOIN order_products b
            ON a.sku = b.id
        INNER JOIN order_orders c
            ON a.master_rec_id = c.id
                AND c.completed = false
        WHERE a.deleted = 0 and a.start_date IS NULL and a.machine IS NULL;
    ''')
    for v in recs:
        rows.append({
            'id': v[0],
            'master_rec_id': v[1],
            'sku': v[2],
            'title': v[3],
            'colours': v[4]
        })
    return rows
```

After checking your attached samples, I was able to trigger scrolling by adding this part to my client module:

```py
function on_view_form_created(item) {
    item.paginate = false;
}
```

AY never advised scrolling, I don't think it's possible with VT by default. Unless the SQL is customized for some pagination and button actions.

For your example, I would look at Pivot.js.

#### Related field in virtual table

I have a virtual table with a related field (category). This is how I populate the table:

```js
function  plant_next(item) {
    var selected = task.categories.selections;
    if (!selected.length) {
        selected.add(task.categories.id.value);
    }

    item.server('plant_suggestions', [selected],  function(result, err) {
        if (err) {
            item.alert('Failed to send the mail: ' + err);
        }
        else {
            // Load suggestions into a temporary dataset for the modal
            var modal = task.newplanting;
            modal.open({ open_empty: true }, function() {
                modal.disable_controls(); // avoid event firing during fill

                for (var i = 0; i < result.length; i++) {
                    modal.append();
                    modal.id.value = i;
                    modal.category.value = result[i].plant;
                    modal.plant.value = result[i].plant;
                    modal.name.value = result[i].name;
                    modal.rank.value = result[i].rank;
                    modal.post();
                }
                modal.enable_controls();
                modal.edit_options.multiselect = true; // allow selecting multiple plants
            });
            modal.view();
        }
    });
}
```

For some reason, the "Category" field is not displaying the category name. However, when I edit the category field, it is displayed?

If you need to immediately open a form with a new record, on the initial pages of the "modal" field, you need to write and "modal.category.lookup_value".
