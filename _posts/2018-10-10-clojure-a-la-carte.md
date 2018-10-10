---
layout: post
title:  "Syntax Synonyms"
date:   2018-10-09 12:00:00
tags:   clojure instaparse parser syntax hickory html css selector
---

I had an itch to implement some basic CSS selectors for [Hickory](https://github.com/davidsantiago/hickory), a Clojure HTML representation.
Having done [something similar in F# for XPath and HTML]({% post_url 2015-12-27-youtube-comment-markov %})
I thought the same approach would be a good starting point.

My first step was to build a parser for the CSS selector syntax I was interested
in. After some entertaining trial and error in the REPL I had a _Minimum Viable Parser_ using Instaparse.

Then I started thinking about how I could translate its syntax tree into an evaluator against a document tree.
Hickory has a concept of [selectors](https://github.com/davidsantiago/hickory#selectors) which very
conveniently share similar semantics with CSS selectors.

What surprised me most was how easily I could leverage the similarities between the
Instaparse syntax tree and Hickory's selector combinators (in the `syntax->selector` map below)
to turn the syntax tree into a functional evaluator.
I'd expected much more of a struggle to get something working, but the bulk of the code ended up
being declarative grammar and plain syntax transformation:

<script src="https://gist.github.com/taylorwood/d0e7143cb79dd1a077b534a196c94798.js"></script>

And using it is easy too:
```clojure
(->> (slurp "https://clojure.org")
     (h/parse)
     (h/as-hickory)
     (s/select (parse-css-selector "a[href~=reference]")))
=>
[{:type :element,
  :attrs
  {:href "/reference/documentation", :class "w-nav-link clj-nav-link"},
  :tag :a,
  :content ["Reference‍"]}]
```

## Nice

I thought this was a good example of how two well-factored Clojure libraries
and a little combinatory glue can accomplish a relatively complex goal concisely.

See the [repo](https://github.com/taylorwood/hickory-css-selector) and [tests](https://github.com/taylorwood/hickory-css-selector/blob/master/test/hickory_css_selectors_test.clj)
for more examples.
