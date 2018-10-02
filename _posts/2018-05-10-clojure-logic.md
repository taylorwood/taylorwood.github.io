---
layout: post
title:  "Logic programming in Clojure"
date:   2018-05-10 12:00:00
tags:   clojure logic core logic core.logic
---

This post covers some toy problems I've solved with Clojure's [core.logic](https://github.com/clojure/core.logic).

## Introduction

Logic and constraint-based programming interest me, which might explain why I like writing SQL queries so much.
The idea that you can describe a problem declaratively and have a program give you an exhaustive set of answers is a bit mindblowing.
For some problems, a "mechanical" solution might require much more code, and maybe much more complex code.
I love trying to think about solutions in this way, even if logic programming isn't always the most appropriate solution.

See [this primer](https://github.com/clojure/core.logic/wiki/A-Core.logic-Primer) for basics, but I'll say here the basic starting point for core.logic is usually the `run*` or `run` form, with binding(s) that you want to solve for:
```clojure
(run* [q])
```
That won't do anything by itself. We need to specify some goals/constraints on `q`:
```clojure
(run* [q]
  (== [1 2 3] [1 q 3]))
=> (2)
```
This is telling us, for those two vectors to _unify_, the only possible value of `q` is `2`.

## Non-consecutive Ordering

This is a basic permutation problem with a filtering step, but solving it with core.logic is fun. Given a sequence, we want to rearrange it such that no two values are repeated consecutively.

My approach was to define a goal that recursively constrained each pair in (a sliding window over) the input sequence to be unequal. Core.logic defines a `firsto` to set a goal for the first item in a sequence, but we need to define our own `secondo` that does the same but for the second item:
```clojure
(defn secondo [l s]
  (fresh [x]
    (resto l x)
    (firsto x s)))
```

Using that we can write a `nonconseco` goal to recursively constrain the input:
```clojure
(defn nonconseco [l]
  (conde
    ;; empty list is non-consecutive
    [(== l ())]
    ;; singleton list is non-consecutive
    [(fresh [x] (== l (list x)))]
    ;; the real work
    [(fresh [lhead lsecond ltail]
       (conso lhead ltail l)
       (secondo l lsecond)
       (!= lhead lsecond)
       (nonconseco ltail))]))
```
The third case in the `conde` defines three "fresh" logic variables for us to work with: `lhead` and `ltail` will be constrained to the first item of `l` and the rest of `l` respectively, by passing all three to `conso`. `lsecond` is constrained by our new `secondo` goal. Then we constrain `lhead` and `lsecond` to be unequal. Finally we recurse `nonconseco` with the the tail of `l` as its new input.

Those are the only two goals we need to define to make this work. Now we can just wrap it in a function and call it:
```clojure
(defn non-consecutive [coll]
  (first
    (run 1 [q]
      (permuteo coll q)
      (nonconseco q))))

(non-consecutive [1 1 2 2 3 3 4 4 5 5 5])
=> (3 2 3 4 2 4 5 1 5 1 5)
```

This works fine with our custom `nonconseco` goal, but it's not very fast for larger sequences, or when finding _all_ non-consecutive permutations. We can define a predicate function to replace `nonconseco` which should be computationally cheaper than a recursive core.logic goal:

```clojure
(defn non-consecutive? [coll]
  (every? (partial apply not=) (partition 2 1 coll)))

(run* [q]
  (permuteo [\a \b \b \a] q)
  (pred q non-consecutive?))
=> ((\b \a \b \a)
    (\a \b \a \b))
```

Here we found _all_ possible answers by using `run*` instead of `run`. We used core.logic's `pred` to apply our custom predicate function to `q`. We can't apply it directly to `q` because `q` is a logic variable and we must get its actual _value_ to apply a predicate. See the `pred` source for how it does that using `project`.

## Denomination Sums (or making change)

Given a set of denominations e.g. $1, $5, $10, how many ways are there to reach a given amount? This one was tricky!

First we define (another recursive) goal `productsumo` that will initially take a sequence of "fresh" logic variables, a set of denominations, and a sum that we want to reach. We first bind head and tail vars for both `vars` and `dens`, then we use arithmetic functions from the finite domain namespace of core.logic to constrain new fresh vars `product` and `run-sum` such that multiplying a denomination by some number and adding it to a running sum eventually reaches our desired sum (or not).
```clojure
(defn productsumo [vars dens sum]
  (fresh [vhead vtail dhead dtail product run-sum]
    (conde
      [(emptyo vars) (== sum 0)]
      [(conso vhead vtail vars)
       (conso dhead dtail dens)
       (fd/* vhead dhead product)
       (fd/+ product run-sum sum)
       (productsumo vtail dtail run-sum)])))
```

That's the only custom goal we need to solve the problem. Now we can wrap this in a function:
```clojure
(defn change [amount denoms]
  (let [dens (sort > denoms)
        vars (repeatedly (count dens) lvar)]
    (run* [q]
      ;; we want a map from denominations to their quantities
      (== q (zipmap dens vars))
      ;; prune problem space...
      ;; every var/quantity must be 0 <= n <= amount
      (everyg #(fd/in % (fd/interval 0 amount)) vars)
      ;; the real work
      (productsumo vars dens amount))))
```

And now let's find all the ways we can make 14 out of denominations 1, 2, 5, and 10:
```clojure
(change 14 #{1 2 5 10})
=>
({10 0, 5 0, 2 0, 1 14}
 {10 1, 5 0, 2 0, 1 4}
 {10 0, 5 1, 2 0, 1 9}
 {10 0, 5 0, 2 1, 1 12}
 {10 1, 5 0, 2 1, 1 2}
 {10 1, 5 0, 2 2, 1 0}
 {10 0, 5 0, 2 2, 1 10}
 {10 0, 5 2, 2 0, 1 4}
 {10 0, 5 1, 2 1, 1 7}
 {10 0, 5 0, 2 3, 1 8}
 {10 0, 5 0, 2 4, 1 6}
 {10 0, 5 1, 2 2, 1 5}
 {10 0, 5 0, 2 5, 1 4}
 {10 0, 5 2, 2 1, 1 2}
 {10 0, 5 0, 2 6, 1 2}
 {10 0, 5 1, 2 3, 1 3}
 {10 0, 5 0, 2 7, 1 0}
 {10 0, 5 2, 2 2, 1 0}
 {10 0, 5 1, 2 4, 1 1})
```
🤯

## Relational Flatten

How about a slower, more convoluted version of clojure.core's `flatten` using only relational goals?
```clojure
(defn flatteno [l g]
  (condu
    [(emptyo l) (emptyo g)]
    [(fresh [h t]
       (conso h t l)
       (fresh [ft]
         (flatteno t ft)
         (condu
           [(fresh [fh]
              (flatteno h fh)
              (appendo fh ft g))]
           [(conso h ft g)])))]))

(defn logical-flatten [coll]
  (first (run 1 [q] (flatteno coll q))))

(logical-flatten '(((((1) 2)) 3) (4) 5 6 ((() 7)) 8 9))
=> (1 2 3 4 5 6 7 8 9)
```
