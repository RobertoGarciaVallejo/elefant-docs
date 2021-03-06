# Page routing and handler basics

Requests in Elefant are all handled by the main `index.php` file, which uses `lib/Controller.php` to determine where to send requests for handling. Here is a diagram that illustrates the flow of a request through Elefant:

![Request flow in Elefant framework](http://jbroadway.github.com/elefant/diagrams/request_flow.png)

1. A new request is received by `index.php`, the front controller
2. The request is routed to the correct handler, e.g., `myapp/handler`
3. The handler determines how to handle the request, and calls on your model(s) to return the requested data
4. The handler renders the data with a view template and returns it to the front controller
5. The front controller optionally renders the final output with a layout template
6. The response is returned to the user

## Routing basics

In Elefant, friendly URLs are automatic, and automatically routed to the correct handler via Elefant's smart routing. This is different than many frameworks that require you to specify each URL pattern manually.

In Elefant, URLs are matched to app and handler names, so you simply craft your app and handler names to create the URL naming conventions that you want, and the routing is flexible enough to account for most any needs.

The controller takes `$_SERVER['REQUEST_URI']` and looks for matching handlers in the following pattern:

    /{appname}/{handler} -> apps/{appname}/handlers/{handler}.php

Additional folder levels in the URL translate to sub-folders in the handlers folder. For example:

    /myapp/path/to/handler

The controller will try to map this to:

    apps/myapp/handlers/path/to/handler.php

If that's not found, it will take the last part of the URL out, add it to the `$this->params` array (`$this` is the Controller object), and look for a handler at the next level down:

    apps/myapp/handlers/path/to.php

If that's not found, it repeats the cycle again:

    apps/myapp/handlers/path.php

And of course, `$this->params` now contains 2 values: `['to', 'handler']`. If this isn't found, it adds the last element to the start of `$controller->params` so that it contains `['path', 'to', 'handler']` and checks for the default handler for the app:

    apps/myapp/handlers/index.php

And failing that, finally sends the request to the default handler.

But what about top-level requests?

Simple. A top-level request goes like this:

    /myapp

Becomes:

    apps/myapp/handlers/index.php

And failing that, the default handler once again takes over. Requests to `/` of course go directly to the default handler. Using this knowledge, it becomes easy to route requests to individual handlers, and to craft the URLs for our applications.

## Default handler

There is a `default_handler` setting in `conf/config.php` that allows you to define a handler that any non-matching request falls back to. Writing a custom default handler, you can use any URL naming convention you like, adding to the built-in routing with your own rules.

The default setting of this is `admin/page`, which serves pages from the `webpage` table, or returns a 404 error if no page is found. If you write a custom default handler, make sure to handle 404 requests as well.

## How a handler is called

Once the right handler has been found, the `Controller::handle()` method is called, which roughly does the following:

	<?php
	
	function handle ($handler, $data = array()) {
		ob_start ();
		require ($handler);
		return ob_get_clean ();
	}
	
	?>

To access the controller object, use `$this`. To return output from your handler, simply echo it and the controller will capture it. So to render a view template, you would say:

	<?php
	
	echo View::render (
		'myapp/viewtemplate',
		array (
			'param1' => 'Some value'
		)
	);
	
	?>

> Note: We did leave out many extras like caching and chunked output support, but the above code illustrates what the `handle()` method is doing for you.

## Writing your own handler

Let's start with a hello world and then look at a few extras from there. To save time, if you're a command-line guy like myself, you can create a new app for yourself as follows:

	$ cd /path/to/your/site
	$ ./elefant build-app myapp

This will create a new app named `myapp` in the `apps` folder with a proper app structure, a single handler and a view. The handler, `apps/myapp/handlers/index.php`, contains:

	<?php echo View::render ('myapp/index', array ()); ?>

But that's getting ahead of ourselves. For now, replace that with the following:

	<?php echo 'Hello world'; ?>

Now you should be able to go to `/myapp` at your site and see it display ""Hello world"". If you view the source, you'll notice there's a bit of extra HTML we didn't write wrapped around it. This comes from the global layout template, more on that on the [[Templates]] page.

### Using the extra URL components in your handler

Suppose you want to create dynamic URLs of the form /myapp/{some_id} and map these to a database table. In your handler you can get that value like this:

	<?php echo 'Hello ' . $this->params[0]; ?>

Now try going to /myapp/world or /myapp/everyone and see the output change based on the URL you use.

To turn this into a list of named parameters, you can simply add this line to the top of your handler:

	<?php
	
	list ($name) = $this->params;
	
	echo 'Hello ' . $name;
	
	?>

### Specifying a view for your handler

To send your handler output through a specific view, simply call `$tpl->render()` like this:

	<?php
	
	echo View::render ('myapp/index', array ('who' => $this->params[0]));
	
	?>

This calls the view `apps/myapp/views/index.html`.

If we add the following to the file `apps/myapp/views/index.html`, we'll get the same output as before.

	Hello {{ who }}

### Alternate layouts

You can specify a different global layout template for your handler like this:

	<?php
	
	$page->layout = 'myapp';
	echo View::render ('myapp/index', array ('who' => $this->params[0]));
	
	?>

Now in the `layouts` folder, duplicate the `default.html` file and name it `myapp.html`. Edit as you please so you can see the change take effect.

### Using no layout

To use no layout template, you can set the `$page->layout` to false. This stops the output from being sent to a template at all, which is useful for things like JSON-formatted responses. For example:

	<?php
	
	$page->layout = false;
	header ('Content-Type: application/json');
	echo json_encode (array ('foo' => 'bar'));
	
	?>

### Calling handlers from each other

As your application grows, you may want to combine the output of one handler with another. To do this, simply run the other handler inside the current one:

	<?php
	
	echo $this->run ('myapp/otherhandler');
	
	?>

Remember, we're in `Controller::handle` so we can use `$this` to refer to the controller. If you need to pass custom data to a handler, you can do so via:

	<?php
	
	echo $this->run ('myapp/otherhandler', array ('foo' => 'bar'));
	
	?>

This is then accessible in the other handler as `$this->data`.

A handler can check if it's being called from the web or internally with the `$this->internal` value, which is false for directly external requests, and true whenever a handler is called from within another handler or view template.

## Redirects

To redirect a request and exit the script, use the following command:

	<?php
	
	$this->redirect ('/go/here');
	
	?>
