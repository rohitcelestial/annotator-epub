Upgrading guide
===============

Annotator 2.0 represents a substantial change from the 1.2 series, and
developers are advised to read this document before attempting to upgrade
existing installations.

In addition, plugin authors will want to read this document in order to
understand how to update their plugins to work with the new Annotator.

.. contents::


Motivation
----------

The architecture of the first version of Annotator dates back to 2009, when the
Annotator application was developed to enable annotation in a project called
"Open Shakespeare". At the time, Annotator was designed primarily as a drop-in
annotation application, with only limited support for customisation.

Over several years, Annotator gained primitive support for plugins, which
allowed developers to customise and extend the behaviour of the application.

In order to ensure a stable platform for future development, we have made some
substantial changes to Annotator's plugin architecture. This unfortunately means
that the upgrade from 1.2 to 2.0 is not always going to be painless.

Specifically, if you're very happy with Annotator 1.2 as it is now, you may wish
to continue using it until such time as the features added to the 2.x series
attract your interest -- we'll continue to answer your questions about 1.2. The
main target audience for Annotator 2.0 is people who have been frustrated by the
coupling and architecture of 1.2.

If any of the following apply to you, Annotator 2.0 should make you happier:

- You work on an Annotator application which overrides part or all of the
  default user interface.

- You have made substantial modifications to the annotation viewer or editor
  components.

- You use a custom storage plugin.

- You use a custom server-side storage component.

- You integrate Annotator with your own user database.

- You have a custom permissions model for your application.

If you want to know what you'll need to do to upgrade your application or
plugins to work with Annotator 2.0, keep reading.


Upgrading an application
------------------------

The first step to understanding what you need to do to upgrade to 2.0 is to
identify which parts of Annotator 1.2 you use. Review the list below, which
attempts to catalogue Annotator 1.2 patterns and demonstrate the new patterns.


jQuery integration
~~~~~~~~~~~~~~~~~~

Annotator 1.2 ships with a jQuery integration, allowing you to write code such
as::

    $('body').annotator();

This has been removed in 2.0. Here's what you'd write now::

    new annotator.App()
        .include(annotator.ui.main, {element: document.body})
        .start();


Core annotator
~~~~~~~~~~~~~~

The default construction of Annotator in 1.2 looked like this::

    new Annotator(document.body);

This sets up an Annotator with a user interface. The default construction of
Annotator in 2.0 doesn't set up a user interface so you need to include the
``annotator.ui.main`` module if you want similar behaviour::

    new annotator.App()
        .include(annotator.ui.main, {element: document.body})
        .start();


Store plugin
~~~~~~~~~~~~

In Annotator 1.2 configuring storage looked something like the following::

    new Annotator(elem)
        .addPlugin('Store', {
            prefix: 'http://example.com/api',
            loadFromSearch: {
                uri: window.location.href,
            },
            annotationData: {
                uri: window.location.href,
            }
        });

This code is doing three distinct things:

1. Load the "Store" plugin pointing to an API endpoint at
   ``http://example.com/api``.
2. Make a request to the API with the query ``{uri: window.location.href}``.
3. Add extra data to each created annotation containing the page URL: ``{uri:
   window.location.href}``.

In Annotator 2.0 the configuration of the storage component
(:func:`annotator.storage.http`) is logically separate from both a) the
loading of annotations from storage, and b) the extension of annotations with
additional data. An example which replicates the above behaviour would look like
this in Annotator 2.0::


    var pageUri = function () {
        return {
            beforeAnnotationCreated: function (ann) {
                ann.uri = window.location.href;
            }
        };
    };

    var app = new annotator.App()
        .include(annotator.ui.main, {element: elem})
        .include(annotator.storage.http, {prefix: 'http://example.com/api'})
        .include(pageUri)

    app.start()
       .then(function () {
           app.annotations.load({uri: window.location.href});
       });

We first create an Annotator extension module (once known as a plugin) that sets
the ``uri`` property on new annotations. Then we create and configure an
:class:`~annotator.App` that includes the :func:`annotator.storage.http` module.
Lastly, we start the application and load the annotations using the same query
as in the 1.2 example.


Auth plugin
~~~~~~~~~~~

The auth plugin, which in 1.2 retrieves an authentication token from an API
endpoint and sets up the Store plugin, is not yet available in 2.0.



Upgrading a plugin
------------------

The first thing to know about Annotator 2.0 is that we are retiring the use of
the word "plugin". Our documentation and code refers to reusable pieces of code
such as :func:`annotator.storage.http` as "modules". Modules are included into
an :class:`~annotator.App`, and are able to register providers of named
interfaces (such as "storage" or "notifier"), as well as providing runnable
"hook functions" that the app may call at important moments. Annotator 1.2's
lifecycle events (``beforeAnnotationCreated``, ``annotationCreated``, etc.) are
still available as hooks, and it should be reasonably straightforward to migrate
plugins which simply respond to lifecycle events.

The second important observation is that Annotator 2.0 is written in plain old
JavaScript, not CoffeeScript. This may not make much difference to you, and you
can continue to write plugins in either dialect as you see fit. That said, it
should now be clearer and simpler to write modules in JavaScript.

Lastly, writing an extension module is simpler and more idiomatic than writing a
plugin. Whereas Annotator 1.2 assumed that plugins were "subclasses" of
``Annotator.Plugin``, in Annotator 2.0 a module is simply a function which
returns an object containing hook functions. It is through these hook
functions that plugins provide the bulk of their functionality.

For full documentation on writing modules, please see :doc:`module-development`.
Below you'll find a few examples on how to translate some simple plugins.

A trivial plugin
~~~~~~~~~~~~~~~~

Here's an Annotator 1.2 plugin that logs to the console when started::

    class Annotator.Plugin.HelloWorld extends Annotator.Plugin
      pluginInit: ->
        console.log("Hello, world!")

Or, in JavaScript::

    Annotator.Plugin.HelloWorld = function HelloWorld() {
        Annotator.Plugin.call(this);
    };
    Annotator.Plugin.HelloWorld.prototype = Object.create(Annotator.Plugin.prototype);
    Annotator.Plugin.HelloWorld.prototype.pluginInit = function pluginInit() {
        console.log("Hello, world!");
    };

Here's the equivalent for Annotator 2.0::

    function hello() {
        return {
            start: function () {
                console.log("Hello, world!");
            }
        };
    }