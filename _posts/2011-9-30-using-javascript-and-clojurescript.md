---
layout: post
title: Using JavaScript libraries in ClojureScript.
---

# Using JavaScript libraries in ClojureScript

One of the great features of Clojure, and one that allowed it to gain so much mindshare so quickly, is that it allows full use of any existing Java library through its comprehensive and easy Java interop syntax.

ClojureScript is also fully capable of interoperation with libraries written in its own host language, JavaScript. Unfortunately, this capability isn't as well-known or frequently used, mostly because ClojureScript leverages the powerful [Google Closure Compiler](http://code.google.com/closure/compiler/) which adds extra complexity to the compilation process. It isn't always clear exactly how to include an external library, and there are several different ways to do it that all have slightly different implications and effects. Further confounding the issue is the fact that the compiler can be run in either 'simple' or 'advanced' mode, with different implications for library importation.

Fortunately, using external or foreign libraries in ClojureScript is quite easy and effective once you know how to do it. An important first step is understanding what the Google Closure compiler does, and why ClojureScript uses it. However, if you'd rather just see examples of how to import libraries, feel free to skip to the last section of this post, "Using External Libraries."

## The Google Closure Compiler

In Closure, Google provides a powerful optimizing javascript-to-javascript compiler, including advanced features such as whole-program code path analysis with dead code elimination.

By using the Closure compiler as a step in its own compilation process, ClojureScript is able to produce output that is considerably faster and smaller than would otherwise be possible, at least without reinventing all the techniques that the Google compiler already uses.

The dead code elimination, in particular, means that developers don't need to worry about the cost in bandwidth incurred by using large libraries. The compiler will automatically strip all the code that isn't actually used in a given program, resulting in small .js files containing the bare minimum of code they need to run. This is especially important for ClojureScript, since the language itself is rather large and a ClojureScript program will result in a large .js file without Closure's dead code elimination and compression.

Google Closure (and therefore ClojureScript) provide three levels of optimization:

- **Whitespace**: This optimization only removes comments and whitespace, and does not affect the code itself.
- **Simple**: Performs localized optimizations and compressions, including renaming of local variables.
- **Advanced**: Performs whole-program analysis with global optimizations and renaming. This is the only optimization level which performs dead code elimination. Advanced compilation requires that the source JavaScript adhere to certain standards: these are outlined [on the Google Closure site](http://code.google.com/closure/compiler/docs/api-tutorial3.html).

To select which optimization to use, just set the `:optimizations` key in the options passed to the ClojureScript compiler. Possible values are `:whitespace`, `:simple` or `:advanced`

A sample invocation of ClojureScript's `build` function:

{% highlight clojure %}
(closure/build "src/cljs" {:optimizations :simple
                           :output-to "helloworld.js"})
{% endhighlight %}

At every compilation level, the Closure compiler combines all a program's dependencies and processes them together, resulting in a single .js file. Dependencies are, ultimately, specified using Google Closure's dependency resolution system, which depends on files containing the `goog.require` and `goog.provide` functions. ClojureScript's built-in namespace and require mechanisms use the same technique - compiled ClojureScript files provide their namespace and require their dependencies.

The high-level compilation process is always the same:

1. For each .cljs file in the source directory, compile it to JavaScript and add it to the compilation set.
1. For each file in the compilation set, check its dependencies, and add them to the compilation set. Perform this step recursively until all dependencies are resolved.
1. Pass the compilation set to the Google Closure compiler, which combines them, performs the requested operations, and returns the compiled JavaScript.
1. Save the resulting JavaScript to the file specified in the `:output-to` compiler option.

The ClojureScript compiler can find any .cljs dependency as long as it is named and located appropriately for its namespace in the classpath of the compiling JVM.

Google Closure aware libraries (JavaScript libraries which include a `goog.provide` statement) need to be specified. Although they can still be seamless `require`'d from ClojureScript code they have no naming convention which allows the compiler to find them, so it is necessary to inform the compiler of their location by passing the `:libs` key in the compiler options. This will ensure that they are added to the compilation set.

{% highlight clojure %}
(closure/build "src/cljs" {:optimizations :advanced
                           :libs ["libs/my-gclosure-library.js"]
                           :output-to "helloworld.js"})
{% endhighlight %}

It is not necessary to specify any of the Google Closure library modules in this way - they are included by default.

## Munging

It is important to realize that advanced compilation renames (or "munges") all the symbols and function names in a program. This renaming is consistent *within* the files processed by the compiler. Typically, this includes your .cljs source files as well as their dependencies - anything imported using `:require` in a ClojureScript namespace, or its underlying JavaScript `goog.require` mechanism.

But in order to reference symbols declared in your compiled code from *outside* your code (e.g, in in-line JavaScript in your HTML), or in order to reference variables declared outside your code *within* your code (e.g, a JavaScript library from a separate HTML script tag), special handling is required to ensure that names are not munged.

#### Exporting

Protecting symbols you declare from renaming is easy; just define `:export` metadata on any ClojureScript var, and the ClojureScript compiler will ensure that it is not munged. 

For example, a function can be declared like this:

{% highlight clojure %}
(ns example)
(defn ^:export hello [name]
  (js/alert (str "Hello," name "!")))
{% endhighlight %}

It is then available, via the same name, in an external JavaScript context:

{% highlight html %}
<script>
    example.hello("Eustace")
</script>
{% endhighlight %}

#### Externs

In order to go the other way, and reference a variable declared externally from within your code, you must provide the Google Closure compiler with an "extern file", a .js file defining javascript variables which will not be munged. This file is passed to the Google Closure compiler, and when compiling, it will not munge any names defined in it.

For example, I can try compiling the following ClojureScript (utilizing the Raphael.js library) in advanced mode without an extern file. 

{% highlight clojure %}

(defn test []
  (let [raphael (js/Raphael. 10 50 320 200)]
    (. raphael (circle 50 50 50))))

{% endhighlight %}

When I call the `test` function, it will throw a javascript error that `new Raphael(10, 50, 320, 200)).K is not a function`. `K` is the munged name of the `circle` function from Raphael, and obviously can't be found, since it isn't defined anywhere. We need to tell the compiler to preserve the name `circle`, not munge it.

I can do this by creating the following externs.js file:

{% highlight javascript %}
var Raphael = {};
Raphael.circle = function() {};
{% endhighlight %}

And use it when compiling my ClojureScript:

{% highlight clojure %}
(closure/build "src/cljs" {:optimizations :advanced
                           :externs ["externs.js"]
                           :output-to "helloworld.js"})
{% endhighlight %}

If I've included `raphael.js` in my HTML page so it's available in the global JavaScript runtime context, then everything works exactly as it should. The mere reference to `Raphael.circle` in the externs file ensures that `circle` is no longer munged during compilation, and it resolves to the `Raphael.circle` function available within the JavaScript context.

It's worth noting that because all the externs file needs are references to the vars that need protecting, any JavaScript library can serve as the extern file *for itself*. Be aware, however, that if you take advantage of this fact you will see lots of warnings during compilation. The Closure compiler attempts warns about code in an extern file that does *not* create an extern, and when using normal JavaScript as an extern file there is bound to be a fair amount of code which qualifies.

## Using External Libraries

Given these basic features of compilation, it is evident that there are two possible approaches to including an external JavaScript library in your ClojureScript program, while still maintaining the ability to leverage advanced mode compilation for your own code.

- Include the JavaScript file *within* the compilation set of the Google Closure compiler, and compiling it together with your own code. This requires that the library be compatible with advanced compilation.
- Include the JavaScript file externally to the Google Closure compilation (e.g, in a separate HTML script tag), and define externs such that your code can reference it.

#### Including external JavaScript *within* your compilation

If possible, this technique is preferable. It provides all the advantages of advanced compilation for the included javascript as well as your Clojurescript code, and performs unused code elimination *through* your code, so it only includes the parts of the library you actually use. Often, the resulting .js file containing your code and the library is smaller than the library itself before compilation. If it works, this method also incurs less hassle, because there's no need to compensate for munging, which is always consistent within the same compilation set.

Unfortunately, this technique does require that all the libraries you want to include be fully compatible with Google Closure advanced-mode compilation. Many large and popular libraries, just as jQuery or Raphael.js, are not. See the [Google Closure](http://code.google.com/closure/compiler/docs/api-tutorial3.html) site for a detailed explanation of what is required for compatibility.

There are two ways to inject an external file into the Google Closure compilation set. If the library is Google Closure aware, and includes a `goog.provide()` statement, you can just include the file using the `:lib` key in the compilation options:

{% highlight clojure %}
(closure/build "src/cljs" {:optimizations :advanced
                           :libs ["libs/foobar.js"]
                           :output-to "helloworld.js"})
{% endhighlight %}

This is likely to be a desirable technique when you have control over the JavaScript library you want to include, and can make sure it uses Google's dependency mechanisms.

Sometimes, however, you need to include a library that is not built to play in Google's dependency resolution system. In this case, ClojureScript (after revision [96b38](https://github.com/clojure/clojurescript/commit/96b38e2c951ef07b397e9d)) provides a `:foreign-libs` compiler option, which allows you to specify both the path of the JavaScript file to include, *and* the namespace that it provides.

{% highlight clojure %}
(closure/build "src/cljs" {:optimizations :advanced
                           :foreign-libs [{:file "http://foo.com/foobar.js"
                                           :provides ["foo.bar"]}]
                           :output-to "helloworld.js"})
{% endhighlight %}

Note that the value for `:file` can be either a URL or a classpath-relative path to a local file.

Once you've included your library, using either `:libs` or `:foreign-libs`, it's easy to reference it within your application. Just `:require` the library in your ClojureScript namespace declaration along with everything else:

{% highlight clojure %}
(ns helloworld
  (:require [goog.dom :as dom]
            [foo.bar :as fb]))
{% endhighlight %}

Note that defining a namespace for a file using `:foreign-libs` doesn't actually scope any variables the library creates - if the library is built to insert a `Foobar` object at the global JavaScript context, for example, you'd still need to refer to it at the global context with `js/Foobar`, not `namespace/Foobar`.

#### Referencing a library outside your compilation

If the library you want to use simply doesn't work with advanced compilation, and you still want to use it for your own code, then you can include it in the external JavaScript context, rather than within your compilation set. For example, the following HTML file both some ClojureScript code and the Raphael.js library: 

{% highlight html %}
<html> 
  <head>
    <title>Hello, world</title>
    <script type="text/javascript" src="../jslib/raphael-min.js"></script>
    <script type="text/javascript" src="helloworld.js"></script>
  </head>
  <body>
    <script type="text/javascript">
      goog.require('helloworld')
    </script>
  </body>
</html>
{% endhighlight %}

This means that Raphael is loaded and ready to use. But if your ClojureScript was compiled in advanced mode, you won't be able to refer to it directly - the function names will have been munged out of recognition.

To prevent munging, you need to include an "externs" file and include it in your compilation options. You can either create an externs file manually, or if you're willing to put up with some warnings, you can use the library as its own externs file, as is demonstrated below.

{% highlight clojure %}
(closure/build "src/cljs" {:optimizations :advanced
                           :externs ["lib/raphael-min.js"]
                           :output-to "helloworld.js"})
{% endhighlight %}

See the section above, "Munging", for a detailed explanation of how extern files work.

Once you've included Raphael in your JavaScript execution context, and ensured that your external references won't be munged, you can use the library from your ClojureScript code using standard interop features:

{% highlight clojure %}
(def paper (js/Raphael 0 0 500 500))
(def circle (. paper (circle 50 50 10)))
(. circle (attr "fill" "#f00"))
{% endhighlight %}

And that's all there is to it!
