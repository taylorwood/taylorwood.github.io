---
layout: post
title:  "Parsing with Clojure and Instaparse"
date:   2018-05-17 12:00:00
tags:   clojure parse parsing instaparse grammar
---

Parsing is fun, and I've covered some simple Parsec-style parsers in previous posts, but this one's about Clojure and Instaparse. Instead of building up parsers with combinators, Instaparse takes a grammar as a string and generates a parser. I think this makes for more concise parser definitions that are both easier to write and understand after the fact.

## Bowling Scores

I'm not much of a bowler, but the game has an interesting scoring notation with some special rules. I don't even know if this parser is totally _correct_ but it should be very close. Generally the grammar definitions start _top to bottom_ so things are declared in reverse dependency order. The top line is the general format—ten frames of bowling where each frame is `F` and `10TH` has special rules. Reading downward you can see more terms are defined and combined.

```clojure
(def bowling-score-parser
  (insta/parser
    "S = F F F F F F F F F 10TH
     F = OPEN | CLOSED
     OPEN = ROLL ROLL
     CLOSED = STRIKE | SPARE
     10TH = OPEN |
            SPARE (STRIKE | ROLL) |
            STRIKE (SPARE | ROLL ROLL) |
            STRIKE STRIKE (STRIKE | ROLL)
     STRIKE = 'X'
     SPARE = ROLL '/'
     ROLL = PINS | MISS
     PINS = #'[1-9]'
     MISS = '-'"))
```

And the parser outputs a tree of tagged values:
```clojure
(bowling-score-parser "6271X9-8/XX3572-/X")
=>
[:S
 [:F [:OPEN [:ROLL [:PINS "6"]] [:ROLL [:PINS "2"]]]]
 [:F [:OPEN [:ROLL [:PINS "7"]] [:ROLL [:PINS "1"]]]]
 [:F [:CLOSED [:STRIKE "X"]]]
 [:F [:OPEN [:ROLL [:PINS "9"]] [:ROLL [:MISS "-"]]]]
 [:F [:CLOSED [:SPARE [:ROLL [:PINS "8"]] "/"]]]
 [:F [:CLOSED [:STRIKE "X"]]]
 [:F [:CLOSED [:STRIKE "X"]]]
 [:F [:OPEN [:ROLL [:PINS "3"]] [:ROLL [:PINS "5"]]]]
 [:F [:OPEN [:ROLL [:PINS "7"]] [:ROLL [:PINS "2"]]]]
 [:10TH [:SPARE [:ROLL [:MISS "-"]] "/"] [:STRIKE "X"]]]

;; some more scores to try
(bowling-score-parser "XXXXX6/XX7/XX5")
(bowling-score-parser "XXXXXXXXXXXX")
(bowling-score-parser "XXXXXXXXXX-/")
(bowling-score-parser "XXXXXXXXX--")
(bowling-score-parser "9-9-9-9-9-9-9-9-9-9-")
(bowling-score-parser "X7/9-X-88/-6XXX81")
(bowling-score-parser "X7/9-X-88/-6XX-/X")
(bowling-score-parser "X-/X5-8/9-X811-4/X")
```

## Nested Infix Expressions

This is a nice example of the power of recursive parsers. Notice how the root `S` term is used several times in other terms. This allows for a simple grammar that can parse complex expressions like `((2*3)+1+2)/4`.

```clojure
(def expr-parser
  (insta/parser
    "<S> = VAL | EXPR | PAR
     PAR = <'('> S <')'>
     <EXPR> = S OP S
     VAL = #'[0-9]+'
     OP = '+' | '-' | '*' | '/'"))
```

The `<`angle brackets`>` around LHS terms indicate we don't want those outputs tagged, and angle brackets on the RHS indicate the parsed value shouldn't be returned at all. Omitting tags can make the output easier to work with, and in this case we don't care about the actual parentheses characters.

```clojure
(expr-parser "((2*3)+1+2)/4")
=> ([:PAR [:PAR [:VAL "2"] [:OP "*"] [:VAL "3"]]
          [:OP "+"] [:VAL "1"] [:OP "+"] [:VAL "2"]]
    [:OP "/"] [:VAL "4"])
```

Then with a simple `prewalk` we can interpret the output, ultimately transforming the string into its pure data structure form:
```clojure
(walk/prewalk
  (fn [e]
    (if (vector? e)
      (let [[tag val :as p] e]
        (case tag
          :PAR (rest p)
          :VAL (Integer/parseInt val)
          :OP (symbol val)
          p))
      e))
  (expr-parser "((2*3)+1+2)/4"))
=> (((2 * 3) + 1 + 2) / 4)
```

Happy parsing!