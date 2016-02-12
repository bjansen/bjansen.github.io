---
layout: post
title:  "Ceylon in the browser (again)"
date:   2016-02-12 16:43:00
categories: Ceylon
background: "/images/ceylon-browser/bg.png"
language: en
use_js_highlighter: true
---

As you might (or might not) know, Ceylon is more than a JVM language. It has been possible to compile Ceylon code
to JavaScript [for a long time](http://ceylon-lang.org/blog/2011/12/31/compiling-ceylon-2-js/), but other platforms
such as [Dart](https://github.com/jvasileff/ceylon-dart) or [LLVM](https://github.com/sadmac7000/org.americanteeth.ceylon_llvm)
are around the corner.

Having a JS backend means that you can actually write Ceylon code that can be 
[run in a web browser](http://ceylon-lang.org/blog/2013/02/26/ceylon-in-the-browser/), giving the opportunity to share code
between the server and the client. The [web IDE](http://try.ceylon-lang.org) is a very good example of this. Up until now, 
using Ceylon in a browser wasn't really straightforward though. The good news is, Ceylon 1.2.1 brought two major
features that overcome this problem:

* a [RepositoryEndpoint](https://modules.ceylon-lang.org/repo/1/ceylon/net/1.2.1/module-doc/api/http/server/endpoints/RepositoryEndpoint.type.html) in `ceylon.net` on the server side
* a new JS module [ceylon.interop.browser](https://herd.ceylon-lang.org/modules/ceylon.interop.browser) on the client side

Let's see how these fit together.

## Creating a new project

First things first, we need an empty project that will hold two modules:

* `com.acme.client` is a `native("js")` module that imports `ceylon.interop.browser`:

<pre><code data-language="ceylon">
native("js")
module com.acme.client "1.0.0" {
    import ceylon.interop.browser "1.2.1-1";
}
</code></pre>

* `com.acme.server` is a `native("jvm")` module that imports `ceylon.net`:

<pre><code data-language="ceylon">
native("jvm")
module com.acme.server "1.0.0" {
    import ceylon.net "1.2.1";
}
</code></pre>

## Serving Ceylon modules with RepositoryEndpoint

In order to run `com.acme.client` in a browser, we have to import it from an HTML file. The recommended way is to use 
[RequireJS](http://requirejs.org/):

```html
<!doctype html>
<html lang="en">
<head>
	<meta charset="utf-8">
	<title>Hello from Ceylon!</title>
</head>
<body>
    <div id="container">
    </div>
    <script src="//requirejs.org/docs/release/2.1.22/minified/require.js"></script>
	<script type="text/javascript">
	require.config({
		baseUrl : 'modules'
	});
	require(
			[ 'com/acme/client/1.0.0/com.acme.client-1.0.0' ],
			function(app) {
				app.run();
			}
	);
	</script>
</body>
</html>
```

Here, we tell RequireJS to use the prefix `modules` when downloading artifacts from the server, which means
we need something on the server that will listen on `/modules`, parse module names and serve the correct artifact. This is now
super easy thanks to `RepositoryEndpoint`:

<pre><code data-language="ceylon">
import ceylon.net.http.server.endpoints {
    RepositoryEndpoint
}
import ceylon.net.http.server {
    newServer
}

"Run the module `com.acme.server`."
shared void run() {
    value ep = RepositoryEndpoint("/modules");
    value server = newServer { ep };
    
    server.start();
}
</code></pre>

By default, this endpoint will look for artifacts in your local Ceylon repository. To simplify our development
workflow, let's also configure the endpoint to load artifacts from the compiler's output directory 
(`${user.dir}/modules/`):

<pre><code data-language="ceylon">
value outputDir = (process.propertyValue("user.dir") else "")
        + "/modules";

value repoManager = CeylonUtils
        .repoManager()
        .extraUserRepos(Collections.singletonList(javaString(outputDir)))
        .buildManager();

value modulesEp = RepositoryEndpoint("/modules", repoManager);
</code></pre>

This way, each time we modify files in `com.acme.client`, Ceylon IDE will rebuild the JS artifacts, which can be
immediately refreshed in the browser.

Finally, to serve static files (HTML, CSS, images etc), we need a second endpoint that uses `serveStaticFile`
to look up files in the `www` folder, and serve `index.html` by default:

<pre><code data-language="ceylon">
function mapper(Request req) 
        => req.path == "/" then "/index.html" else req.path;

value staticEp = AsynchronousEndpoint(
    startsWith("/"), 
    serveStaticFile("www", mapper),
    {get}
);

value server = newServer { modulesEp, staticEp };
</code></pre>

If we start the server and open `http://localhost:8080/`, we can see in the web inspector that the modules 
are correctly loaded:

<img src="/images/ceylon-browser/network.png"/>

## Using browser APIs

Now that we have bootstrapped a Ceylon application running in a browser, it's time to do actual things
that leverage browser APIs. To do this, we'll use the brand new `ceylon.interop.browser` which was
introduced in the Ceylon SDK 1.2.1 a few days ago. Basically, it's a set of `dynamic` interfaces that
allow wrapping native JS objects returned by the browser in nice typed Ceylon instances. For example, this
interface represents the browser's `Document`:

<pre><code data-language="ceylon">
shared dynamic Document satisfies Node & GlobalEventHandlers {
    shared formal String \iURL;
    shared formal String documentURI;
    ...
    shared formal HTMLCollection getElementsByTagName(String localName);
    shared formal HTMLCollection getElementsByClassName(String classNames);
    ...
}
</code></pre>

An instance of `Document` can be retrieved via the toplevel object `document`, just like in JavaScript:

<pre><code data-language="ceylon">
shared Document document => window.document;
</code></pre>

Note that `window` is also a toplevel instance of the dynamic interface `Window`.

`ceylon.interop.browser` contains lots of interfaces related to:

* [DOM nodes](https://www.w3.org/TR/dom/#nodes)
* [DOM events](https://www.w3.org/TR/dom/#events)
* [DOM traversal](https://www.w3.org/TR/dom/#traversal)
* [XMLHttpRequest](https://www.w3.org/TR/XMLHttpRequest/)

Making an AJAX call, retrieving the result and adding it to a `<div>` is now super easy in Ceylon:

```ceylon
import ceylon.interop.browser.dom {
    document,
    Event
}
import ceylon.interop.browser {
    newXMLHttpRequest
}

shared void run() {
    value req = newXMLHttpRequest();

    req.onload = void (Event evt) {
        if (exists container = document.getElementById("container")) {
            value title = document.createElement("h1");
            title.textContent = "Hello from Ceylon";
            container.appendChild(title);
            
            value content = document.createElement("p");
            content.innerHTML = req.responseText;
            container.appendChild(content);
        }
    };
    
    req.open("GET", "/msg.txt");
    req.send();
}
```

## Going further

Dynamic interfaces are really nice when it comes to using JavaScript objects in Ceylon. They are somewhat
similar to TypeScript's [type definitions](https://github.com/DefinitelyTyped/DefinitelyTyped), which means
in theory, it is possible to use any JavaScript framework directly from Ceylon, provided that someone writes dynamic
interfaces for its API.

The Ceylon team is currently looking for ways to [convert TypeScript type definitions to dynamic interfaces](https://github.com/ceylon/ceylon/issues/3041),
which would greatly simplify the process of adding support for a new framework/API.

The [complete source code](https://github.com/bjansen/ceylon-browser-demo) for this article is available on GitHub.