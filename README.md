# Dispatch

Dispatch is another PHP micro-framework. It's very small and very straightforward
to use. No classes, no namespaces.

## Requirements
Officially, Dispatch requires **PHP 5.4**. Unofficially though, it can run with
PHP 5.2 and up.

No strict version check is being done by Dispatch, other than the requirements
imposed by `composer.json`. Functions specific to 5.4 are only used when available.

To run the tests though, you'll need PHP 5.4 as it makes use of PHP's built-in
web server.

## Code
Get the code on GitHub: <http://github.com/noodlehaus/dispatch>.

## Installation
To install using `composer`, have the following lines in your `composer.json` file.

```javascript
{
  "require": {
    "php": ">= 5.4.0",
    ...
    "dispatch/dispatch": "dev-master",
  }
}
```

Then do a `composer install` or `composer update` to install the package.

If you don't use `composer`, just download and include `dispatch.php` directly in
your application.

Note that Dispatch functions are all loaded into the global namespace.

If you have access to `mod_rewrite`, make sure to redirect all your PHP requests to
your app.

```
<IfModule mod_rewrite.c>
  RewriteEngine on
  # in this case, our app bootstrap file is index.php
  RewriteRule !\.(js|html|ico|gif|jpg|png|css)$ index.php
</IfModule>
```

## Configuration Variables

Some settings are needed by Dispatch, and they can be set via `config()`.

```php
<?php
// REQUIRED, base path for your views
config('dispatch.views', '../views');

// OPTIONAL, layout file to use (defaults to 'layout')
config('dispatch.layout', 'layout');

// OPTIONAL, cookie for flash messages (defaults to '_F')
config('dispatch.flash_cookie', '_F');

// OPTIONAL, specify your app's full URL
config('dispatch.url', 'http://somedomain.com/someapp/path');

// OPTIONAL, routing file to be taken off of the request URI
config('dispatch.router', 'index.php');

// you can also just pass a hash of settings in one call
config([
  'dispatch.views' => '../views',
  'dispatch.layout' => 'layout',
  'dispatch.flash_cookie' => '_F',
  'dispatch.url' => 'http://somedomain.com/somapp/path',
  'dispatch.router' => 'index.php'
]);
?>
```

## Routing
Application routes are created via calls to `on($method, $path, $callback)`. Supported methods are
`GET`, `POST`, `PUT`, `DELETE`, `HEAD` and `PATCH`. The `$method` parameter can be a single method, an
array of methods, or `*` (for all methods).

```php
<?php
// get route for /
on('GET', '/', function () {
  echo "hello, world!\n";
});

// get route for /index
on('GET', '/index', function () {
  echo "hello, world!\n";
});

// support the route for multiple methods
on(['GET', 'POST'], '/greet', function () {
  echo "hello, world!\n";
});

// handle all supported methods for a route (get, post, put, delete, head, patch)
on('*', '/multi', function () {
  echo "it works!\n";
});
?>
```

## Grouped Routes (Resources)
When working on APIs, you tend to create routes that resemble resources. You can do this by
including the resource name in your route, or by scoping your route creation with a
`prefix($path, $routine)` call, where `$path` contains the name of the resource, and
`$routine` is a callable that contains routing calls.

```php
<?php
// let's create a users resource
prefix('users', function () {

  on('GET', '/index', function () {
    // show list of users
  });

  on('GET', '/:username/show', function () {
    // show user details
  });
});

// this is a route created outside of users
on('GET', '/about', function () {
  // about page
});
?>
```

From the code sample, routes `/users/index` and `/users/:username/show` will be made. Then
outside of the `users` resource, a `/about` route is also made.

## Site Path and URL Rewriting
If your app resides in a subfolder, include this path in your `dispatch.url` setting, so Dispatch
knows which parts of the `REQUEST_URI` need to be removed. This URL or your app path can then be
accessed via `site($path_only = false)`.

```php
<?php
// our app lives in /mysite
config('dispatch.url', 'http://somehost.com/mysite');

on('GET', '/users', function () {
  echo "listing users...";
});

// requested URI = http://somehost.com/mysite/users
// response = "listing users..."

// get your full URL
$url = site();

// get just /mysite
$path = site(true);
?>
```

If you don't have access to URL rewrites, and are using a file router (ie. /index.php/controller/action),
you need to specify this via `dispatch.router`. The string you set this to gets stripped off of the
`REQUEST_URI` before Dispatch routes the request.

```php
<?php
// strip index.php from all route requests
config('dispatch.router', 'index.php');

on('GET', '/users', function () {
  echo "listing users...";
});

// requested URI = /index.php/users
// response = "listing users..."
?>
```

## HTTP Redirects
Redirects are done via `redirect($path, $code = 302, $condition = true)`. The third
parameter, `$condition`, is useful if you want your redirects to happen depending on
the result of an expression.

```php
<?php
// basic redirect
redirect('/index');

// with a custom code
redirect('/new-url', 301);

// redirect if authenticated() is false
redirect('/denied', 302, !authenticated());
?>
```

## Request Method Overrides
Method overrides set either via the `_method` form field or the
'X-Http-Method-Override' header are supported. If either of these two are
present in the request, their values are respected. Below is the particular
code that describes how this is handled.

```php
<?php
  // check for method override
  if (isset($_SERVER['HTTP_X_HTTP_METHOD_OVERRIDE']))
    $method = $_SERVER['HTTP_X_HTTP_METHOD_OVERRIDE'];
  else if (params('_method'))
    $method = params('_method');
  else
    $method = $_SERVER['REQUEST_METHOD'];
?>
```

## Request Headers
Dispatch provides a  convenience function for accessing request headers via
`request_headers($name = null)`. When called without parameters, it returns
an associative array of all the request headers. If given a `$name` parameter,
returns the value for that header or null if it doesn't exist.

```php
<?php
// get all of it
$headers = request_headers();

// or just one
$accept_encoding = request_headers('accept-encoding');
?>
```

## Request Body in PUTs or JSON POSTs
In cases where you're handling PUT requests or JSON posts and you need access
to the raw http request body contents, you can use `request_body()`.

For content of type `application/json` and `application/x-www-form-urlencoded`,
the content are automatically parsed and returned as arrays.

```php
<?php
on('PUT', '/users/:id', function ($id) {
  $data = request_body();
});
?>
```

## Route Symbol Bindings and Filters
Named parameters or symbols in routes can have callbacks associated with it.
With these callbacks, you can perform routines like data loading, or value
filtering/transforms. Functions that let you do this are `bind($symbol, $callback = null)` and `filter($symbol, $callback = null)`.

Callbacks mapped using `bind()` transform the final value that gets passed
to your route handler.

```php
<?php
// bind() callbacks need to return a value
bind('hashable', function ($hashable) {
  return md5($hashable);
});

// the value received for $hash is already
// transformed by our bind() callback
on('GET', '/md5/:hashable', function ($hash) {
  echo $hash . '-' . params('hashable');
});
?>
```

For callbacks mapped using `filter()`, they are not required to have
a return value, and if they do, the return value does not transform
the parameter value passed to the route handler.

```php
<?php
// preload blog entry whenever a matching route has :blog_id in it
filter('blog_id', function ($blog_id) {
	$blog = Blog::findOne($blog_id);
	// scope() lets you store stuff for later use (NOT a cache)
	scope('blog', $blog);
});

// here, we have :blog_id in the route, so our preloader gets run
on('GET', '/blogs/:blog_id', function ($blog_id) {
  // $blog_id is still the original, but
	// we can pick up what we stored with scope()
	$blog = scope('blog');
	render('blogs/show', array('blog' => $blog);
});
?>
```

##

## Before and After Callbacks
To setup routines to be run before and after a request, use `before($callable)` and `after($callable)`
respectively. The callback routines will receive two arguments - the `REQUEST_METHOD`, and the `REQUEST_URI`.

```php
<?php
// setup a function to be called before each request
before(function ($method, $path) {
  // setup stuff
});

// setup a function to be called after each request
after(function ($method, $path) {
  // clean up stuff
});
?>
```

## HTTP Errors and Error Handling
You can create custom HTTP error handlers and trigger them as well via calls to
`error($code, $callback_or_string = null)`.

```php
<?php
// create a 404 handler
error(404, function () {
  echo "Oops!\n";
});

// trigger the error
error(404);

// trigger another error, with a custom message
error(500, "Something broke!");
?>
```

## Layout, Views and Partials
For Dispatch to work with layouts, views and partials, you need three settings:

* `dispatch.views` - where to find all the views
* `dispatch.layout` - the layout file to use (without .html.php) from the views path
* your layout, views and partials should end with `.html.php`

The layout file you specify needs to contain a call to `content()`. This will plug in
the contents of your view into your layout file.

```php
<!DOCTYPE html>
<html>
<head><title>My Layout File</title></head>
<body>

<!-- this call will plug in the contents of your view -->
<?= content() ?>

</body>
</html>
```

With these set, you can then call `render()` in the following ways:

```php
<?php
// render a view with locals using the configured layout file
render('index', ['name' => 'joe']);

// render a view using a different layout file (mobile-layout.html.php)
render('index', ['name' => 'bob'], 'mobile-layout');

// render a view without using a layout file
render('index', ['name' => 'bob'], false);
?>
```

If you just want to fetch the results of a template using some locales, without
echoing the results, you can make a call to `template()` instead.

```php
<?php
$page = template('index', ['name' => 'bob']);
?>
```

For partials, the files are expected to begin with the `_` character, and can be
loaded via `partial($path, $locals = [])`.

```php
<?php
// underscore on the filename is added automatically by partial()
$html = partial('users/profile_page', array('data' => $data));
?>
```

If you don't really need to do anything but to render a template for a route,
and maybe use some local variables and a layout, you can use
`inline($file, $locals = array(), $layout = null)` instead. `inline()` creates
a route handler for you that does nothing but render the view specified using
the scope variables, and layout file, if any, when the route is invoked.

```php
<?php
// just render the about-us template for the route
on('GET', '/about-us', inline('about-us'));

// render a template with some locals
on('GET', '/contact-us', inline('contact-us', array('email' => 'support@blah.com')));

// render using a different template
on('GET', '/faq', inline('faq', array(), 'faq-layout'));
?>
```

## JSON and JSONP Responses
JSON and JSONP responses are done via `json_out($obj, $func = null)`.

```php
<?php
// object to dump
$obj = ['name' => 'noodlehaus', 'age' => 34];

// non-cacheable json response
json_out($obj);

...

// jsonp callback name
$fxn = 'parseResponse';

// non-cacheable jsonp response
json_out($obj, $fxn);
?>
```

## No-Cache
If you want to output non-cacheable content, you can do this by calling `nocache()` before
outputting any content.

```php
<?php
// output nocache headers
nocache();

echo "comes fresh, everytime!";
?>
```

## Cookies and Sessions
Get and set cookie values via `cookie($name, $value = null, $expire = 0, $path = '/')`.

```php
<?php
// set a cookie
cookie('user_id', 'user-12345');

// get a cookie
$user_id = cookie('user_id');
?>
```

For getting and setting session values, use `session($name, $value = null)`. Calls to `session()` will
fail and raise an error if you have sessions disabled in your `php.ini`.

If sessions are enabled, `session_start()` is called automatically for you.

```php
<?php
// set a session value
session('authenticated', false);

// get a session value
$authenticated = session('authenticated');

// remove a session variable
session('authenticated', null);
?>
```

## Cross-Request Messages (Flash)
Cross-request messages, or flash messages, can be done via `flash($name, $message = null, $now = false)`.

```php
<?php
// set an error message to show after a redirect
flash('error', 'You did something wrong!');
redirect('/some-page');

// .. then on your other page ..

$message = flash('error');
?>
```

## $\_GET, $\_POST Values and Route Symbols
To fetch a value from a request without regard to wether it comes from `$_GET`,
`$_POST`, or the route symbols, use `params($name)`. This is just like Rails' `params` hash.

```php
<?php
// get 'name' from $_GET, $_POST or the route symbols
$name = params('name');

// get 'name', set a default value if not found
$name = params('name', 'stranger');
?>
```

## File Downloads (Content-Disposition)
You can push a file to the client using the `Content-Disposition` header via
`send_file($path, $filename, $sec_expire = 0)`. `$path` points to the filesystem path
of the file to push, `$filename` will be the filename to be used in the header, and
`$sec_expire` will be the cache lifespan of the file in seconds.

```php
<?php
// push a pdf that can be cached for 180 days
send_file('/path/to/file/to/push.pdf', 'ebook.pdf', 60*60*24*180);
?>
```

## $\_FILES Values
During file uploads, to get consolidated info on the file, call `upload_info($name)`, where `$name`
is the name of the file input field. If the file input field is an array, info is grouped
conveniently by file and returned as an array.

```php
<?php
// get info on an uploaded file
$file = upload_info('photo');
?>
```

## Loading INI Files
You can make use of ini files for configuration by calling `config('source', 'myconfig.ini')`.

```php
<?php
// load the contents of my-settings.ini into config()
config('source', 'my-settings.ini');

// load another ini file, merge it with the previous one
config('source', 'my-other-settings.ini');

// get a config value from the loaded configs
$secret = config('some.setting');
?>
```

## Utility Functions
Some utility functions are also provided - for getting the client's IP,
for making a string HTML-safe, for making a string URL-safe, and for
setting/fetching values cross-scope.

```php
<?php
// get the client's ip
$ip = ip();

// store a value that can be fetched later
scope('user', $user);

// fetch a stored value
scope('user');

// client's IP
$ip = client_ip();

// escape a string's entities
html('Marley & Me');

// make a string url-safe
url('http://noodlehaus.github.com/dispatch');
?>
```

## Function Catalog
Below's the list of functions provided by Dispatch.

```php
<?php
// routing functions
function on($method, $path, $callback)
function resource($name, $cb)
function error($code, $callback = null)
function before($callback)
function after($callback)
function bind($symbol, $callback = null)
function filter($symbol, $callback)
function redirect($path, $code = 302, $condition = true)

// views, templates and responses
function render($view, $locals = null, $layout = null)
function template($view, $locals = null)
function partial($view, $locals = null)
function json_out($obj, $func = null)
function nocache()

// request data helpers
function params($name = null, $default = null)
function cookie($name, $value = null, $expire = 0, $path = '/')
function scope($name, $value = null)
function upload_info($name)
function send_file($path, $filename, $sec_expire = 0)
function request_headers($name = null)
function request_body()

// configurations and settings
function config($key, $value = null)
function site($path_only = false)

// misc helpers
function flash($key, $msg = null, $now = false)
function url($str)
function html($str, $flags = ENT_QUOTES, $enc = 'UTF-8')
function ip()

// entry point
function dispatch($method = null, $path = null)
?>
```

## Suggested Libraries

Some extra functions can be added to Dispatch via the following libraries:

* [dispatch-extras](http://github.com/noodlehaus/dispatch-extras) - encryption, caching
* [dispatch-mustache](http://github.com/noodlehaus/dispatch-mustache) - use mustache templates for your views

## About the Author

Dispatch is written by [Jesus A. Domingo].

[Jesus A. Domingo]: http://noodlehaus.github.io/

## Credits and Contributors

The following projects served as both references and inspirations for Dispatch:

* [ExpressJS](http://expressjs.com)
* [Sinatra](http://sinatrarb.com)
* [BreezePHP](http://breezephp.com)

Thanks to the following contributors for helping improve this tool :)

* Kafene [kafene](https://github.com/kafene)
* Martin Angelov [martingalv](https://github.com/martinaglv)
* Lars [larsbo](https://github.com/larsbo)
* 4d47 [4d47](https://github.com/4d47)
* Bastian Widmer [dasrecht](https://github.com/dasrecht)
* Lloyd Zhou [lloydzhou](https://github.com/lloydzhou)
* darkalemanbr [darkalemanbr](https://github.com/darkalemanbr)
* Ross Masters [rmasters](https://github.com/rmasters)

## LICENSE
MIT <http://noodlehaus.mit-license.org/>
