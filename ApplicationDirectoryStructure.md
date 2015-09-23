# setup/ #

Setup related code.  As part of the initial application setup process, it should contain a `README` with details on the default files.  There are two required files, one optional file, and a series of application specific setup files
  * `setup/global_conf.php` - **required** generated during configuration and has notes about what each setting means.
  * `setup/smarty_custom.php` - **required** but may define just an empty class.  this file provides custom smarty functionality that your application may require.  See VanillaSmartyExtensions for more information.
  * `setup/template_conf.php` - **optional** If this file exists, it will be included at every point when a new smarty object is instanciated.  Any customizations to the smarty object should happen here.
  * `setup/*.setup.php` - **optional** any file that ends in `.setup.php` will be executed, in sorted order, to setup the application's environment.  Things like connecting to the database, initiating the schema scanning, or creating a PHP session should go in these files.

# models/ #

Contains the definitions for all custom models for your application.  Models only need to be defined explicitly if you require functionality beyond what is defined by default during the schema scanning.

# views/ #

Contains the template files that implement the views/presentation for the application.  It is recommended to create a directory in `views/` for each functional unit (often based on the related controller's name when a view is for a full page).  If the _media_ extension is enabled, view-specific CSS and javascript files can be served right from these directories to keep your code easy to maintain and compartmentalized.

Common template elements can also be stored here (perhaps in a subdirectory called `common/`).  The `views/` directory is the the default _smarty template dir_ setting, so smarty functionality that manipulates files will be anchored from this directory.

# controllers/ #

Contains the controller objects for the project.  Each file should be named as the controller short name and should contain a class definition for the controller.  All controller class names must begin with `controller_` in order to be automatically dispatched (see RoutingAndDispatch)

The `controllers/` directory can also contain files related to controllers, like a custom parent class that your controllers will inherit from if necessary.

# media/ #

Store all media and related assets in this directory.  When using Apache's `mod_rewrite`, this directory should have `Rewrite off` (in an `.htaccess` file) so the web server can serve the files in here without invoking PHP.  This should be used for graphics, common style sheet and javascript assets, or other static content.

See SetupWithApacheModRewrite.

# vanilla/ #

Contains the vanilla code.