---
layout: post
title:  "A Year in Clojure"
date:   2017-09-15 12:00:00
tags:   clojure lisp
---
A lot has changed since my last post nearly a year ago. I want to write about my experience coming from a ML-family language to a Lisp. It's not going to be very informative as a tutorial; it's mostly scattershot notes and things I've found interesting along the way.

> _Updated 2018-02-11: Added notes on metadata, pre-/post-conditions, and refactoring._

## Background

My very first exposure to functional programming was through a book called [The Little Schemer](https://mitpress.mit.edu/books/little-schemer): a fairly thin, flourescent, mind-melting paperback that I'm not quite sure how I came upon. I'd been programming for ten years but it was like my inner programmer was reincarnated.

I was working mostly with .NET at the time so F# was a natural choice for functional exploration, even though it wasn't a Lisp. After a few years of F# I found myself back in Lispland.

<img src="/img/oh-lawd.jpeg" style="float: right; max-width: 250px; margin: 10px" />

My new language was [Clojure](https://clojure.org) and it was totally foreign. So many parentheses instead of semantic whitespace. No rigid type system to keep me honest. How am I going to deal with these parentheses? _(Thank you, [Parinfer!](https://shaunlebron.github.io/parinfer/))_

While this was another functional language, it encouraged a very different style of programming---one I thought I wouldn't like with all the ambiguity of dynamic typing. How am I supposed to feel my program is even possibly correct if the compiler isn't telling me so? ML is so expressive in its ability to model a problem domain with algebraic data types, and for the most part isn't littered with type hints thanks to inference.

## First Impressions

First I looked for analogues of ML/F#. I quickly found all the familiar functional tidbits pretty much unchanged.

I wondered why there's a `reduce` in clojure.core but no `fold`? (It's just a different arity of `reduce` that's missing the initial state.)

There were record types but they didn't seem often used. Polymorphism via protocols and multimethods seemed like powerful tools, but I only see them used sparingly.

Where's the pattern matching? I found [clojure.core.match](https://github.com/clojure/core.match) but I've never felt motivated enough to try it, nor have I seen it used. I have no idea about the state of pattern matching in other Lisps, but I assume it's a similar situation. `cond` gets you most of the way there I suppose?

No tuple type? Vectors are tuples. "What if one of the vectors is of a different length?" I gasped as I clutched my pearl monads.

I was sure I'd miss discriminated union types. There seemed like no stand-in for those, although I think clojure.spec's `or` might facilitate something similar.

The reality of the Clojure language design started to sink in. The language itself was extremely _basic._ There was so little syntax to learn that I could be productive almost immediately. Conversely, even when writing F# daily I sometimes forgot less common syntax. (Do I need a semicolon or comma in this record definition?!)

I was able to easily dive into clojure.core source interactively while I coded, which usually required third-party tools for FSharp.Core and .NET in general. I do find myself reading more third-party Clojure code out of necessity, a little more than I needed to with F#/.NET, and I chalk it up to needing to read code to see what it _does_ rather than making assumptions based on the _types_ it works on.

## Keywords

These are everywhere in Clojure. I think of these as more-or-less interned strings that are granted some special abilities.
```clojure
:foo ;; a keyword
```
You can think of that as `"foo"` for most intents and purposes, and there's a pair of functions for converting between keywords and strings.
```clojure
(keyword "foo")
=> :foo
(name :foo)
=> "foo"
```

Keywords are the de facto _type_ for keys in map literals e.g.
```clojure
{:name "Bill", :age 23}
```

Keywords can also be used as functions of maps i.e. to get a value from the map by keyword: `(:foo m)`.
They can be namespaced to avoid naming collisions e.g. `:my-namespace/foo`. Note the `name` of the namespaced keyword is just the part after the `/`.

Keywords don't have to be _declared_ anywhere before use; they're just literals. I've read/heard the Clojure designers encourage the use of namespaced keywords, but I rarely encounter them outside clojure.spec definitions.

It's also common to _keywordize_ ingested data structures like JSON objects, XML DOMs, etc.

## Arity

The really neat-o thing about Clojure function arities is the ability to define functions of arbitrary arity. In these cases the numerous arguments are bound to a collection: either a vector or a map, and a map can be thought of as a collection of key/value tuples. A basic example of this is the `+` function:
```clojure
(+ x y & more)
```
Here, `more` can be any number of additional arguments. This ties into another concept that was new to me: _applying_ a sequence of arguments to a function.
```clojure
(apply + [1 2 3 4 5 6 7 8 9])
=> 45
```

Clojure has a nice compact way of defining multiple function arities: a single function symbol form with each arglist + implementation in parentheses:
```clojure
(defn foo
 ([a]
  (println a))
 ([a b]
  (println a b)))
```
And you do get some compile-time safety checks on function arities.

I wondered what the equivalent of F#'s `Seq.zip` was in Clojure but a quick search turned up nothing. I didn't realize you could simply pass _more than one_ collection to `map`, and the step function would then take an argument per collection. This seemed really like a really elegant use of variable-arity functions and there are several examples of this technique in clojure.core.

Multi-arity functions are also one way Clojure deals with named optional arguments. Both these function calls use the same function definition:
```clojure
(foo 123)
(foo 123 :append true :path "~/.sup")
```
The additional "arguments" are actually treated as a map and I've seen this style referred to as _kwargs_. It's a form of destructuring in the function signature like this:
```clojure
[num & {:keys [append path]}]
```
This syntax obviously requires an even number of key/value pair arguments.

## Destructuring

I used destructuring in F# most often in pattern matching clauses. In Clojure I'm finding most destructuring I do involves getting values from maps, followed by grabbing items from sequences by position.

The map destructuring syntax is powerful and a bit confusing at first although it seems so simple now. It even allows you to specify default values for keys that aren't present in the map, all in one line:
```clojure
{:keys [foo] :or {foo 1} :as m}
```
`foo` will have the map's value for key `:foo`, or `1` if the map doesn't have a `:foo` key, and `m` is bound to the entire map. This makes getting values from maps very easy and terse. You can also _alias_ destructurings inline:
```clojure
{another-name :foo}
```

Another neat feature is positional destructuring, commonly used with vectors. In addition to the traditional _head and tail_ destructuring of sequences, you have more flexibility with the `&` syntax.
```clojure
[x & xs] [1 2 3]
```
Here, `x` is `1` and `xs` is `[2 3]`. But you can also do this:
```clojure
[x & [next]] [1 2 3]
```
Here, `x` is `1` and `next` is `2`. The nested destructuring indicates we only care about the first element of the tail.

You can also destructure more than one binding before the _tail_ like so:
```clojure
[x y & zs] [1 2 3]
```
Here, `x` is `1`, `y` is `2`, and `zs` is `[3]`.

In the "anything goes" spirit, positional destructuring doesn't throw exceptions on "failure":
```clojure
[x y z] [1 2]
```
Here, `z` will simply be nil as there is no third element. The same goes for map destructuring i.e. you just get nil back for keys that aren't in the map.

All this destructuring is also available in function argument signatures! This allows for flexible function signatures and can get a lot of argument destructuring "out of the way" of the function's body---a nice separation of concerns. In some sense, you can imagine all Clojure function argument bindings as a destructured vector.

Destructurings can be nested but it becomes unreadable two–three levels deep.

## Data

Clojure and F# have similar data structures but working with them in Clojure feels lower-ceremony. The literal syntax is nice and terse and all the basic data structures can be conveniently composed and transformed.

Maps would seem to be Clojure's commonplace record type, even though it has `defrecord` which seems [somewhat deprecated](https://groups.google.com/d/msg/clojure/pnDl4OgzqBM/zjDioSsxkvEJ). Something that bugged me about this initially was there were no guarantees a _record_ (map) would contain what you expected, or that it wouldn't contain a bunch of stuff you _didn't_ expect! I came to realize this doesn't matter in practice as often as I thought.

### clojure.spec

I can't talk about data in Clojure without talking about [clojure.spec](https://clojure.org/about/spec)!

I believe there's a drawback to the ambiguities of working with data in Clojure, and one that clojure.spec can address: sometimes you really do need a rigid _specification_ of data. With clojure.spec you can _document_ the shape of data (in a machine-readable format), then use that to _verify_ properties of functions of that data (via generative and property-based testing), and a whole lot more. I've even leveraged clojure.spec to generate API documentation and sample code. Watch this [excellent talk](https://www.youtube.com/watch?v=oyLBGkS5ICk) for more on clojure.spec.

Spec solves many problems but what interested me at first was using it as a sort of opt-in _gradual typing_ system for my projects. I could write specs for only the data structures and functions I wanted. With function specs, I could assert the inputs and outputs to specific functions at runtime; not quite as reassuring as compile-time but better than nothing!

I hope clojure.spec adoption continues and this kind of opt-in type checking becomes more prevalent. I think there's a very sweet spot to occupy between _everything must be typed_ and total dynamic ambiguity. The new Clojure JDBC library shipping with specs is hopefully a good sign of what's to come.

### Code as Data

Whatever superlative or diss you might use to describe a Lisp's syntax or lack thereof, it makes one thing evident: your code itself is just data, and data can be _manipulated._ A function at rest is just a list containing other lists and data structures. _Code as data_ was one of the profound enlightenment moments I've had as a programmer, and one that can't be realized in most popular languages.

Clojure's macros allow you to work with code as data. Much of Clojure's core functionality is written as macros that get _expanded_ into final forms before evaluation. F# has some metaprogramming facilities but they're clunky (and fragmented depending on what kind of syntax tree you want).

I don't often write new macros, and it's not advised to write macros where functions work just as well, but they're invaluable in certain contexts.

A few times I've been able to do some trivial refactorings of code _programmatically_ in the REPL. Of course .NET has Roslyn but it's a totally separate facility; in Clojure this ability is inherited more or less by virtue of being a Lisp.

### Data as Code

Some core Clojure data structures can be used as functions. I didn't know what to make of this at first but I love it now. For example, a map is a function of one (key) argument. A set is a function of one (member) argument.

```clojure
(def my-set #{1 2 3})       ;; a set of integers
(my-set 1) => 1             ;; returns the member value
(my-set 3) => 3             ;; returns the member value
(my-set "no") => nil        ;; returns nil 
(map my-set [3 2]) => (3 2) ;; mapping the set over a vector
```

Lately I find myself thinking more in terms of _data_ than procedural code. For example, sometimes it's nice to express program logic _declaratively_ with data as opposed to control-flow/`case`/`cond` statements. Consider this `case`:
```clojure
(case x
  "High"   100
  "Medium" 50
  "Low"    0)
 ```

 This can also be expressed as a map, and that map can be used as a _function_ that returns the matched key's value. This code has the same effect as the `case` above:
```clojure
(def foo
  {"High"   100
   "Medium" 50
   "Low"    0})
(some foo "Medium") => 50
```
Here we're passing the map `foo` as a _function_ to `some`, which will return the first value of the matching key.

This approach is nice when you want to define some "rules" in one place and it use them in many, or maybe it's loaded from a database or configuration file.

### Nil

Some functional languages go to lengths to cordon null/nil values. Clojure embraces nil (in most cases). I remember thinking "there should be an Option type" a lot in my first days. Now I embrace the nil. I rarely find myself needing to write defensive nil-check code.

I do miss [nullable comparison operators](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/symbol-and-operator-reference/nullable-operators) and I wish there was something like F#'s `Option` module, but it's just a matter of defining your own functions to do the same.

There are many nil-friendly core functions, and some nil-adverse functions too. It's not always clear which functions will happily propagate nils and which ones will throw exceptions, so I often find myself experimenting in the REPL to find out. It's not something I have to worry about much, but a nil-propagation bug can be hard to debug when it happens. Personally, I'd rather see less nil-propagation in the Clojure core libraries but it does make sense in many cases.

### Metadata

Clojure can attach [metadata](https://clojure.org/reference/metadata) to objects. You can use `with-meta` (or a prefix syntax `^`) to tag objects with metadata. The object/value will still present itself to callers as before, but now it has a secret bag (map) of goodies and you can inspect it with the `meta` function.

For example, a function to pad the left side of a collection that also returns metadata for the number of padding items:

```clojure
(defn left-pad [n p s]
  (let [diff (- n (count s))]
    (with-meta (concat (repeat diff p) s)
               {:padding diff})))
(left-pad 5 0 [1 2 3])
=> (0 0 1 2 3) ;; looks like a typical result
(meta (left-pad 5 0 [1 2 3]))
=> {:padding 2} ;; but it has metadata too
```

The metadata facilities are also commonly used for compiler type hints, docstrings, deprecation tags, etc. In fact, that `left-pad` function itself has been decorated with a bunch of metadata behind-the-scenes, including its definition location down to the line and column!

```clojure
(meta #'left-pad) ;; #' gets the Var of the function definition
=>
{:arglists ([n p s]),
 :line 8,
 :column 1,
 :file "/Users/Taylor/Projects/playground/src/playground/core.clj",
 :name left-pad,
 :ns #object[clojure.lang.Namespace 0x547b55b7 "playground.core"]}
```

## REPL-driven Development

I didn't use the F# REPL that much for interactive _development_ as much as I did for interactive execution. The Clojure REPL experience is far more ingrained into my development workflow, and I like it. I feel I'm more of an inhabitant of the program as I write it rather than an observant designer. I also get more immediate feedback about the runtime behavior of code, which can be hard to reason about in functional languages.

I think there's another reason for the REPL being so integral to the Clojure development experience: discovery through interaction. Clojure just doesn't have the information I would normally get via type definitions (and IntelliSense/auto-complete). In Clojure it's more natural to get that information by playing with the code in-hand. Doc strings help but they aren't schematics. Again, another area where clojure.spec can change the landscape.

I haven't tried this, but I've heard you can be so bold as to have a [deployed application host a network REPL](https://stackoverflow.com/questions/48209370/sending-message-to-clojure-application-from-terminal/48209930#48209930) which can be used to remotely inspect (and monkey with) the running application's code and state. 😮

### Debugging

I used an interactive debugger much more often with F#. I've only tried to interactively debug Clojure once or twice. It's possible but I find it's just easier to rely on REPL evaluation, `printf` debugging, and break-like macros in most scenarios. I don't get the feeling interactive debuggers are a hot topic in the Clojure community, but I've seen this trip up others that were used to setting breakpoints, watches, etc. in imperative languages.

## Invariants

One of my favorite things about more strongly-typed functional languages is the ability to _make illegal states unrepresentable._ In Clojure it's very hard to make illegal states unrepresentable, in fact I'd say in most cases it's just not worth the effort. In ML variants it's somewhat difficult to write a program that compiles but doesn't work on some level; I quickly found in Clojure it's quite easy to write a program that compiles but doesn't work at all.

Clojure feels very open and permissive when it comes to data and passing it around. The thought that a caller could pass _anything_ as an argument was unsettling at first.

As I embraced this _laissez faire_ approach, the Clojure philosophy started to make sense. Clojure codebases are driven more by convention than contract. In F# I mostly thought about problems in terms of the types I'd use to model them, essentially defining a contract. If my program compiled I could be fairly certain it'd do at least something like what I intended. This meant putting a lot of upfront thought into the _types_ I'd need to model the problem.

In Clojure I find myself doing the opposite---just diving into the REPL and immediately writing code that operates on data. That same code can be sculpted into tests (if I'm not too lazy) and ultimately end up in production.

### Refactoring

In my experience---and related to the topic of invariants---non-trivial Clojure can be more difficult to refactor than similar F# code. This isn't hard to imagine since there are no guardrails ensuring your functions/types stay aligned with your assumptions. (Yes, you can use clojure.spec but it's still in alpha and is also totally opt-in unlike static type systems.)

I've felt this pain a few times and the only conclusions I've drawn are to use clojure.spec for non-trivial code, especially around domain boundaries, and try even harder to solve very small problems in ways that can compose to solve larger ones. This way your small-problem code is more likely to be "obviously correct" and more stable (or less likely to require adaptation). The best example of this approach I can point to is clojure.core itself, although it doesn't have to worry itself with messy "business" domains.

### Pre- and Post-conditions

Clojure does allow you to define pre- and post-condition functions for assertions on inputs and output, but they don't seem very common in practice and they only work at _runtime_.

```clojure
(defn product-str [& nums]
  {:pre  [(every? number? nums)] ;; every input must be a number
   :post [(string? %)]}          ;; output must be a string
  (str (apply * nums)))

(product-str 1 1/2 3.0 4 5)
=> "30.0"
(product-str 1 2 "3")
;; throws "CompilerException java.lang.AssertionError: Assert failed: (every? number? nums), compiling ..."
```

I suspect clojure.spec's function instrumentation would mostly deprecate pre- and post-conditions, but I'm just guessing.

## (Parenthetical Gestalt!)

It's the carefully and conservatively curated confluence (alliteration achievement unlocked 🏅) of Clojure's design decisions and philosophies, _especially_ those that were hard for me to embrace at first, that make it such an enjoyable programming experience for me.

I pre-judged Clojure for being a dynamic language because I'd never enjoyed working in a dynamic language before. As I look back on the things I thought I'd miss from a strongly typed language, I can easily say I've faired just as well without them.

P.S. There were a bunch of other cool Clojure tidbits I wanted to mention but this post was getting too long!
