---
layout: post
title:  "Clojure.spec Beginner's FAQ"
date:   2018-10-15 00:00:00
tags:   clojure spec intro beginner faq guide manual tips
---

[Clojure.spec](https://clojure.org/about/spec) has been available (in alpha) for some time, and there are great [talks](https://www.youtube.com/watch?v=oyLBGkS5ICk) and resources like the [rationale](https://clojure.org/about/spec) and [official guide](https://clojure.org/guides/spec). This post supplements those with some common questions and issues I've seen.

These are mostly things I've experienced myself or seen on [Stack Overflow](https://stackoverflow.com/questions/tagged/clojure.spec), [Clojurians Slack](http://clojurians.net), [r/Clojure](https://www.reddit.com/r/Clojure/) or [Twitter](https://twitter.com/search?q=clojure%20spec).
The community is extremely helpful so reach out if you have questions --- it's not unusual to get an answer straight from a Clojure maintainer.
_And while this post follows question & answer format, my answers are extremely non-authoritative. Please [report any issues](https://github.com/taylorwood/taylorwood.github.io/issues/new) you find!_

## Usage

**Q: How should I integrate clojure.spec into my project?**

**A: However you like!**

Clojure.spec usage is totally à la carte. You can use only the features you want to whatever extent you want.
This is great for consumers, but a lack of _prescribed_ patterns doesn't help newcomers wondering how to "best" leverage spec in their work.

Here are some things I've used spec for:
- Validating inputs to service endpoints, and using spec metadata to return helpful details for invalid inputs.
- Generating sample inputs to service endpoints for API documentation.
- Writing specs for important functions, and instrumenting them during development/testing as a kind of run-time type checker.
- Using specs to get corresponding test.check generators "for free", using the generated data for other purposes.
- Using a `s/or` spec as a kind of discriminated union to dispatch based on different types of inputs.

## Organization

**Q: Should I put specs in the same namespace as my code, or in a separate namespace?**

**A: It's up to you.**

You may find having specs alongside relevant code is helpful.
You can just as easily put specs for _data_ in separate namespaces, and specs for _functions_ next to their `defn` counterparts --- and you can put the spec before or after the `defn`.

One consideration is whether you're using qualified keywords in `s/keys` map specs.
I think a potential benefit of having data specs in the same namespace as your regular code is that it's natural to inherit that namespace for namespaced map specs:
```clojure
(in-ns 'customer)
(s/def ::id int?)
(s/def ::name string?)
(s/def ::contact (s/keys :req [::id ::name]))
(defn print [{::keys [id name]}] (prn id name))
(print #:customer{:id 1 :name "Jane" :some.other/id "FhI-1"})
```

## Instrumentation & Testing

`instrument` can be used to assert arguments to spec'd function invocations.
See [my post on function specs]({% post_url 2017-10-15-fspec %}) for more detailed examples.
The 0-arity `instrument` will instrument every loaded, instrumentable function.
There's a cost associated with instrumentation, and you may want `instrument` to only affect particular functions:
```clojure
(clojure.spec.test.alpha/instrument [`foo `bar])
```
And there's an `unstrument` function for removing instrumentation.

**Q: When and where should I call `instrument`?**

**A: It depends.**

Your instrumentable functions must be loaded _before_ you can `instrument` them.
If you have per-profile entry points to your program e.g. `-main`, `-dev-main`, you might choose to call `instrument` in the dev entry point.
If your project uses Leiningen, you could use `:injections` to call `instrument` on a per-profile basis:
```clojure
:injections [(require 'lib.core) ;; all instrumented fns should be loaded here
             (require 'clojure.spec.test.alpha)
             (clojure.spec.test.alpha/instrument)]
```
Or you may only want to `instrument` functions during test runs, maybe as a clojure.test fixture or in the namespace itself.

**Q: Should I use `instrument` in production builds?**

**A: Probably not.**

From the guide:
>Instrumentation is likely to be useful at both development time and during testing to discover errors in calling code. It is not recommended to use instrumentation in production due to the overhead involved with checking args specs.

**Q: Why doesn't `instrument` check my function return value?**

**A: It's not meant to.**

This is a common point of confusion. If you `instrument` a function spec'd with `fdef` or `fspec`, only its `:args` spec will be checked during each invocation.
Why is that? I think one rationale might be that functional programs are generally a composition of functions where each output becomes an input to another function.
If you have spec'd the `:args` and `:ret` of related functions `(f (g x))`, you'd be redundantly checking the same specs for the `:ret` of `g` and `:args` of `f`. For larger examples this could become very costly.

**Q: So why do `fdef`/`fspec` accept `:ret` and `:fn` specs too?**

**A: They're used to `check` the function.**

`check`ing a function involves generating many random inputs (from the `:args` spec) for the function, invoking the function with each input, and checking that the return value conforms to the `:ret` spec, and if `:fn` spec is defined that the relationship between the input and output values are satisfied. This is commonly called generative or property-based testing and spec relies on test.check for this.

You can use `check` as part of a test suite, for example with clojure.test:
```clojure
(deftest foo-test
  (is (= 1 (-> (st/check `foo)
               (st/summarize-results)
               :check-passed))))
```

## Sequences & Regex Specs

**Q: What's this odd nesting behavior with regex specs?**

**A: Nested regex specs compose to describe a single sequence.**

Or as stated in the [rationale](https://clojure.org/about/spec#_sequences):
> These nest arbitrarily to form complex expressions.

I think this is much easier to understand through examples, which can be found [in the official guide](https://clojure.org/guides/spec#_sequences).

It's easy to forget the `s/&` spec which is useful for adding additional predicates to regex specs, where using `s/and` would destroy the regex nesting behavior.
Consider these two conforming examples where the only difference is `s/&` vs. `s/and`:
```clojure
(s/conform
  (s/+ (s/alt :n number?
              :s (s/and (s/+ string?)
                        #(every? (complement empty?) %))))
  [1 ["x" "a"] 2 ["y"] ["z"]])

(s/conform
  (s/+ (s/alt :n number?
              :s (s/& (s/+ string?)
                      #(every? (complement empty?) %))))
  [1 "x" "a" 2 "y" "z"])
```

**Q: How can I escape the regex spec nesting behavior?**

**A: Wrap the regex spec with `s/spec`.**

This is also described in the official guide above, but here's another example:
```clojure
(s/conform
  (s/+ (s/alt :n number? :s (s/spec (s/* string?))))
  [1 2 3 ["x" "y" "z"]])
=> [[:n 1] [:n 2] [:n 3] [:s ["x" "y" "z"]]]
```
But this example might be better specified with `s/coll-of` or `s/every`:
```clojure
(s/+ (s/alt :n number? :s (s/coll-of string?)))
(s/+ (s/alt :n number? :s (s/every string?)))
```

## Map Specs

**Q: How can I assign different specs to the same key in different maps?**

**A: That's not supported (for qualified keys).**

Spec encourages use of qualified map keys. If you control the shape of your map you should consider using qualified keys:
```clojure
{:customer/name "Frieda" :company/name "Acme"}
```

Of course you may not have this luxury when working with external data sources where the same key may have different meanings at different paths:
```clojure
{:name "Frieda" :company {:name "Acme"}}
```
In this case there's a workaround if you're using unqualified keys in your map specs. You can use qualified keys to _name_ your specs, but create `s/keys` specs with unqualified keys e.g. `:req-un` and `:opt-un`:
```clojure
(s/def :customer/name string?)
(s/def :company/name (s/nilable string?))
(s/def ::company (s/keys :req-un [:company/name]))
(s/def ::customer (s/keys :req-un [:customer/name ::company]))
(s/valid? ::customer {:name "Taylor" :company {:name nil}}) => true
```

**Q: How can I make a `s/keys` spec that disallows extra keys?**

**A: You should reconsider that.**

Map specs are meant to be _open_ instead of _closed_. Adding data to a map spec should not make the map invalid.
There are some cases where you might really want this, and of course [it's possible](https://github.com/gfredericks/schpec/blob/b2d80cff29861925e7e3407ef3e5de25a90fa7cc/src/com/gfredericks/schpec.clj#L13).

## Generation

**Q: Why does generation fail with _Couldn't satisfy such-that predicate after 100 tries_?**

**A: It might be the default generator for a `s/and` spec.**

Generators are created from specs automagically, but for `s/and` specs the generator is based solely on the _first_ spec inside the `s/and`.
This spec conforms string palindromes:
```clojure
(s/def ::palindrome
  (s/and string? #(= (clojure.string/reverse %) %)))
```
But you may get an error message when trying to exercise its generator (or possibly indirectly if the spec is involved in a `check`'d function):
```clojure
(gen/sample (s/gen ::palindrome))
ExceptionInfo Couldn't satisfy such-that predicate after 100 tries.  clojure.core/ex-info (core.clj:4739)
```
This is because the generator for `::palindrome` is actually a generator for `(s/gen string?)`,
which will just generate random strings that are _very unlikely_ to be palindromes, and after 100 random tries it gives up.
In these cases you can provide your own generator with `s/with-gen`:
```clojure
(s/def ::palindrome
  (s/with-gen
    (s/and string? #(= (clojure.string/reverse %) %))
    ;; use fmap to create palindromes of the generated strings
    #(gen/fmap (fn [s] (apply str s (rest (reverse s))))
               (s/gen string?))))
(gen/sample (s/gen ::palindrome))
=> ("" "" "e" "PA6AP" "OUdTdUO" "k" "N" "0" "1T353T1" "D4V4D")
```
Some spec functions also take an optional _overrides_ map of custom generators without needing to associate them with the spec definitions directly.

**Q: How can I generate data with parent/child relationships?**

**A: Use test.check functions like `fmap`, `bind`, or `let` macro.**

As in the previous example you can use `fmap` to create a new generator that alters the results of a generator, so for recursive structures you can make any modifications there.
For more complex scenarios --- perhaps involving multiple generators --- test.check provides a `let` macro that's syntactically similar to clojure.core `let`.
(Note that clojure.spec.gen.alpha only aliases _some_ of test.check's functionality; you'll need to require test.check namespaces directly for its `let` macro.)
The RHS of the bindings are generators and the LHS names are bound to the generated values, and it all expands to `fmap` and `bind` calls.

Consider an example of a recursive map spec:
```clojure
(s/def ::id uuid?)
(s/def ::children (s/* ::my-map))
(s/def ::parent-id ::id)
(s/def ::my-map
  (s/keys :req-un [::id]
          :opt-un [::parent-id ::children]))
(gen/sample (s/gen ::my-map))
```
We need a custom generator to ensure the `:parent-id` for child maps is accurate, using `fmap` again:
```clojure
(defn set-parent-ids [m]
  (clojure.walk/prewalk
    (fn [v]
      (if (map? v)
        (update v
          :children #(map (fn [c] (assoc c :parent-id (:id v))) %))
        v))
    m))
(gen/sample
  (gen/fmap set-parent-ids (s/gen ::my-map)))
```
Also see test.check's `recursive-gen` for another approach.

Another example might be non-recursive relationships between values, like arguments to a function.
Consider writing a function spec for `clojure.core/subs`:
```clojure
(s/def ::subs-args (s/cat :str string? :i int?))
(s/fdef clojure.core/subs :args ::subs-args)
(s/exercise-fn `subs)
StringIndexOutOfBoundsException String index out of range: -1  java.lang.String.substring (String.java:1927)
```
This is because the generators for the strings and integer indices are independent and unrelated:
```clojure
(gen/sample (s/gen (s/cat :str string? :i int?)))
=> (("" 0) ("" 0) ("" -1) ("C" 3) ("c9gD" 1) ("" 6) ("cfo2s7" -7) ("3fRj30" -2) ("W" 0) ("cEzS" -15))
```
If we want to ensure that the index argument refers to a position in the string input, we can use a custom generator:
```clojure
(s/def ::subs-args
  (s/with-gen
    (s/cat :str string? :i int?)
    #(gen/let [str (s/gen string?)
               index (gen/large-integer* {:min 0 :max (count str)})]
       [str index])))
(st/summarize-results (st/check `subs))
=> {:total 1, :check-passed 1}
```

**Q: Why is my recursive/sequential spec slow to generate or check?**

**A: The recursion may be too deep, or sequence spec generators may be unbounded.**

Spec defines a dynamic binding `*recursion-limit*` that puts a "soft" limit on recursive generation depth.
You may want to decrease this limit in cases where generated structures are unwieldy.

The other type of sizing/growth to be concerned about relates to sequences generated by specs like `s/every`, `s/coll-of`.
The `:gen-max` option will limit the length of generated sequences:
```clojure
(gen/sample (s/gen (s/coll-of string? :gen-max 2)))
=> (["" ""] [] ["1D" ""] ["I"] [] ["Ne4" "y6i"] ["93" "oe"] ["4wUue7"] [] [])
```

## Specs as Data

The current version of spec doesn't make it very easy to create or inspect specs programmatically.
This is reportedly being worked on for a future release.

**Q: How can I get the keys from a `s/keys` map spec?**

**A: Use `s/form` or `s/describe` on the spec.**

`s/form` (and its abbreviated sibling `s/describe`) will return the original form of the spec:
```clojure
(s/def ::my-map
  (s/keys :req-un [::id]
          :opt-un [::parent-id ::children]))
(->> (s/describe ::my-map) ;; get abbrev. form of original spec
     (rest)                ;; drop `keys` symbol
     (apply hash-map))     ;; put kwargs into map
=> {:req-un [:sandbox/id]
    :opt-un [:sandbox/parent-id :sandbox/children]}
```

**Q: How can I get the `:args`, `:ret`, or `:fn` specs from a function spec?**

**A: Same as the previous answer, for any type of spec.**

And if for some reason you also needed to recreate one of those specs, you could use `eval`:
```clojure
(s/fdef foo :args (s/tuple number?) :ret number?)
(eval (->> (s/form `foo) ;; get non-abbrev. form
           (rest)
           (apply hash-map)
           :args))  ;; get :args spec
(gen/sample (s/gen *1))
=> ([2.0] [-1.5] [0] [-2] [-1.5] [-1.5] [14] [1] [##-Inf] [-5.875])
```

## Off-label Usage

**Q: How can I use spec to parse strings?**

**A: Spec is not intended for string parsing.**

Specs can be used as parsers in that specs can describe a syntax, and conforming valid inputs produces tagged outputs that can be treated as a syntax tree, so... why not use that on strings of characters?
You'll have an easier time using a purpose-built parser for string inputs. Spec's regex specs are also more limited than typical regular expression libraries.

**Q: I can use `s/conformer` to transform data with specs. Is that a good idea?**

**A: Not really.**

[Cognitect's Alex Miller says](https://stackoverflow.com/a/41679859):
>spec is not designed to provide arbitrary data transformation (Clojure's core library can be used for that).
>It is possible to achieve this using s/conformer but it is not recommended to use that feature for arbitrary transformations like this (it is better suited to things like building custom specs).

There's also a [JIRA with discussion](https://dev.clojure.org/jira/browse/CLJ-2116) on the topic. The official advice is to do this separately from spec, and there's a [library to assist with that](https://github.com/wilkerlucio/spec-coerce).

I think some simple coercion isn't terrible in limited, internal use cases e.g. conforming date strings to date values at your API boundary.
