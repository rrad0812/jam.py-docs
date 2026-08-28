
# Pivot tables in Jam app

I want to interduce you pivottable.js plugin. With this plugin, you can create on easy and fast way dinamic pivot table in Jam application.

This is very usefull tool when you need to analize data with combine different parameters.

To use this plugin, you need to add there libs:

```html
<!-- jquery ui for drag and drop-->
<script src="https://cdnjs.cloudflare.com/ajax/libs/jqueryui/1.11.4/jquery-ui.min.js"</script>

<!-- pivottable.js -->
 <script type="text/javascript" src="js/pivot.js"></script>
  <script src="https://cdn.plot.ly/plotly-basic-latest.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/pivottable/2.23.0/plotly_renderers.min.js"></script>

 <!--- table to excel -->
 <script type="text/javascript" src="static/js/tableExport.js"></script>
```

You need to create virtual table and html form template:

```html
<!-- pivot table start -->
    <div class="pivot_table-view">
        <div class="form-body" height="1000">
            <div id="pivot_table"></div>
        </div>
        <button type="button" id="export_pivot" class="btn btn-primary expanded-btn">
            <i class="icon-o-download"></i>&nbsp; Download
        </button>
    </div>
<!-- pivot table end -->
....
```

Further, you need to load setup form "pivottable.js" like this:

```js
function on_view_form_created(item) {
    create_pivot_table(item);
}

function create_pivot_table(item) {
    let pivot = task.pivot_table.copy();
        pivot.view_options.title = 'Pivot - sales analisis';
        pivot.view(task.forms_container);
       
    let pivot_records = task.order_details.copy(),
        records = [];
        pivot_records.open();
       
        pivot_records.each(function(c) {
            records.push({
                "order_id": pivot_records.order_id.value,
                "customer": pivot_records.customer.display_text,
                "city": pivot_records.city.display_text,
                "order_date": pivot_records.order_date.display_text,
                "date_allocated": pivot_records.date_allocated.value,
                "employee": pivot_records.employee.display_text,
                "order_status": pivot_records.order_status.display_text,
                "product_id": pivot_records.product_id.display_text,
                "product_category": pivot_records.product_category.display_text,
                "shipper": pivot_records.shipper.display_text,
                "status_id": pivot_records.status_id.display_text,
                "total_price": pivot_records.total_price.value,
                "quantity": pivot_records.quantity.value,
                "quarter": get_quarter(item, pivot_records.order_date.display_text)
            });
        });
   
        var dateFormat = $.pivotUtilities.derivers.dateFormat;
        var sortAs = $.pivotUtilities.sortAs;
                       
        $("#pivot_table").pivotUI(records,
            {
            renderers: $.extend(
                $.pivotUtilities.renderers,
                $.pivotUtilities.plotly_renderers,
                $.pivotUtilities.gchart_renderers,
                $.pivotUtilities.d3_renderers
            ),
            derivedAttributes: {
                "day": dateFormat("order_date", "%w", false),
                "month": dateFormat("order_date", "%n", false),
                //"month_no": (((month - 1)/3)+1).floor,
                "year": dateFormat("order_date", "%y", false)
            },
            rows: ["day"],
            cols: ["month"],
            sorters: {
                "day": sortAs(["Mon","Tue","Wed", "Thu","Fri","Sat", "Sun"]),
                "month": sortAs(["Jan","Feb","Mar","Apr", "May", "Jun","Jul","Aug","Sep","Oct","Nov","Dec"])
            },
        });
   
    item.view_form.find("#export_pivot").click(function(e) {
        $('#pivot_table').tableExport({
            type: 'excel',
        });
    });
}

function get_quarter(item, order_date) {
    let date = new Date(order_date);

    return "Q" + Math.floor(date.getMonth() / 3 + 1);
}
...
```

You can user charts and heatmaps and also do export table to excel with tableExport plugin.

---

To force the Pivot table to actually display exactly the same look as above, we can expand on this so the user does not need to do anything.

Change this:

```js
rows: ["day"],
cols: ["month"],
sorters: {
```

to:

```js
rows: ["employee"],
cols: [ "year", "quarter","month"],
aggregatorName: "Sum",
vals: ["total_price"],
rendererName: "Heatmap",
```