Vanilla supports two different methods of dispatch:
  1. by using an array of routes that maps first level URL entry to a file and and a class name
  1. by mapping the URL directly to file and class names in the `controllers/` directory.

In both cases, the found controller class is instantiated and the class itself handles invoking a handler method on itself.

The second form is the suggested, preferred form for controllers specific to your application.  The first form is used to find and invoke controllers in the `vanilla/extensions/` directory, however either can be used.  The first provides a way to name the controller classes differently from the URLs that route to them, however doing so may make things more difficult to maintain.

The default base controller (defined as `base_controller` in `vanilla/base_controller.php`), when constructed, makes the following calls, where `$request` is an array that results from slash-expansion of the `REQUEST_URI` CGI variable:
  1. calls `::_invoke($request)` on itself.
  1. if `$request` is empty, looks for a method named `::index` on itself
  1. if not found, `::_invoke` looks for a method named like `$request[0]` with and without a trailing underscore (a trailing underscore is used to allow methods to be used that conflict with PHP keywords)
  1. if not found, looks for a method named `default_`

If either `::index` or `::default_` is called, the entire `$request` value is provided as a single first argument.  Otherwise, it is expanded out into individual arguments to the method.

A simple controller that merely passes anything else received from the URL to it's view is:
```
class controller_example {
   public function default_() {
      $a = func_get_args();
      $this->view->assign('a', $a);
      $this->viewname = 'example/index.tpl';
   }
}
```
This next controller defines two methods, which can be invoked as
  * `/person/delete/`_idvalue_
  * `/person/create`

```
class controller_person {
   public function delete_($personid=NULL) {
      ...
   }
   public function create() {
      ...
   }
}
```