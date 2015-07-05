---
layout: post
title:  "Markov chain text generation in F#"
date:   2015-07-04 23:11:00
tags:	markov chain text generator
---
A [Markov chain](https://en.wikipedia.org/wiki/Markov_chain) is series of random transitions from one state to another, with no regard to previous state. This idea has [plenty of applications](https://en.wikipedia.org/wiki/Markov_chain#Applications), one of them happens to be generating weird text, and we'll do just that in this post.

## Judging a book by every word between its covers

Imagine we have a large book, made of sentences, made of words. If we consider this book to be a sequence of words and word transitions, we can imagine building a Markov model from it.

Our first requirement is to be able to break a big book string into individual words:
{% highlight fsharp %}
let splitOnSpace text = Regex.Split(text, @"\s+")

let getWordPairs pairSize filePath =
    File.ReadAllText filePath
    |> splitOnSpace
    |> Seq.windowed pairSize
{% endhighlight %}

Some of these techniques are also applied in my [previous text classification posts]({% post_url 2015-06-15-text-classification %}). You may recognize that we're forming *n*-grams from the text using a sliding window (`Seq.windowed`), where *n* is `pairSize`. If our book is *really* big then we may consider a streamed/lazily evaluated approach, but this is a blog post.

At this point you need a book. May I suggest one of the timeless classics available from [Project Gutenberg](https://www.gutenberg.org)? I'll use [Franz Kafka's](https://en.wikipedia.org/wiki/Franz_Kafka) rather neurotic Metamorphosis --- where the main character Jeff Goldblum builds a machine to transform himself into a roach.

Let us build a windowed sequence of roach poetry, with trigrams:
{% highlight fsharp %}
let wordPairs = getWordPairs 3 "/Users/Taylor/Documents/Metamorphosis.txt"
{% endhighlight %}

That'll give you a sliding window sequence of arrays like so:
{% highlight fsharp %}
[[|"One"; "morning,"; "when"|]; [|"morning,"; "when"; "Gregor"|];
 [|"when"; "Gregor"; "Samsa"|]; [|"Gregor"; "Samsa"; "woke"|]; ...]
{% endhighlight %}

That windowed sequence represents the first words of the book: "One morning, when Gregor Samsa woke".

We need a function to help us decompose those word arrays, which will give us a tuple where the first value is every word but the last, and the second value is the last word:
{% highlight fsharp %}
let bisectWords (arr:_[]) =
    let len = Array.length arr
    let preds = arr |> Seq.take (len - 1)
    (preds |> joinWords, arr.[len - 1])
    
// example output
> wordPairs |> Seq.nth 0 |> bisectWords;;
val it : string * string = ("One morning,", "when")
{% endhighlight %}

## Climbing Mt. Markov

For each distinct word, we can find the words directly after it throughout the book. We can construct a `Map<string, string list>`, i.e. each distinct word and a list of the words that can follow it.

This is probably the hardest part to grasp; maybe best to read it from bottom to top. We're going to `fold` over the windowed sequence of words and construct our Markov `Map` as we go. That `Map` will be threaded through the `fold` operation as the accumulated value, and modified with each pass. In `updateMap`, if a particular word (or key) already exists in the `Map` then we replace the binding, otherwise we add a new binding.

{% highlight fsharp %}
let updateMap (map:Map<_,_>) key value =
    if map.ContainsKey key then
        let existingValue = map.[key]
        let map = map |> Map.remove key
        map |> Map.add key (value :: existingValue)
    else
        map.Add(key, [value])

let mapBuilder map words = bisectWords words ||> updateMap map
let buildMap = Seq.fold mapBuilder Map.empty
{% endhighlight %}

In an imperative language this might be written as an iterative loop that modifies a mutable dictionary, and we could do that in F#, but this is more of a recursive solution using immutable data structures. Hopefully the compiler still optimizes this routine to be iterative so it won't overflow the stack!

Let's actually construct the map, which shouldn't take more than a second:
{% highlight fsharp %}
let map = buildMap wordPairs
{% endhighlight %}

Now we'll partition our map into two pieces:

1. Bindings that can start a sentence
2. Bindings that can follow the start of a sentence

We'll need a function to tell us whether a word begins a sentence or not, and we'll use it to partition the map:
{% highlight fsharp %}
let isStartSentence = Regex(@"^[A-Z]").IsMatch // match any title-cased word
let startWords, otherWords = map |> Map.partition (fun k _ -> isStartSentence k)
{% endhighlight %}

Now that we have a `Map` of only "start" words, we can randomly pick one to start a sentence:

{% highlight fsharp %}
let startWordArray = map |> Seq.map (fun kvp -> kvp.Key) |> Array.ofSeq

let rng = Random()
let getRandomItem seq =
    let randIndex = rng.Next (Seq.length seq)
    seq |> Seq.nth randIndex
{% endhighlight %}

## Chaining it up

We're dangerously close to actually generating gibberish sentences. All we need is a function to tell us when we've reached the end of a sentence:

{% highlight fsharp %}
// just look for ending punctuation, but don't match 'Mr.' or 'Mrs.'
let (|IsEndOfSentence|) = Regex(@"(?<!Mr(s)?)[\.\?\!]$").IsMatch
{% endhighlight %}

And then we can write a recursive function to walk our Markov map, accumulating randomly chosen words for each state transition, and then joining them together to form a sentence:

{% highlight fsharp %}
// we'll need this function to work with larger word pair sizes
let combineWords prev next =
    [prev; next]
    |> List.filter (fun s -> not (String.IsNullOrWhiteSpace s))
    |> joinWords

// recursive function to accumulate Markov sentence until end-condition reached
let rec markovChain state acc =
    let nextChoice = map.[state] |> getRandomItem
    match nextChoice with
    | IsEndOfSentence true ->
        nextChoice :: acc
    | IsEndOfSentence false ->
        let currWords = state |> splitOnSpace |> Seq.skip 1 |> joinWords
        markovChain (combineWords currWords nextChoice) (nextChoice :: acc)
        
let getMarkovSentence startWord =
    markovChain startWord [startWord]
    |> List.rev
    |> joinWords
{% endhighlight %}

This has some similarities to our recursive `fold` method of building the Markov map, in that it's working with a current state and an accumulator. The `markovChain` will go on indefinitely, randomly choosing the next element and accumulating until reaching its stop condition.

## Hot Kafka

How many sentences of this stuff do we want? The sky's the limit. We can generate more Kafkaesque material in milliseconds than Franz himself could conjure up in a million years!

We'll start with a few sentences:

{% highlight fsharp %}
let sentences = [for i in 1 .. 3 do yield getMarkovSentence (getRandomItem startWords).Key]
sentences |> joinWords |> printfn "%A"
{% endhighlight %}

> "Gregor found it easy to see him?", and Gregor would have to re-arrange his life. Unlike him, she was looking out the window after she had to carry on with our lives and remember him with respect. It was only able to cover it and be patient, just to move about, crawling up and flat, they had no thought of closing the door and without letting the women when they went round the chest, pushing and pulling at it and rubbed it against the ground.

This may be even creepier than the source material!

### Word pair sizing

Notice if you increase the size of the word pairings, e.g. `getWordPairs 4`, the generated material starts to resemble the original text more closely --- this is because each state of your Markov map is more specific. Using a word pair size of 4 or 5 will often regenerate sentences from the source text verbatim. Using a value of 2 will generate much more *random* looking text, because each state consists of only one word. This will also depend on the amount and variety of source text used to build your Markov model.