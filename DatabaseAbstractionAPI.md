# Goals and History #

Having existed in some form since and used on PHP4, this dbi module was originally a way to unify PHP's traditionally inconsistent database APIs, ease a transition for developers coming from a perl background, and provide additional levels of security through (emulating) prepared statements.  PDO support in PHP5 has attempts to many of PHP's database shortcomings.  Where it can or needs to use PDO on the backend (because of database API support) it does, and ends up being a consistent wrapper around the different database abstractions that PHP and your environment supports.  It also provides simple statistics gathering and makes it a little easier to debug database interactions en mass (PHP's lack of open classes make it difficult to effectively extend PDO at run time

`vanilla/dbi.php` is an attempt to bring [perl-like DBI](http://search.cpan.org/~timb/DBI-1.601/) support to PHP.  It is functional for many of the most common and simple DBI operations.  It currently supports mysql and sqlite.

Nothing in vanilla strictly requires the use of this dbi class to interface with a database, however the automatic schema scanning does currently rely on it.  This may be changed in a future version and unify everything under the more PHP-ish PDO library of functions.

# Connecting #

The DBI::connect method can take its argument in either array or string form and returns a database handle.
```
$dbh = DBI::connect('mysql', array('host'=>'127.0.0.1',
                                   'user'=>'user',
                               'password'=>'password',
                               'database'=>'database',
                             'persistent'=>true));
```
or
```
$dbh = DBI::connect('dbi:mysql:host=127.0.0.1;user=user;password=password;database=database;persistent=1');
```

The first argument, or the part of the string after `dbi:`, designates the database driver name.  The following database types are currently supported:
  * `mysql` - uses the older, `mysql_` family of PHP functions)
  * `sqlite3` - uses the PDO interface to sqlite3
  * `sqlite` - uses the sqlite version 2 `sqlite_` family of functions.
  * `sqlite2` - an alias for `sqlite`

The rest of the arguments are dependent on the database.  For `mysql`, the options are
  * host
  * user
  * password
  * database
  * persistent
And for sqlite related DBD, the only option is:
  * dbname - the path to the database file

# Performing Queries #

`dbi` provides a "prepared query" interface which has a slightly different syntax than other methods.

## Simple Positional Syntax ##

```
$q = "select col from table where x = ? and y = ?";
$sth = $dbh->prepare($q);

$sth->execute($x, $y);

$l = array($x, $y);
$sth->execute_array($l);
```


## Simple Named Syntax ##

Note that when using the named syntax, `::execute` takes a single array argument listing the keys and values.  This is due to a limitation in PHP being able to use anything resembling named arguments (like passing in hash/string-indexed arguments without wrapping it another array and passing it as the first argument).  The named variable syntax is incompatible with `::execute_array`.  In most cases, this won't matter much with vanilla, as models will be handling a lot of this functionality.

```
$sth = $dbh->prepare("select col from table where x = ?:xval and y = ?:yval");
$sth->execute(array('xval'=>$x, 'yval'=>$y));
```

## Named syntax with array join ##

This is useful if you have an array of values to test against.  You don't need to quote and join the elements yourself, you can have dbi do it for you.

```
$sth = $dbh->prepare("select col from table where x in (?:xval:join) and y = ?:yval");

$x = array('one', 'two', 3);
$sth->execute(array('xval:join'=>$x, 'yval'=>10));
# will build and execute the query
# select col from table where x in ('one', 'two', 3) and y = 10
```


# Functions on database handles #

  * `->quote_label($x)` - returns the passed string quoted for the database when using the string as a label (for table, column, etc names)
  * `->quote($value)` - returns the value converted to a quote value ready to be inserted into a query.  If passed an array, it will quote each element of the array and return the result.
  * `->prepare($stmt)` - returns a statement handle that can be used to manipulate the SQL statement in `$stmt`
  * `->do_($stmt, $opts, $arg1, ...)` - executes `$stmt` with the given options `$opts` using the remaining arguments as positional substitutions.  Currently `$opts` is ignored, but should be an array.  Named with the trailing underscore to avoid conflicts with the `do` keyword.
  * `->stats()` - returns a string listing the number of queries executed, the number of rows fetched, and the time spent executing queries.  informational purposes only
  * `->tables()` - returns a list of tables in the database
  * `->table_info($table)` - returns information about the table named `$t`.  _interface unstable_
  * `->column_info($t, $c)` - returns information about column `$c` in table `$t` _interface unstable_

# Functions on statement handles #

  * `->bind_param($i, $v, $type=NULL)` - bind the value `$v` to the `$i`th positional parameter. _interface unstable_
  * `->execute(...)` - execute the statement, using the arguments passed as the parameters for the statement
  * `->execute_array($a, $b, $c,...)` - treat each of the passed arguments as their own array of arguments to `::execute`
  * `->num_rows()` - returns the number of rows for the last query
  * `->affected_rows()` - returns the number of rows affected by the last query, may only make sense in the context of data changing statements, like `delete` or `update`
  * `->insert_id()` - returns the last autoincrement value generated
  * `->finish()` - destroys the statement handle and all related data.  called automatically when the statement object is destroyed
  * `->fetchrow_array()` - returns the next row indexed by numerical offsets
  * `->fetchrow_arrayref()` - alias for `fetchrow_array`
  * `->fetchrow_hash()` - returns the next row indexed by column name
  * `->fetchrow_hashref()` - alias for `fetchrow_hash`
  * `->fetchrow_object($bc=NULL)` - returns the next row as an object with the properties named as the columns in the result.  If `$bc` is an object instance, will set properties on that object.  If `$bc` is a string and a class named as that string exists, will create a new instance of that class and set properties on it.  Otherwise, will return an instance of `stdClass` (the stock PHP "empty class", "root class").
  * `->_stmt()` - useful for debugging, returns the full statement last executed
  * `->execution_time()` - returns the execution time of the query in seconds.