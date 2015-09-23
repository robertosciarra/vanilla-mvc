vanilla includes a system that automatically scans your database schema and creates base model objects that map to each table, filling in the relationships.

By following the rules outlined in DatabaseSchemaDesign, vanilla can make quick work of creating the necessary model classes that allow straightforward manipulation to the database without manually writing a lot of database queries.

# Initialization #

In a file in the `setup/` directory, you need to:
  1. connect to your database using `DBI::connect` (see DatabaseAbstractionAPI)
  1. load all models explictly defined in the `models/` directory
  1. create a `SchemaDatabase` object that map the explicitly created models to tables and implicitly create additional models for the remaining tables

The vanilla idiom for this is along these lines (in file `setup/database.setup.php`, for example):
```
$dbh = DBI::connect('dbi:sqlite3:dbname=/some/file');

# include the base model code.  There are three implementations.  dbmodels/3/ is
# the most efficient and reliable version.
require_once "vanilla/dbmodels/3/setup.php";

# all models should/will subclass an abstract base class named Model.
# the dbmodels define an abstract class AbstractModel which can be subclassed
# to provide custom functionality across all models.  If custom functionality 
# is not necessary, we just define an empty abstract class named Model
# that inherits from the common AbstractModel class.
abstract class Model extends AbstractModel { }

lib::load_model_definitions();

$opts = array();
$opts['ignore'] = 'comma,delimited,list,of,tables,to,ignore'; # optional
$opts['ignorere'] = '/^(temp_|junk_)/'; # optional, tables that match this regular expression will be ignored
$tables = $dbh->tables();
Model::init_examine($tables, $opts, $dbh);
Model::init_create_models();
```

# Objects and Classes #

## SchemaDatabase ##

Has three public methods.

  * the constructor.  Takes three arguments, the name of the database, a data handle (as returned by `DBI::connect`) and an array of options.  The allowed options are:
    * `ignore` : a comma delimited list of table names to ignore
    * `ignorere` : a string containing a PCRE regular expression.  if this regular expression matches a table name, that table is ignored
  * `::dump()` - instance method, prints a description of the database schema found
  * `::init($dbh)` - the database only has to be scanned once (unless the structure changes), and scanning can be a relatively expensive operation.  After creating the `SchemaDatabase` object, you can serialize it, cache it, and on later invocations restore it.  After unserializing it, it will need access to the database, so you can use this function to initialize the database handle (as returned from `DBI::connect`) the `SchemaDatabase` should use.

## SchemaTable ##

## SchemaColumn ##

## ModelBase ##

## Model ##

## ModelCollection ##

# Using Models #

## Finding Matching Rows ##

## Updating Rows ##

## Saving Data ##


## Notes about Caching ##

vanilla tries to create only one instance of an object that maps to any given row, so updates to the same row via different references show the proper, same data.  It does this by keeping a reference to each object that is created and if the same row is read again, returning the same object.

# Extending Models #

Due to limitations in PHP's object implementation, extending classes and calling extended classes in a static context is filled with problems.  vanilla attempts to get around this to let you program in a more natural way, with the cost that some additional setup is required for the models you custom create.

Model objects are fully functional, created automatically by `SchemaDatabase` when the database schema is scanned.  However, should you want to extend their functionality, you can instead create the model yourself.  A simple model subclass that offers no additional functionality takes the following form:
```
class myModel extends Model {
    public static $__table__;
```

Only the `$__table__` public static class variable is required (this can not, unfortunately, be defined statically on the `Model` class and be expected to be static in the subclasses).

New methods can then be added to this model that extend, or override, base `Model` functionality.  Most commonly, methods for creating generated fields (see DatabaseSchemaDesign) and new `::find` methods will be added.

```
class company extends Model {
    public static $__table__;

    public function __get_age() {
        return time() - $this->creation_date;
    }

    public function __set_age($ageseconds) {
        $this->creation_date = time() - $ageseconds;
        $this->save();
        $this->_refresh();
    }

    public function find_by_age($maxage) {
        return company::find(array('creation_date'=>cond::gt(time()-$maxage)));
    }
}
```

The above block creates a generated field that can be accessed as `->age`, creates an accessor method for the fake `age` field that sets the natural (real) field named `creation_date`, and provides a new `::find_by_age` method that filters companies on a calculated condition.

vanilla does not currently support dynamic `::find` methods that filter by field names given in the method name (like `::find_by_name`).  You will have to define those methods if you desire that kind of functionality.  However, these are rarely necessary because of the support the default `::find` method has for filtering on specific fields.


# Useful Internal Fields #

  * `$this->_t` - the `SchemaTable` object that this model is based on
  * `$this->_db` - the `SchemaDatabase` object that this model uses

There may be times that you need to access the table name or the database handle (created to connect to the database) directly, such as when doing direct database manipulation or when you want to run a custom `::find` function for your model.

Interesting fields are:
  * `$this->_t->name` and `$this->_t->nameQ` - the actual name of the table this model maps to.  The first is just the raw name, and the second is already quoted (using the database's label quoting method) ready for insertion into queries.
  * `$this->_t->pk` - the name of the primary key column for the table
  * `$this->_db->dbhandle` - the database connection object returned by `DBI::connect` (see DatabaseAbstractionAPI`) and passed to the `SchemaDatabase` constructor.

# Deletion #

Because deletion is a destructive operation, the base `Model` class does not currently provide direct support for deleting rows.  You can add a row deletion method by defining your model directly and adding a delete method like the following:

```
class myModel extends Model {
    public static $__table__;

    public function delete_() {
        $q = sprintf('delete from %s where %s = ?', $this->_t->nameQ, $this->_t->pk);
        $sth = $this->_db->dbhandle->prepare($q);
        $sth->execute($this->id);
    }
}
```

# Optimization #

Numerous problems with PHP's object model (see [Bug #30934 Special keyword 'self' inherited in child classes](http://bugs.php.net/bug.php?id=30934) and [Bug #12622 scope of $this and static functions](http://bugs.php.net/bug.php?id=12622) for two such cases) force some pretty hairy code to be run when the `::find` or `::find_first` methods are called.  This is not normally a problem, but may sit uncomfortably with people who want to avoid a potentially nasty performance hit that exists to overcome the problems with PHP's inheritance implementation.

When `::find` (or `::find_first`) is called, `Model::find` looks up the call stack for the file and line that called it.  It then searches this file (using `preg_match`) for the specific call signature to figure out the model class name that it was called on.  This allows you to program naturally using static method invocation (`company::find(...)`) without requiring an existing instance of the model class to call on to get the proper scope.  See [the `\_find\_static\_invoking\_class` function at the end of the Model.php file](http://code.google.com/p/vanilla-mvc/source/browse/trunk/Model.php) for the exact implementation.

Model classes automatically created by `SchemaDatabase` already have this optimization, but model classes you create do not.  If you insert the block of code from the file [ModelCommonCode.php](http://code.google.com/p/vanilla-mvc/source/browse/trunk/ModelCommonCode.php) at the top of your model class definition, you can avoid the expensive regular expression search that results.  These functions make the calls to the Model implementation directly, because once defined on the model class itself, they have direct access to the static class property `$__table__`.

Unfortunately PHP does not have parse/compile-time code inclusion functionality that works at the class-definition scope, otherwise one could work around this with a C-preprocessor style `#include` at the top of the class definition.