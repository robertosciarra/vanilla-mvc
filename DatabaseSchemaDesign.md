With a little forethought in the design of your database, vanilla's automatic schema scanning will make quick work of maintaining your models' mapping to the database so you can concentrate on the customized parts of the models.  The ideal situation is a new database.  Support for legacy database, already defined databases that do not conform to the limits laid out here may be added in the future.

There are four rules for the schema that should be followed for the best results:
  * tables should be named in the singular, like `product`, `customer`, and `box`; avoid pluralized table names like `products`, `customers`, or `boxes`.
  * primary keys should be named merely `id`, and can be either integer auto-increment or strings.  primary keys can also be foreign keys if they are named as foreign keys are described next.
  * foreign keys should be named to end with the foreign table name followed by `_id`.  If the `order` table references a customer, then it should have a column named `customer_id`
  * primary keys should be on a single column to be able to uniquely identify them (this limitation may be lifted in the future)

The system automatically figures out various one-to-one, one-to-many and many-to-many mappings based on the naming and provides easy on-demand-loading access to related rows/objects.  This allows the controller to provide the minimal related objects to the task to the view without limiting exactly how or what the view is able to display.  Future versions may support using schema metadata (like explicit foreign key references) to reverse engineer the relationships.

There are three kinds of fields available on a model:
  1. _natural fields_ - those fields listed in the database
  1. _virtual fields_ - fields that are created automatically based on relational mappings.
  1. _generated fields_ - fields that are created by property access overloading methods on the model

## Natural Fields ##

Natural fields are the raw data that is stored in the database.  Natural fields will always be the base non-object types that the database and PHP support.

Natural fields are the only ones that are updated when a model object is saved to the database.

## Virtual Fields ##

Virtual fields are the result of automatic schema scanning, and will usually be either other model objects or collections of model objects.

After saving changes to the database, the virtual fields will need to be updated with a call to the `::_refresh` method on the object (however, this is not always necessary because most updating/saving operations will not need long-lived objects; see FormHandlingUpdatingModels).

If a table references another table with a field name ending in `TABLE_id`, a virtual field will automatically be created named `TABLE` referencing the row in the foreign table with the given primary key.  The value of this field will be a single object of the class of the referenced model.  These fields could also take the form `EXTRA_TABLE_id`, where `EXTRA` might be used to better label the column among the others in the table.  For example, if you had two columns named `parent_node_id` and `child_node_id`, then the virtual columns created for these two would be `parent_node` and `child_node` and would both reference rows (node models) from the `node` table.

If a join table is found, virtual fields are created for a many-to-many mapping.  Virtual fields referencing both the rows in the join table _and_ the rows in the related table are created.  For example, for the following three tables:
```
create table product ( 
   id int(11) primary key
);

create table product_image (
   id int(11) primary key,
   product_id int(11),
   image_id int(11)
);

create table image (
   id int(11) primary key
);
```
the `product` model will have the virtual fields:
  * `image` - access the image model objects directly, a `ModelCollection`
  * `product_image` - manipulate the mapping model objects directly, a `ModelCollection`
the `image` model will have the virtual fields:
  * `product` - access the product model objects directly, a `ModelCollection`
  * `product_image` - manipulate the mapping model objects directly, a `ModelCollection`
the `product_image` model will have the virtual fields:
  * `product` - a single product object (since `product_image` -> `product` is a one-to-one mapping)
  * `image` - a single image object (since `product_image` -> `image` is a one-to-one mapping)
Any one-to-many mappings are `ModelCollections`, which are iteratable just like regular arrays, and are indexed by the primary key of the foreign relation.

If a primary key is found that is named `TABLE_id`, then a one-to-one mapping virtual column is made on the model `TABLE` that references the referring row.

Virtual fields are not saved to the database automatically when their parent object is saved, they must be saved explicitly.


## Generated Fields ##

Generated fields are the result of property access overloading on the model class.  As far as the code is concerned, these look like normal properties, albeit read-only.

These are defined through a `__get_FIELD` method on the model, which takes no argument and returns the value of the given field.

The following sample code, if on the model, would create a property named `name_and_city`, accessed as `$obj->name_and_city`:
```
public function __get_name_and_city() {
   return $this->name." of ".$this->city;
}
```

Generated fields override the values of natural fields and virtual fields, allowing all property values to be customized at run time.

Unless you define an appropriate `__set_FIELD` method, generated fields are read-only and should be used to easily provide more complex functionality to the views without needing complex logic in the template.  Any overloaded property access would need to perform whatever is necessary to update natural fields.

# Example #


# Other Notes #

It is recommended you store dates in the database as integers (using functions like `unix_timestamp()` in mysql).  This provides ease of use when setting values (the result of the PHP function `time()` can be used without conversion to a database specific datetime formatted string) and this leaves the printing of dates and times up to the template (where it belongs, as part of the view/presentation layer) using Smarty modifiers like `date_format` or the PHP function `strftime`.


The built in data inspector extension can take "hints" to make understanding models and database schema easier.
  * naming the integer datetime columns to start with `date_` or end with `_at` or `_date` will tell the data inspector to format the data as a date.