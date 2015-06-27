---
layout: post
title:  "Simple text classification with a little F# (Part 3)"
date:   2015-06-27 10:15
tags:	machine learning text classification term frequency inverse document frequency tfidf
---
My [previous post]({% post_url 2015-06-19-text-classification-2 %}) described the TF-IDF technique. Now we're going to put it all together to build a working text classifier.

### Similarities

Our simple classifier is based on a similarity (or distance) function. All we need to do is compare an unknown document with each known document. When we know what known document is most similar to our unknown document, then we can make a pretty good guess about what class the unknown document belongs to.

Let's define a similarity function:
{% highlight fsharp %}
let similarity unknown known =
    let getTerms = Seq.map fst >> Set.ofSeq
    let getWeights = Seq.map snd

    let unknownTerms = getTerms unknown
    let unknownWeights = getWeights unknown

    let knownTerms = getTerms known
    let knownWeights = getWeights known

    let commonTerms = Set.intersect unknownTerms knownTerms
    let isCommonTerm term = commonTerms |> Set.exists (fun w -> w = fst term)
    let commonWeights tfidfs = tfidfs |> Seq.filter isCommonTerm |> getWeights
    
    let commonUnknownWeights = commonWeights unknown
    let commonKnownWeights = commonWeights known

    (Seq.sum commonKnownWeights + Seq.sum commonUnknownWeights)
    / (Seq.sum unknownWeights + Seq.sum knownWeights)
{% endhighlight %}

Now a play by play analysis:

* We have two arguments: an `unknown` document, or a sequence of `TFIDF` values, and a `known` document, also a sequence of `TFIDF` values.
* `getTerms` is a function to "decompose" a sequence of `TFIDF` values. A `TFIDF` is a tuple with the first (`fst`) item being a `Term`. This output will be a set of `Term` values.
* `getWeights` is about the same, only it gets the second (`snd`) value from the pair: the actual TF-IDF weighting.
* Next we'll get the terms and weights of the `unknown` document, and then the `known` document: `unknownTerms`, `unknownWeights`, `knownTerms`, `knownWeights`.
* `commonTerms` is the intersection set of both known and unknown `Terms`.
* `isCommonTerm` is a function that tells us if a given `Term` is in the set of `commonTerms`.
* `commonWeights` will take a sequence of TF-IDF values and only give back the ones that correspond to the set of common terms.
* Next we'll get the common weights for both the known and unknown document: `commonUnknownWeights` and `commonKnownWeights`.
* Finally, we calculate the similarity between the weights of both documents. This will give us a number between 0 and 1, with 1 meaning the two documents are identical.

### Finding a match

Now we can evaluate our similarity function between our unknown document and every unknown document:

{% highlight fsharp %}
let calcCategoryProbabilities (trainedData: TrainedData) query =        
    //calculate similarity score per category, using highest score from each
    let calcSim ws = similarity query ws
    let scoreSamples samples = samples |> Seq.map calcSim
    let scores =
        trainedData
        |> Seq.map (fun kvp -> kvp.Key, (scoreSamples kvp.Value) |> Seq.max)

    //sort scores descending
    scores |> Seq.sortBy snd |> List.ofSeq |> List.rev
{% endhighlight %}

* Our two arguments are `trainedData`, representing the entirety of our training set, and `query` is our unknown document (a sequence of `TFIDF`).
* `calcSim` is a partially applied function; a shorthand for calculating the similarity between each weighted sample (`ws`) and our `query` document.
* `scoreSamples` will give us a sequence of similiarities between a sequence of weighted samples and our `query` document.
* `scores` is the result of feeding our `trainedData` into a `map` function that will give us the highest similarity score per document category/class.
* Finally, we sort the `scores` in descending order, giving us a set of each document category and how similar our `query` document is to each one. The first result would be the document category that `query` is most similar to.

## All done

We've built a very rudimentary document classification program. It was written for tutorial purposes, so it's not highly optimized or sophisticated. You could replace the similarity/distance function with something fancier. You could extract different features besides TF-IDF. The **best** thing you could do is pick one of many existing libraries for machine learning and learn how to use it instead.

Hopefully this series of posts has given you some insight and maybe even inspiration to learn more about machine learning. Happy programming!

Get all the F# code for this series [here](https://github.com/taylorwood/ADRDemo).