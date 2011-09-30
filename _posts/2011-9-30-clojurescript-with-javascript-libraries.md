---
layout: post
title: Using JavaScript libraries in ClojureScript.
---

# Using JavaScript libraries in ClojureScript

One of the great features of Clojure, and one that allowed it to gain so much mindshare so quickly, is that it allows full use of any existing Java library through its comprehensive and easy Java interop syntax.

ClojureScript is also fully capable of interoperation with libraries written in its own host language, JavaScript. Unfortunately, this capability isn't as well-known or frequently used, mostly because ClojureScript leverages the powerful [Google Closure Compiler](http://code.google.com/closure/compiler/) which adds extra complexity to the compilation process. It isn't always clear exactly how to include an external library, and there are several different ways to do it that all have slightly different implications and effects. Further confounding the issue is the fact that the compiler can be run in either 'simple' or 'advanced' mode, with different implications for library importation.

Fortunately, using external or foreign libraries in ClojureScript is quite easy and effective once you know how to do it. An important first step is understanding what the Google Closure compiler does, and why ClojureScript uses it.

## The Google Closure Compiler

In Closure, Google provides a powerful optimizing javascript-to-javascript compiler, including advanced features such as while-program code path analysis with dead code elimination.

By using the Closure compiler as a step in its own compilation process, ClojureScript is able to produce output that is considerably faster and smaller than would otherwise be possible, at least without reinventing all the techniques that the Google compiler already uses.

The dead code elimination, in particular, means that developers don't need to worry about the cost in bandwidth incurred by using large libraries. The compiler will automatically strip all the code that isn't actually used in a given program, resulting in small .js files containing the bare minimum of code they need to run. This is especially important for ClojureScript, since the language itself is rather large and a ClojureScript program will result in a large .js file without Closure's dead code elimination and compression.

Google Closure (and therefore ClojureScript) provide three levels of optimization:

- **Whitespace**: This optimization only removes comments and whitespace, and does not affect the code itself.
- **Simple**: Performs localized optimizations and compressions, including renaming of local variables.
- **Advanced**: Performs whole-program analysis with global optimizations and renaming. This is the only optimization level which performs dead code elimination.

All three optimization levels also combine all a program's dependencies and process them together, resulting in a single .js file.

To select which optimization to use, just set the `:optimizations` key in the options passed to the ClojureScript compiler. Possible values are `:whitespace`, `:simple` or `:advanced`

A sample invocation of ClojureScript's `build` function:

{% highlight clojure %}
(closure/build "src/cljs" {:optimizations :simple
                           :output-to "main.js"})
{% endhighlight %}


## Munging

It is important to realize that advanced compilation renames (or "munges") all the symbols and function names in a program. This renaming is consistent *within* the files processed by the compiler. Typically, this includes your .cljs source files as well as their dependencies - anything imported using `:require` in a ClojureScript namespace, or its underlying JavaScript `goog.require` mechanism.

But in order to reference symbols declared in your compiled code from *outside* your code (e.g, in in-line JavaScript in your HTML), or in order to reference variables declared outside your code *within* your code (e.g, a JavaScript library from a seperate HTML script tag), special handling is required to ensure that names are not munged.

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

In order to go the other way, and reference a variable declared externally from within your code, you must provide the Google Closure compiler with an "extern file", a .js file containing javascript variables which will not be munged.

For example, if I compile the following ClojureScript (utilizing the Raphael.js library) in advanced mode without an extern file, when I call the `test` function, it will throw a javascript error that `new Raphael(10, 50, 320, 200)).K is not a function`. Evidently, `K` is the munged name of the `circle` function from Raphael.

{% highlight clojure %}

(defn test []
  (let [raphael (js/Raphael. 10 50 320 200)]
    (. raphael (circle 50 50 50))))

{% endhighlight %}

However, I can create the following externs.js file:

{% highlight javascript %}
var Raphael = {};
Raphael.circle = function() {};
{% endhighlight %}

And use it when compiling my ClojureScript:

{% highlight clojure %}
(closure/build "src/cljs" {:optimizations :advanced
                           :externs ["externs.js"]
                           :output-to "main.js"})
{% endhighlight %}

If I've included `raphael.js` in my HTML page, then everything works exactly as it should. The mere reference to `Raphael.circle` in the externs file ensures that `circle` is no longer munged during compilation, and it resolves to the `Raphael.circle` function available within the JavaScript context.

It's worth noting that because all the externs file needs are references to the vars that need protecting, any JavaScript library can serve as the extern file *for itself*. Be aware, however, that if you take advantage of this fact you will see lots of warnings during compilation. The Closure compiler attempts warns about code in an extern file that does *not* create an extern, and when using normal JavaScript as an extern file there is bound to be a fair amount of code which qualifies.


