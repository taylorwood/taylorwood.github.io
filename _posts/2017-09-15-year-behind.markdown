---
layout: post
title:  "A Year in Clojure"
date:   2017-09-15 12:00:00
tags:   clojure lisp
---
A lot has changed since my last post nearly a year ago. I want to write about my experience coming from a ML-family language to a Lisp. It's not going to be very informative as a tutorial; it's mostly scattershot notes and things I've found interesting along the way.

## Background

My very first exposure to functional programming was through a book called [The Little Schemer](https://mitpress.mit.edu/books/little-schemer): a fairly thin, flourescent, mind-melting paperback that I'm not quite sure how I came upon. I'd been programming for ten years but it was like my inner programmer was reincarnated.

I was working mostly with .NET at the time so F# was a natural choice for functional exploration, even though it wasn't a Lisp. After a few years of F# I found myself back in Lispland.

<img src="/img/oh-lawd.jpeg" style="float: right; max-width: 250px; margin: 10px" />

My new language was [Clojure](https://clojure.org) and it was totally foreign. So many parentheses instead of semantic whitespace. No rigid type system to hold me close. How am I going to deal with these parentheses? _(Thank you, Parinfer.)_

While this was another functional language, it encouraged a very different style of programming---one I wasn't used to and thought I wouldn't like. The chaotic ambiguity of dynamic typing. How am I supposed to feel my program is even possibly correct if the compiler isn't telling me so? ML is so expressive in its ability to model a problem domain with algebraic data types. How could I ever do without it?

## First Impressions

First I looked for analogues of ML/F#. I quickly found all the familiar functional tidbits pretty much unchanged.

I wondered why there's a `reduce` in clojure.core but no `fold`? (It's just a different arity of `reduce` that's missing the initial state.)

There were record types but they didn't seem often used. Polymorphism via protocols and multimethods seemed like powerful tools, but I only see them used sparingly.

Where's the pattern matching? I found [clojure.core.match](https://github.com/clojure/core.match) but I've never felt motivated enough to try it, nor have I seen it used. I have no idea about the state of pattern matching in other Lisps, but I assume it's a similar situation. `cond` gets you most of the way there I suppose?

No tuple type? Vectors are tuples. "What if one of the vectors is of a different length?" I gasped as I clutched my pearl monads.

I was sure I'd miss discriminated union types. There seemed like no practical stand-in for those.

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

Keywords don't have to be _declared_ anywhere before use; they're just literals. You can reference a namespaced keyword from a namespace without it ever being written in that namespace. Maybe the namespace doesn't even need to exist, I don't know. I've read/heard the Clojure designers encourage the use of namespaced keywords, but I rarely encounter them outside clojure.spec definitions.

## Destructuring

I used destructuring in F# most often in pattern matching clauses. In Clojure I'm finding almost all destructuring I do involves getting values from maps. The map destructuring syntax is powerful and a bit confusing at first although it seems so simple now. It even allows you to specify default values for keys that aren't present in the map, all in one line:
```clojure
{:keys [foo] :or {foo 1} :as m}
```
`foo` will have the value of `:foo` from the map, or `1` if the map doesn't have a `:foo` key, and `m` is bound to the entire map. This makes getting values from maps very easy and terse.

Another neat feature is positional destructuring, commonly used with vectors. In addition to the traditional _head and tail_ destructuring of sequences, you have more options with the `&` syntax.

```clojure
[x & xs] [1 2 3]
```
Here, `x` is `1` and `xs` is `[2 3]`. But you can also do this:
```clojure
[x & [next]] [1 2 3]
```
Here, `x` is `1` and `next` is `2`.

You can also destructure more than one binding before the _tail_ like so:
```clojure
[x y & zs] [1 2 3]
```
Here, `x` is `1`, `y` is `2`, and `zs` is `[3]`.

In the "anything goes" spirit, positional destructuring doesn't throw exceptions or anything on failure:
```clojure
[x y z] [1 2]
```
Here, `z` will simply be nil as there is no third element. The same goes for map destructuring i.e. you just get nil back for keys that aren't in the map.

All this destructuring is also used in function argument signatures! This allows for very flexible function signatures. This can also get a lot of destructuring "out of the way" of the function's body---a nice separation of concerns. In some sense, you can imagine all Clojure function argument bindings as a destructured vector. Destructurings can be nested but it becomes unreadable after 2-3 levels.

## Data

Clojure and F# have similar data structures but working with them in Clojure feels lower-ceremony. The literal syntax is nice and terse and all the basic data structures can be conveniently composed and transformed.

Maps would seem to be Clojure's commonplace record type, even though it has `defrecord` which seems [somewhat deprecated](https://groups.google.com/d/msg/clojure/pnDl4OgzqBM/zjDioSsxkvEJ). Something that bugged me about this initially was there were no guarantees that a _record_ (map) would contain what you expected, or that it wouldn't contain a bunch of stuff you _didn't_ expect! I came to realize this doesn't matter as often as I thought.

### clojure.spec

I believe there is a drawback to the ambiguities of dealing with data in Clojure, and one that clojure.spec can address: sometimes you really do need a rigid _specification_ of data. With clojure.spec you can _document_ the shape of data (in a machine-readable format), then use that to _verify_ properties of functions of that data to guarantee correct usage throughout. I've even used clojure.spec to generate API documentation and sample code.

I can't talk about data in Clojure without talking about [clojure.spec](https://clojure.org/about/spec)! It solves many problems but what interested me at first was using it as a sort of opt-in _gradual typing_ system for my Clojure code. I could write specs for only the data structures and functions I wanted.

I could write incredibly flexible specs for data and just as easily for the functions that operate on the data. With function specs, I could assert the inputs and outputs to specific functions at runtime; not quite as reassuring as compile-time but better than nothing!

I hope clojure.spec adoption continues and this kind of opt-in type checking becomes more prevalent. I think there's a very sweet spot to occupy between _everything must be typed_ and total dynamic ambiguity. I think the new Clojure JDBC library shipping with specs is a good sign of what's to come.

### Code as Data

Whatever superlative or diss you might use to describe a Lisp's syntax or lack thereof, it makes one thing evident: your code is itself just data, and data can be manipulated. A function at rest is just a list containing other lists and data structures. _Code as data_ was one of the profound enlightenment moments I've had as a programmer, and one that can't be realized in most popular languages.

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
This is nice in when you want to define some "rules" in one place and it use them from many, or maybe it's loaded from a database or configuration file.

### Nil

Some functional languages go to lengths to cordon null/nil values. Clojure embraces nil (in most cases). I remember thinking "there should be an Option type" a lot in my first days. Now I just embrace the nil. I rarely find myself needing to write defensive nil-check code.

I do miss [nullable comparison operators](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/symbol-and-operator-reference/nullable-operators), but this is just a matter of defining your own functions to do the same. Sometimes I wish clojure.string functions would propagate nils instead of throwing exceptions, but I can imagine at least one reason they wouldn't: they should take and return _strings_ and nil is not a string?

There are many nil-friendly core functions, but some nil-adverse functions too. It's not something I have to worry about too much, but a nil-propagation bug can be hard to debug when it happens.

## Arity

Clojure has a nice compact way to define multiple function arities: a single function symbol form with each arglist + implementation in parentheses:
```clojure
(defn foo
 ([one]
  (println one))
 ([one two]
  (println one two)))
```
And you do get some compile-time safety checks on function arity.

The really neat-o thing with Clojure function arities is the ability to define functions of arbitrary arity. In these cases the numerous arguments are bound to a collection: either a vector or a map, and yes a map can be thought of as a collection of tuple vectors. A basic example of this is the addition function:
```clojure
(+ x y & more)
```
Here, `more` can be any number of additional arguments. This ties into another concept that was new to me: _applying_ a sequence of arguments to a function.
```clojure
(apply + [1 2 3 4 5 6 7 8 9])
=> 45
```

Multi-arity functions are also how Clojure deals with named optional arguments. Both these function calls use the same function definition:
```clojure
(foo 123)
(foo 123 :append true :path "~/.sup")
```
The additional "arguments" are actually treated as a map and I've seen this style referred to as _kwargs_. It's a form of destructuring in the function signature like this:
```clojure
[num & {:keys [append path]}]
```

## REPL-driven Development

I didn't use the F# REPL that much for interactive _development_ as much as I did for interactive execution. The Clojure REPL experience is far more ingrained into my development workflow, and I like it. I feel I'm more of an inhabitant of the program as I write it rather than an observant author. I also get more immediate feedback about the runtime behavior of code, which can be hard to reason about in functional languages.

I think there's another reason for the REPL being so integral to the Clojure development experience: discovery through interaction. Clojure just doesn't have the information I would normally get via type definitions (and IntelliSense/auto-complete). In Clojure it's more natural to get that information by playing with the code in hand. Doc strings help but they aren't schematics. Again, another area where clojure.spec may change the landscape.

I haven't tried this, but I've heard you can be so bold as to have a deployed application host a network REPL which can be used to remotely inspect (and monkey with) the running application's code and state. 😮

## Invariants

One of my favorite things about more strongly-typed functional languages is the ability to _make illegal states unrepresentable._ In Clojure it's very hard to make illegal states unrepresentable, in fact I'd say it's just not worth the effort.

Clojure feels very open and permissive when it comes to data and passing it around. The thought that a caller could pass _anything_ as an argument was unsettling at first. Clojure does allow you to define pre- and post-condition metadata on functions for function call assertions, but they don't seem common in practice and they only work at _runtime_. clojure.spec allows much more powerful assertions, but again only at runtime.

As I embraced this "anything goes" approach, the Clojure lifestyle started to make a bit more sense. Clojure codebases are driven more by convention than contract. In F# I mostly thought about problems in terms of the types I'd use to model them---defining a contract. If my program compiled I could be fairly certain it'd do at least something like what I intended. This meant putting a lot of upfront thought into the _types_ I'd need to model the problem.

In Clojure I find myself doing the opposite---just diving into the REPL and immediately writing code that operates on data.

## Parenthetical Gestalt!

It's the carefully and conservatively curated confluence (alliteration achievement unlocked) of all these design decisions and philosophies, _especially_ those that were hard for me to embrace at first, that make Clojure an enjoyable programming experience for me. I pre-judged Clojure for being a dynamic language because I'd never enjoyed working in a dynamic language before. As I look back on all the things I thought I'd miss from a strongly typed language, I can easily say I've faired just as well without them.

_There were a bunch of other cool Clojure tidbits I wanted to mention but this post was getting too long!_
