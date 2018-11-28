---
layout: post
title:  "GraalVM Polyglot with Clojure and JavaScript"
date:   2018-11-26 00:00:00
tags:   clojure graal graalvm polyglot javascript interop
---

GraalVM enables interesting new interop scenarios between its hosted languages.
This post demonstrates some polyglot interop between Clojure as a host language
and JavaScript as the hosted language.

You'll need to run this Clojure code on a GraalVM SDK, and we'll want
to import some `org.graalvm.polyglot` types:
```clojure
(ns polydact.core
  (:import (org.graalvm.polyglot Context Value)
           (org.graalvm.polyglot.proxy ProxyArray ProxyExecutable ProxyObject)))
```

Now instantiate a polyglot GraalVM `Context` for the JavaScript language, and a helper
function for convenient evaluation:
```clojure
(def context (.build (Context/newBuilder (into-array ["js"]))))
(defn eval-js [code]
  (.eval ^Context context "js" code))
```
The result of `eval-js` will be some [GraalVM polyglot `Value`](http://www.graalvm.org/sdk/javadoc/org/graalvm/polyglot/Value.html), which has an
`.as` method for casting values to particular types.
That's all you need to start evaluating JavaScript from within Clojure:
```clojure
(.as (eval-js "Number.MAX_VALUE") Object)
=> 1.7976931348623157E308
(type *1)
=> java.lang.Double
```
That's a trivial scalar example, but how are more complex data structures
translated between language contexts? This is where GraalVM's polyglot API comes
into play, and there are multiple possible behaviors. Consider a JavaScript
array with an empty map in it:
```clojure
(.as (eval-js "[{}]") Object)
=> {"0" {}} ;; an array becomes a map with indices as keys
(.as (eval-js "[{}]") java.util.List)
=> ({}) ;; a polyglot list with polyglot map inside
```
If we invoke the `.as(Object.class)` overload, Graal uses a set of rules (see `.as` docstring)
to infer the desired result type, but it may not return what you want in some cases.
If we invoke `.as(List.class)` we get a different (and perhaps more familiar) translation,
but still not something we can easily work with in Clojure directly.

## Idiomatic Coercion

We can write some Clojure to help marshal a polyglot `Value` to a sensible Clojure (or Java) value:
```clojure
(defn value->clj
  "Returns a Clojure (or Java) value for given polyglot Value if possible,
   otherwise throws."
  [^Value v]
  (cond
    (.isNull v) nil
    (.isHostObject v) (.asHostObject v)
    (.isBoolean v) (.asBoolean v)
    (.isString v) (.asString v)
    (.isNumber v) (.as v Number)
    (.canExecute v) (reify-ifn v)
    (.hasArrayElements v) (into []
                                (for [i (range (.getArraySize v))]
                                  (value->clj (.getArrayElement v i))))
    (.hasMembers v) (into {}
                          (for [k (.getMemberKeys v)]
                            [k (value->clj (.getMember v k))]))
    :else (throw (Exception. "Unsupported value"))))
```
<p style='font-size:0.6em'>
<em>
A couple things to note: 1) This is not a complete or well-tested implementation, and 2) it handles all my little toy examples for this article.
</em>
</p>

Sparing some details, the `Value` class is a container/boxed value for some polyglot value (or a pointer),
and it has methods for inspecting what type of value it contains. Most of the `cond` clauses should
be fairly self-explanatory. The most interesting ones have to do with `.canExecute` executable values,
`.hasArrayElements` arrays, and `.hasMembers` associative structures.

### Executables

The definition of `reify-ifn` is a macro that reifies `IFn` for the given `Value`.
This is only a macro so that I can easily emit all the arities of `invoke`:
```clojure
(defn- execute [^Value execable & args] ;; just a little sugar
  (.execute execable (object-array args)))

(defmacro ^:private reify-ifn
  "Convenience macro for reifying IFn for executable polyglot Values."
  [v]
  (let [invoke-arity
        (fn [n]
          (let [args (map #(symbol (str "arg" (inc %))) (range n))]
            (if (seq args)
              `(~'invoke [this# ~@args] (value->clj (execute ~v ~@args)))
              `(~'invoke [this#] (value->clj (execute ~v))))))]
    `(reify IFn
       ~@(map invoke-arity (range 22))
       (~'applyTo [this# args#] (value->clj (apply execute ~v args#))))))
```
<p style='font-size:0.6em'>
<em>
A couple things to note: 1) This is not a complete or well-tested implementation, and 2) it handles all my little toy examples for this article.
</em>
</p>

Now we can treat executable polyglot values like typical Clojure functions!

### Arrays

The `.hasArrayElements` case is pretty plain: convert it to a vector and recursively convert
any contained values.

### Associatives

The `.hasMembers` case is not so clear-cut. There are several different interpretations
of what a member can be for a polyglot value. For example, for a Java object each member
would be a field or class member. For a JSON object, each member would be a string key.

This example uses the simplest interpretation and treats everything that has members as a Clojure map.
To paraphrase someone else: now it's data, you've already won!

## Polyglottin'

Equipped with just `value->clj` and a couple helpers, let's play the ployglottery...

```clojure
(def js->clj (comp value->clj eval-js)) ;; fn composition convenience

(js->clj "[{}]")
=> [{}]

(js->clj "false")
=> false

(js->clj "3 / 3.33")
=> 0.9009009009009009

(js->clj "123123123123123123123123123123123")
=> 1.2312312312312312E32

(js->clj "m = {foo: 1, bar: '2', baz: {0: false}};")
=> {"foo" 1, "bar" "2", "baz" {"0" false}}

(js->clj "1 + '1'")
=> "11" ;; a classic

(js->clj "!!\"false\" == !!\"true\"")
=> true ;; we've proven false!

(js->clj "['foo', 10, 2].sort()")
=> [10 2 "foo"] ;; JS sort so funny!
```
Nothing surprising there, just that we're evaluating some _JavaScript_
from a Clojure REPL and getting back _Clojure_ values.

Note that we're sharing a single `Context` for these evaluations, and contexts
are stateful and mutable:
```clojure
(eval-js "var foo = 0xFFFF")
(eval-js "console.log(foo);")
=> #object[org.graalvm.polyglot.Value 0x3f9d2028 "undefined"]
;65535
```

## Functional Programming

What if you could write your functions in JavaScript, which I hear is getting
better all the time, and call them from Clojure? Welcome to the next level!

```clojure
(def doubler (js->clj "(n) => { return n * 2; }"))
(doubler 2)
=> 4 ;; checks out!
```

How about a beautiful, memoizing factorial function that returns a JSON
object containing the `factorial` function and the backing memo array:
```clojure
(def factorial
  (eval-js "
    var m = [];
    function factorial (n) {
      if (n == 0 || n == 1) return 1;
      if (m[n] > 0) return m[n];
      return m[n] = factorial(n - 1) * n;
    }
    x = {fn: factorial, memos: m};"))

((get (value->clj factorial) "fn") 12)
=> 479001600 ;; JS happens to be right here
(get (value->clj factorial) "memos")
=> [nil nil 2 6 24 120 720 5040 40320 362880 3628800 39916800 479001600]

((get (value->clj factorial) "fn") 24)
=> 6.204484017332394E23
(get (value->clj factorial) "memos")
;; the result of this may surprise you! (it now contains many more values)
```

We can use `ProxyArray` to pass collections to JavaScript functions
so they can be treated as native JavaScript arrays.
```clojure
(def js-aset
  (js->clj "(arr, idx, val) => { arr[idx] = val; return arr; }"))
(js-aset (ProxyArray/fromArray (object-array [1 2 3])) 1 nil)
=> [1 nil 3] ;; and we get a mutated vector back
```

JavaScript sports exotic, modern features like variadic functions, and we can treat
them just like their Clojure counterparts. We can even see Clojure-specific values
like keywords (_boxed_ in polyglot values) passed through the foreign JavaScript
function unharmed.
```clojure
(def variadic-fn
  (js->clj "(x, y, ...z) => { return [x, y, z]; }"))
(apply variadic-fn :foo :bar (range 3))
=> [:foo :bar [0 1 2]]
```

Have you ever tried to `sort` a wildly heterogenous collection in Clojure?
```clojure
(sort [{:b nil} \a 1 "a" "A" #{\a} :foo  -1 0 {:a nil} "bar"])
CompilerException java.lang.ClassCastException:
clojure.lang.PersistentArrayMap cannot be cast to java.lang.Character
```
I thought this was a _dynamic_ language! When you just need to get the job done, JavaScript will oblige:
```clojure
(def js-sort
  (js->clj "(...vs) => { return vs.sort(); }"))
(apply js-sort [{:b nil} \a 1 "a" "A" #{\a} :foo  -1 0 {:a nil} "bar"])
=> [-1 0 1 "A" #{\a} :foo {:a nil} {:b nil} "a" "a" "bar"]
```
It's easy to miss in the above example, but notice again the keyword `:foo` and a _set_
in the input collection we passed to the JavaScript `.sort()` function, and
the sorted output contains said keyword and set sorted by some likely inscrutable JavaScript
sorting logic. It "just works" despite the fact JavaScript has no concept of keywords.

Use the JSON serde functions that started it all:
```clojure
(def ->json
  (js->clj "(x) => { return JSON.stringify(x); }"))
(->json [1 2 3])
=> "[1,2,3]"

;; Proxy hash map to be treated as a JSON object in this case
(->json (ProxyObject/fromMap {"foo" 1, "bar" nil}))
=> "{\"foo\":1,\"bar\":null}"

(def json->
  (js->clj "(x) => { return JSON.parse(x); }"))
;; take some round trips
(json-> (->json [1 2 3]))
=> [1 2 3]
(json-> (->json (ProxyObject/fromMap {"foo" 1})))
=> {"foo" 1}

;; access JSON object members naturally
(def json-object
  (js->clj "(m) => { return m.foo + m.foo; }"))
(json-object (ProxyObject/fromMap {"foo" 1}))
=> 2
```

## Clojurvascriptception

We've made JavaScript functions double as Clojure functions, now let's do the opposite.
This function provides an instance of GraalVM's `ProxyExecutable` that implements its sole
`.execute` method for a given Clojure function (or really anything `apply` works on):
```clojure
(defn proxy-fn
  "Returns a ProxyExecutable instance for given function, allowing it to be
   invoked from polyglot contexts."
  [f]
  (reify ProxyExecutable
    (execute [_this args]
      (apply f (map value->clj args)))))
```
That's all we need to be able to pass executable code from Clojure into our polyglot context.

In this example we return a JavaScript function that closes over a nested map and takes another
function to invoke with the map — maybe the term _callback_ would be more apt in JavaScript parlance.
```clojure
(def clj-lambda
  (js->clj "
  m = {foo: [1, 2, 3],
       bar: {
         baz: ['a', 'z']
       }};
  (fn) => { return fn(m); }
  "))

(clj-lambda
 (proxy-fn #(clojure.walk/prewalk
             (fn [v] (if (and (vector? v)
                              (not (map-entry? v)))
                       (vec (reverse v))
                       v))
             %)))
=> {"foo" [3 2 1], "bar" {"baz" ["z" "a"]}}
```
That's Clojure using GraalVM to host and evaluate JavaScript, passing a Clojure
function to a JavaScript lambda that invokes the Clojure function, which uses `prewalk`
to reverse all the arrays in the nested JSON object, then getting back an idiomatic Clojure
map value.

We can lean on JavaScript's `.reduce()` and use it just like Clojure's `reduce` with a lazy sequence:
```clojure
(def js-reduce
  (let [reduce (js->clj "(f, coll) => { return coll.reduce(f); }")
        reduce-init (js->clj "(f, coll, init) => { return coll.reduce(f, init); }")]
    (fn
      ([f coll] (reduce f coll))
      ([f init coll] (reduce-init f coll init)))))
(js-reduce + (range 10))
=> 45
(js-reduce + -5.5 (range 10)
=> 39.5
```
We can pass in a Clojure reducing function, that internally uses the JavaScript `doubler`
function defined above, and build up a Clojure map within JavaScript's `.reduce()`:
```clojure
(js-reduce (fn [acc elem]
             (assoc acc (keyword (str elem)) (doubler elem)))
           {}
           (range 5))
=> {:0 0, :1 2, :2 4, :3 6, :4 8}
```

Some bad news about passing Clojure's lazy sequences to JavaScript: they'll be realized _before_
being evaluated in JavaScript.
Here we can see the Clojure `prn` statements all execute before the JavaScript logs the first value:
```clojure
(def log-coll
  (js->clj "(coll) => { for (i in coll) console.log(coll[i]); }"))
(log-coll (repeatedly 3 #(do (prn 'sleeping)
                             (Thread/sleep 100)
                             (rand)))
sleeping
sleeping
sleeping
0.8793736393816931
0.9610193480723516
0.05981091977823816
=> nil

(log-coll (range)) ;; infinite seq will never complete
```

## Closing Remarks

I have no practical use for any of this at the moment, but I find this whole mixed-language runtime
experiment very exciting!

While these examples only used JavaScript, the same things can be done with several other languages
on GraalVM, like Ruby, Python, R, LLVM-based languages, etc.
See [this page](http://www.graalvm.org/docs/graalvm-as-a-platform/embed/) for more information.

Here's a [gist](https://gist.github.com/taylorwood/bb3ebfec5d5de3cccc867a9eba216c18) with all the code
from this experiment.
