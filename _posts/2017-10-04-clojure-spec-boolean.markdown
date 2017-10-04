---
layout: post
title:  "Boolean Infix DSL Validation with clojure.spec"
date:   2017-10-04 12:00:00
tags:   clojure spec dsl infix
---

Say you have a problem. You're going to write some code to parse and validate logical boolean clauses as data. Maybe you're writing a DSL to allow users to express some rules for a rules engine, e.g. "date is today and junk is true". Wouldn't it be nice to be able to validate these logical declarations, perhaps by defining a **specification** _declaratively?!_

[Clojure.spec](https://clojure.org/guides/spec) gives us the tools to define a specification, evaluate arbitrary input against it, and (by way of [clojure/test.check](https://github.com/clojure/test.check)) even generate random sample data accordant with the specification.

# Simplifying Assumptions

Let's assume a simple case where each logical expression is just a keyword, and they can be joined by boolean operators (also represented as keywords) in _infix_ notation. We'll assume only _and_ and _or_ operators are allowed.
```clojure
[:x :and :y]
```
We can pretend `:x` and `:y` refer to some other logical expression for now.

Of course users want to group and nest these logical expressions.
```clojure
[:x :and :y [:a :or :b [:all :or :nothing]]]
```

_And of course we'll assume we're able to deserialize user input into this vector/keyword form._

# Declaring Specs

When we consider this problem, there are only two unique ingredients to our expressions:
1. Expressions
2. Boolean operators

Clojure.spec defines a simple language of easily-composed primitives to allow very rich specifications over data. We can start by defining a spec per ingredient, and let's start simple with the boolean operators.
```clojure
(def op-keys #{:and :or})
```
We can use sets as predicates here. Our _spec_ for operators is simply a set of keywords.

Those operators need operands. Here we'll `def`ine a _spec_ called `::expression`:
```clojure
(s/def ::expression
  (s/and keyword?
         #(not (contains? op-keys %))))
```
Specs are resolved by _name_ and they are named by namespaced keywords. The double-colon prefix is shorthand for whatever the current namespace is.

Our _expression_ is just a keyword but it can't be one of the operators, so we use spec's `and` to combine two logical predicates.

We have our two ingredients but we need a _recipe_ for how they can be combined.

```clojure
(s/def ::group
  (s/cat :head ::expression
         :tail (s/*
                 (s/cat :op     op-keys
                        :clause (s/or :expr  ::expression
                                      :group ::group)))))
```

This `::group` spec defines our overall expression as a con`cat`enation of elements. It must always have some _head_ element which conforms to our `::expression` spec above. Maybe just one thing must be true, and that's fine. 🤷‍♂️ After the head there can be any number `*` of additional expressions, which are _also_ concatenations of an operator and... you guessed it, more expressions, which may contain more expressions, which may contain more expressions. Very expressive.

Our specification is **recursive**---a grouping of expressions is itself an expression. Luckily, defining a recursive spec is trivial! On the last line we simply refer to the `::group` spec from within its own definition.

# Trial & Error

Whenever I'm writing a spec, I'm always playing with it in the REPL as I go. This should come as no surprise but I went through **many** (oft non-working) iterations before landing on the oh-so-elegant-and-plainly-correct spec above.

```clojure
(s/conform ::group [:x])
=> {:head :x}
```
Yep, checks out. Clojure.spec's `conform` function takes a _spec_ (either a namespaced keyword that refers to one or an inline spec) and some data to be _conformed_. A very interesting aspect of clojure.spec is that the process of _conforming_ input data to a spec can produce arbitrarily different output data. In this case, the `cat` and `or` functions will _tag_ our output as you can see above; we get a map back telling us where our `:head` is.

```clojure
(s/conform ::group [:x :y])
=> :clojure.spec.alpha/invalid
```
When the input data does not conform, we get back a special namespaced keyword. If we simply wanted a yes or no predicate we could use `s/valid?`.

Let's put it to some slightly tougher tests:

```clojure
(s/conform ::group [:x :and :y])
=> {:head :x, :tail [{:op :and, :clause [:expr :y]}]}
(s/conform ::group [:x :and :y :or]) ;; no good trailing operator
=> :clojure.spec.alpha/invalid
(s/conform ::group [:x :and :y :or :z])
=> {:head :x, :tail [{:op :and, :clause [:expr :y]} {:op :or, :clause [:expr :z]}]}
(s/conform ::group [:x :and :xx :and :yy :and [:y :or :z :or [:foo :and :bar]]])
=>
{:head :x,
 :tail [{:op :and, :clause [:expr :xx]}
        {:op :and, :clause [:expr :yy]}
        {:op :and,
         :clause [:group
                  {:head :y,
                   :tail [{:op :or, :clause [:expr :z]}
                          {:op :or, :clause [:group {:head :foo, :tail [{:op :and, :clause [:expr :bar]}]}]}]}]}]}
```

Seems like it works alright.

# Generation

Surely after you validate these input clauses you're going to want to _do something_ with them? What if you could conjure up a universe of test case inputs for that something? (**Spoiler:** you can.) Simply defining this spec gives us a nifty superpower. Using test.check we can randomly generate samples of our spec:
```clojure
 (sgen/sample (s/gen ::group))
 => ;; abbreviated output
 (:o/h?)
 (:E :and :T :or (:X :or (:?* :or :f1/a :and (:+ :and :s))) :and (:z- :or (:r/D :and (:+M) :or :+W/-0)))
 (:s*/PC :or :-P :or (:N-/_ :and :W/_-) :and (:!))
 (:J__/e1 :and :K :and (:K) :and :X7m.W/Dt :or (:W))
```
_Nice._

# Bonus Round

What if you needed to emit some sort of query language for these expressions? Let's transform these infix expressions into strings vaguely resembling Some Query Language! We'll use an acronym SQL, that sounds nice.

```clojure
(defn clause-str [clause] ;; it's too early for doc strings
  (walk/postwalk
    (fn [elem]
      (if (coll? elem)
        (format "(%s)" (cs/join " " (map name elem)))
        elem))
    clause))
```

Here we use [clojure.walk's `postwalk`](https://clojure.github.io/clojure/clojure.walk-api.html#clojure.walk/postwalk) to perform a depth-first traversal of our data strucure (a vector of keywords and/or vectors). The depth-first part is important because we want our transformed output to reflect the nested group structure of our data, which we accomplish here by simply parenthesizing the group strings. For the group elements we just take the `name` of each keyword joined by spaces.

Let's exercise `clause-str` with those randomly generated samples.

```clojure
(map clause-str (sgen/sample (s/gen ::group)))
=>
("(e)"
 "(u)"
 "(!8 or xQ and (b and ?! and (b)))"
 "(L1)"
 "(!M)"
 "(?0V)"
 "(+J or (v? or A and (+4 or (z+ or I or (d+))) and (uhd and ! or (i or (RS1)))) or *82 or SC and (B or _ or G+! or Lj or (_ or j and (O?* or +hK and (P) or Y8 or z and (p4+) and o-r) and (z8 or y*2 and n or (G-) or TFi and (FH)) or GSc or (ZF or (!-3)) and +?)))"
 "(OrT or (h-! or (c or (- and me and G) and (-6 and (G) and ?)) and M and (b or (+*+) and (c* and (n8-) or (v_)) and pKM or a7t or (l or o and (Fz) and (e7x) or (+9Y) and Cl or Mm) and (E or (-) or (_7G) and (M6q) or Uv and +m and (a*3))) or (!+ and ! or (+D? and (e) and (l) and (D)))) or (-K or (_* and p5 and (Wv and (b8+) or k! and W or R7? and l8V or oli) and p3p or (u or (w7P) or +Vi and n and x6 and (?) and (+_))) or n and !7v and rKQ) and t!3 or (c and (x2) or (_+ and (rE and (*2X) or (V) and C or (MW)) and (vx_ and (vZ3) and W5 or (+8?) or (G6)) or (X or d3A or J*9 and zS+) or (G_l or EFr) or (B-H and (T) or (!-C))) or Ne5) or Q and u)"
 "(-1 or o- or ?xE and -X8 or gY and w or (kD* and (hxj and (? or v or (N) and (_) or (z+0) and (+) and (rhc) or ?) or t+ and *) or T4 or d*))"
 "(?J and (A) or (c2 or c or - or (U or (kT and (q!) or (t) and (xF) and g! and -60 and d3H and (R6?)) and A5) and gW8))")
```

Or just call it with some bespoke inputs:
```clojure
(clause-str [[:x
              :or :y
              :or :z
              :or [:q :and :z]]
              :and :x
              :and :y
              :and :z
              :and [:x
                    :or :y
                    :or :z
                    :or [:q :and :z]]])
=> "((x or y or z or (q and z)) and x and y and z and (x or y or z or (q and z)))"
```

# Callers Behave

We can ensure `clause-str` is called with valid arguments using clojure.spec.test's `instrument`.

```clojure
(s/fdef clause-str                              ;; a spec for our function
        :args (s/cat :clause (s/spec ::group))) ;; it takes one argument and it better be a ::group
(stest/instrument `clause-str)                  ;; assert valid args on each call
(clause-str [:x :and :y])
=> "(x and y)"
```

An invalid call will now throw an exception with information:
```clojure
(clause-str [:x :and])
CompilerException clojure.lang.ExceptionInfo: Call to #'playground.test/clause-str did not conform to spec:
In: [0] val: () fails spec: :playground.test/group at: [:args :clause :tail :clause] predicate: (or :expr :playground.test/expression :group :playground.test/group)...
```

Instrumenting our function under test reveals a problem! Our exact input we `postwalk`'d above no longer works:
```clojure
(clause-str [[:x
              :or :y
              :or :z
              :or [:q :and :z]]
             :and :x
             :and :y
             :and :z
             :and [:x
                   :or :y
                   :or :z
                   :or [:q :and :z]]])
ExceptionInfo Call to #'playground.test/clause-str did not conform to spec:
In: [0 0] val: [:x :or :y :or :z :or [:q :and :z]] fails spec: :playground.test/expression at: [:args :clause :head] predicate: keyword?
```

What's the problem? Our `::group` spec is a concatenation expecting a `:head` element that conforms to `::expression`. In this input, our `:head` element is actually a vector/group of expressions, which doesn't conform to `::expression`. We can fix it by revising our `::expression` spec:
```clojure
(s/def ::expression
  (s/or :g ::group
        :v (s/and keyword?
                 #(not (contains? op-keys %)))))
```
Our `::expression` and `::group` specs are now mutually recursive. This might be a bad idea, but it works! Try generating samples of `::group` before and after this change to see the difference; the latter samples are much more complex.