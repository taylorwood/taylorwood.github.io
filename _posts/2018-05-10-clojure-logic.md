---
layout: post
title:  "Logic programming in Clojure"
date:   2018-05-10 12:00:00
tags:   clojure logic core logic core.logic
---

This post covers some toy problems I've solved with Clojure's [core.logic](https://github.com/clojure/core.logic).

## Introduction

Logic and constraint-based programming interest me, which might explain why I enjoy SQL so much.
I like the idea of describing a problem declaratively and having a computer give an exhaustive set of answers.
For some problems, a "mechanical" solution might require much more code, and maybe much more complexity.

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

## Cryptarithmetic

I'd never seen this type of problem before [this Stack Overflow question](https://stackoverflow.com/q/55315374) but it's great for logic programming. See [Wikipedia](https://en.wikipedia.org/wiki/Verbal_arithmetic) for a detailed description and examples:

>Verbal arithmetic, also known as alphametics, cryptarithmetic, cryptarithm or word addition, is a type of mathematical game consisting of a mathematical equation among unknown numbers, whose digits are represented by letters. The goal is to identify the value of each letter.

>The classic example, published in the July 1924 issue of Strand Magazine by Henry Dudeney is:

>        SEND
>     +  MORE
>     -------
>     = MONEY

>The solution to this puzzle is O = 0, M = 1, Y = 2, E = 5, N = 6, D = 7, R = 8, and S = 9.

I saw some similarities between this and the denominational sum problem above, and I was able to use the `productsumo` goal in solving this too. I also need another, simpler summation goal:
```clojure
(defn sumo [vars sum]
  (fresh [vhead vtail run-sum]
    (conde
      [(== vars ()) (== sum 0)]
      [(conso vhead vtail vars)
       (fd/+ vhead run-sum sum)
       (sumo vtail run-sum)])))
;; and for getting sum of individual digits later
(defn magnitudes [n]
  (reverse (take n (iterate #(* 10 %) 1))))
```

The solver function creates logic variables based on the input words.
1. Get the set of all characters from all words
1. Zip those into a map with fresh logic variables, which will also be used to represent the answer
1. Get the logic variables for the first letter of each word; they must be constrained to non-zero numbers because the word-numbers can't have leading zeroes
1. Create a logic variable per word to hold the sum of each word's digits, reached with `magnitudes` and `productsumo`
1. Create groups of the character-level logic variables, representing each word

Then in `run*` the goals are tied together.
```clojure
(defn cryptarithmetic [& words]
  (let [distinct-chars (distinct (apply concat words))
        char->lvar (zipmap distinct-chars (repeatedly (count distinct-chars) lvar))
        lvars (vals char->lvar)
        first-letter-lvars (distinct (map #(char->lvar (first %)) words))
        sum-lvars (repeatedly (count words) lvar)
        word-lvars (map #(map char->lvar %) words)]
    (run* [q]
      (everyg #(fd/in % (fd/interval 0 9)) lvars) ;; digits 0-9
      (everyg #(fd/!= % 0) first-letter-lvars)    ;; no leading zeroes
      (fd/distinct lvars)                         ;; only distinct digits
      (everyg (fn [[sum l]]                       ;; bind sums per word
                (productsumo l (magnitudes (count l)) sum))
              (map vector sum-lvars word-lvars))
      (fresh [s]
        (sumo (butlast sum-lvars) s) ;; sum all input word sums
        (fd/== s (last sum-lvars)))  ;; input word sums must equal last word sum
      (== q char->lvar))))
```

There are sample _alphametics_ around the internet. It seems the most common example is "send more money", which only has one possible solution. Most random phrases I've tried have either zero or many solutions — of course for the math to work out, the last word must be as long (i.e. have as many digits) as the longest word in the phrase.
```clojure
(cryptarithmetic "send" "more" "money")
=> ({\s 9, \e 5, \n 6, \d 7, \m 1, \o 0, \r 8, \y 2})
```

The longest one I found is on Wikipedia, and takes about 30 seconds to run:
```clojure
(def real-long-alphametic
  ["SO" "MANY" "MORE" "MEN" "SEEM" "TO"
   "SAY" "THAT" "THEY" "MAY" "SOON" "TRY"
   "TO" "STAY" "AT" "HOME" "SO" "AS" "TO"
   "SEE" "OR" "HEAR" "THE" "SAME" "ONE"
   "MAN" "TRY" "TO" "MEET" "THE" "TEAM"
   "ON" "THE" "MOON" "AS" "HE" "HAS"
   "AT" "THE" "OTHER" "TEN" "TESTS"])
(apply cryptarithmetic real-long-alphametic)
=> ({\A 7, \E 0, \H 5, \M 2, \N 6, \O 1, \R 8, \S 3, \T 9, \Y 4})
```

It's useful to print these out:
```clojure
(defn pprint-answer [char->digit words]
  (let [nums (map #(apply str (map char->digit %))
                  words)
        width (apply max (map count nums))
        width-format (str "%" width "s")
        pad #(format width-format %)]
    (println
     (clojure.string/join \newline
       (concat
        (map #(str "+ " (pad %)) (butlast nums))
        [(apply str (repeat (+ 2 width) \-))
         (str "= " (pad (last nums)))]))
     \newline)))

(defn pprint-cryptarithmetic [& words]
  (doseq [answer (apply cryptarithmetic words)]
    (pprint-answer answer words)))

(pprint-cryptarithmetic "what" "the" "hell")
+ 2368
+  831
------
= 3199 

+ 4526
+  651
------
= 5177 

+ 7826
+  685
------
= 8511 

+ 7852
+  281
------
= 8133
```
There's only four possible answers for _what the hell._

Update: I found a couple instances of this problem are also in the [core.logic benchmark suite](https://github.com/clojure/core.logic/blob/10ee95eb2bed70af5bc29ea3bd78b380f054a8b4/src/main/clojure/clojure/core/logic/bench.clj#L328) itself.

## Non-consecutive Ordering

This is a permutation problem with a filtering step, but solving it with core.logic is fun. Given a sequence, we want to rearrange it such that no two values are repeated consecutively.

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

## Non-relational Flatten

How about a slow, impractical version of clojure.core's `flatten`?
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
I have a feeling `flatteno` can be simplified but I haven't finished _The Reasoned Schemer_ yet.
