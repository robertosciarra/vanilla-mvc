This document is meant to demonstrate the following features of vanilla:
  * running the setup script
  * creating a controller
  * creating a view template
  * receiving additional content from the URL (REST-style invocation)
  * processing a simple form

# It's simple #

Two files are all that is necessary.

controllers/example.php:
```
<?php

class controller_example extends base_controller {

    public function list_() {
        # merely an example of passing a variable to a template
        $this->view->assign('audience', 'world');
        $this->viewname = 'example/list.tpl';
    }
}
```

views/example/list.tpl:
```
Hello, {$audience}!
```

The rest of this document explains the details and goes over setting up the vanilla environment.

# Directory setup #

First, create a new directory to store your application.  For the purposes of this demo, we'll assume you have a `public_html` area in your home directory that is accessed as `/~USER/`.
```
$ mkdir -p ~/public_html/vandemo1
```
Now, you'll need to check out the vanilla source code into a subdirectory named `vanilla`:
```
$ cd ~/public_html/vandemo1
$ svn checkout http://vanilla-mvc.googlecode.com/svn/trunk/ vanilla
```

# Running the setup script #

From the directory that **contains** the `vanilla` directory, run the script `vanilla/configure.sh`.  It takes one argument, the web accessible path your application will live in.  The configure script uses this to setup a default `.htaccess` file
```
$ sh ./vanilla/configure.sh
Usage: ./vanilla/configure.sh <document root>
$ sh ./vanilla/configure.sh /~USER/vandemo1
```

If you are serving your application out of the web server's `DocumentRoot`, you will want to specify `/` as the argument.  There is also a way to invoke vanilla from the web server configuration files, by using the `Alias` Apache directive in `httpd.conf`, without using `mod_rewrite`.  This may be faster.

You should now have the following files and directories:
  * .htaccess
  * controllers/
  * index.html
  * media/
  * media/.htaccess
  * models/
  * setup/
  * setup/README
  * setup/global\_conf.php
  * setup/template\_conf.php
  * setup/smarty\_custom.php
  * views/

# Creating a controller #

The default controller, as listed in the `setup/global_conf.php` file, is `example::list`.

Now create a controller file in the `controller/` directory.  vanilla automatically routes requests to controller classes in the `controllers/` directory (however, you can explicitly list routes using another method also), based on the the file and class names.  So to route the `/example` URL, create `controllers/example.php`:
```
<?php

class controller_example extends base_controller {

    public function list_() {
        $this->view->assign('audience', 'world');
        $this->viewname = 'example/list.tpl';
    }
}
```

A couple of things to note:
  * the class must be in a file named `controllers/example.php` and must contain a single class named `controller_example`.  Unless you want to fully customize the controller, it should be a subclass of `base_controller` (which is defined in [vanilla/base\_controller.php](http://code.google.com/p/vanilla-mvc/source/browse/trunk/base_controller.php)).  This class naming scheme is designed to not interfere with model class names.
  * the method we'll be invoking is named `list_`, with the trailing underscore, to avoid conflicts with PHP keywords.  The trailing underscore **is not required** to appear in the URL, nor is it required when the method name does not potentially conflict with a PHP keyword.
  * assigning to `$this->viewname` sets the template file to use for the view.  This is a path relative to the `views/` directory.

Switching to a web browser, if you pull up the URL:
```
http://your-server/~USER/vandemo1/
```
You'll get a 404 error page saying the view isn't found.
```
404
/~USER/vandemo1/ not found (view "example/list.tpl" not found)
```
Our controller is working, but has nothing to display yet.

If you comment out the line that assigns to `$this->viewname`, note that a default template file is looked for based on the controller name (which doesn't exist yet either).

# Creating a view template #

Next we want to create the template file to be used for the view.  vanilla comes pre-packaged with [Smarty Templates](http://www.smarty.net/).  I find it is a good idea to keep templates specific to a controller in a subdirectory that matches the name of the controller, and use individual files that match the method being invoked.  You don't have to use subdirectories, you can put the view files right in the `views/` directory itself, or in differently named subdirectories; however, this may make it more difficult to manage.

Create the file in the `views/` directory that the controller references: `views/example/list.tpl`.  You'll need to make the `views/example/` directory first, of course.   Give it the following contents:
```
Hello, {$audience}!
```
Now if you view the URL
```
http://your-server/~USER/vandemo1/
```
You should see a friendly greeting.

# Receiving additional content from the URL #


Take notice of the content that the following URLs serve:
  * `http://your-server/~USER/vandemo1/example`
  * `http://your-server/~USER/vandemo1/example/list`
  * `http://your-server/~USER/vandemo1/example/list/data`
The first shows an error page:

![http://vanilla-mvc.googlecode.com/svn/wiki/images/vandemo1-404error.png](http://vanilla-mvc.googlecode.com/svn/wiki/images/vandemo1-404error.png)

(The message about not being able to find a controller named errorpage is because vanilla allows you to customize and use specific views when errors are generated).

Without an explicit method specified, vanilla defaults to trying to find a `::index` method on the controller.  We didn't define one in this example.

The last two URLs invoke the `example::list_` method.  These could be used to invoke this controller and method if we had a different value for the `$_SERVER['default_controller']` setting in `setup/global_conf.php`.  For the root page of a site, it may be wise to name the controller and method `homepage::index`, `root::index` or `index::index`, setting a matching value for `$_SERVER['default_controller']` in `setup/global_conf.php`.

Anything after the method name in the URL is received by the method as separate arguments.  The method can access these arguments either with a call to [func\_get\_args()](http://www.php.net/func_get_args) or by listing argument variables in the method definition.  If we change the `::list_` method to:
```
    public function list_($audience=NULL) {
        if (!isset($audience)) $audience = 'world';
        $this->view->assign('audience', $audience);
        $this->viewname = 'example/list.tpl';
    }
```
Then the third part of the URL after `/list/` will be available to the method in the `$audience` argument.  It is wise to define these arguments as optional to avoid PHP warnings about being called without arguments.

The above change to the `::list_` method will allow a new audience string to be passed and printed by the view template.  Try it with different strings:
  * `http://your-server/~USER/vandemo1/example/list/universe`
  * `http://your-server/~USER/vandemo1/example/list/user`

# Handling a simple form #

Let's add a simple, one element form to our view.  Edit `views/example/list.tpl`:
```
Hello, {$audience}!

<form action="{$self}" method="GET">
Enter your name:
<input type="text" name="astring" value="{$audience}" />
<button type="submit">submit</button>
</form>
```

This will display a GET form on the page.  We'll also need to modify the controller to set the `$self` template variable, and to do something with the submitted form data.
```
<?php

class controller_example extends base_controller {

    public function list_($audience=NULL) {
        if (!isset($audience)) $audience = 'world';
        if (isset($_GET['astring'])) $audience = $_GET['astring']; # NEW
        $this->view->assign('audience', $audience);

        $this->view->assign('self', new url($this, 'list')); # NEW

        $this->viewname = 'example/list.tpl';
    }
}
```
We've added the lines marked `# NEW`.  The first one looks for a `$_GET` variable named `astring` and if it was provided, assigns it to `$audience` (overriding the default and what was passed via the `$audience` function argument).

The second one creates a new template variable, `$self` that is a URL object that points back to this controller and method.  The form then uses this as the action attribute.  Of course a different URL could be used, to have the submitted form be processed by a different controller.  A URL object is converted to a string automatically when used in the template.

You may notice that the URL actually put into the action attribute does not explicitly list the `/example/list` portion.  This is because vanilla tries to reduce references to the default controller to a canonical form.

A common vanilla idiom for generating self-referential URLs is:
```
public function m($arg1=NULL, $arg2=NULL) {
    ...
    $request = func_get_args();
    $selfurl = new url($this, 'm', $request);
    ...
}
```

vanilla actually contains complex form building, verification/validation, and processing support, allowing things like multiple forms to exist on the same page, forms to be handled transparently by related models, and easy integration of a form into the template HTML.