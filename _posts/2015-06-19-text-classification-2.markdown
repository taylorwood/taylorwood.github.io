---
layout: post
title:  "Simple text classification with a little F# (Part 2)"
date:   2015-06-19 21:19
tags:	machine learning text classification term frequency inverse document frequency tfidf
---
My [previous post]({% post_url 2015-06-15-text-classification %}) laid out the basics for calculating term frequencies from text documents. This post will describe a method for supplementing term frequency that will make it more useful.

## Why term frequency is not enough

The number of times a term appears in a document was perhaps the most obvious statistic we could derive from our bag of words model. Once you start gathering this stat from a variety of documents, you may realize that certain terms are *ubiquitous*. Because they appear in nearly every document, they're almost useless as a signifier of a given document's class.

There's another useful statistic that can combat this problem by augmenting our term frequencies: [inverse document frequency](https://en.wikipedia.org/wiki/Tf–idf).

### Term Frequency - Inverse Document Frequency

We can negatively weight a term's frequency relative to the number of times it appears in *all* documents. For example, if your corpus is all about real estate, the word `address` may have an extraordinary document frequency. We want to negatively weight that word to decrease its importance.

Here's a function that will calculate the document frequency of a given term in a `TrainingSample` set:
{% highlight fsharp %}
let docFreq allSamples term =
    let sampleHasTerm sample = sample.Frequencies |> Seq.exists (fun w -> fst w = term)
    allSamples |> Seq.filter sampleHasTerm |> Seq.length
{% endhighlight %}

And a simple function for calculating the inverse document frequency (adding 1 to `docFrequency` will avoid division by zero when a term isn't found in a set of documents):
{% highlight fsharp %}
let calcInverseDocFreq sampleCount docFrequency = 
    Math.Log (float sampleCount / (float docFrequency + 1.0))
{% endhighlight %}

We'll define a new tuple for this using `float`, since the TF-IDF value won't be a whole number.
{% highlight fsharp %}
type TFIDF = Term * float
{% endhighlight %}

> Words like `the`, `and`, `with`, etc. aren't very useful for text classification purposes, regardless of your subject matter. These are commonly referred to as [stop words](https://en.wikipedia.org/wiki/Stop_words). You would generally create a blacklist of these words to be filtered from your input before you calculate TF-IDF.

### Crunching numbers

Using the functions we've just defined, we can analyze all our samples and create a map of IDF values for all our terms:
{% highlight fsharp %}
let getTermIDFMap allSamples terms =
    let docFreq = docFreq allSamples
    let sampleCount = Seq.length allSamples
    let idf = docFreq >> calcInverseDocFreq sampleCount
    terms
    |> Array.ofSeq
    |> Array.Parallel.map (fun t -> t, idf t)
    |> Map.ofSeq
{% endhighlight %}
In summary, we're taking a set of all samples and all terms as input, and returning a `Map` of each `Term` to its IDF value. Let's break down what's going on in this function:

1.	The `docFreq` function is being [partially applied](https://en.wikipedia.org/wiki/Partial_application). This allows us to reference a shorthand version of this function that will always take `allSamples` as its first argument.
2.	We're defining an `idf` function that composes our partially applied `docFreq` with a partially applied `calcInverseDocFreq`. Functional composition allows us to pipeline these partially applied functions, creating a new function that takes only one argument of interest: `term`. Note that the order of function arguments plays an important role in function composition. This particular example wouldn't work if `term` wasn't the last argument to each composed function.
3.	Notice the use of `Array.Parallel.map`. This will perform our `map` functions in parallel, greatly reducing the time required to compute IDF when multiple cores are available!

#### Augmenting Term Frequency with Inverse Document Frequency

This function takes our `Map` of `Term` to IDF values, and a `TermFrequency`, and gives us a weighted `TermFrequency`:
{% highlight fsharp %}
let tfidf (idfs: Map<Term, float>) tf =
    let term, frequency = tf
    let hasWeight = idfs.ContainsKey term
    let idf = if hasWeight then idfs.[term] else 1.0
    term, float frequency * idf

let weightTermFreqs idfs = Seq.map (tfidf idfs)
{% endhighlight %}
Thanks to TF-IDF, common terms will now carry less weight in our model.

## Next steps

We've got the TF-IDF feature extraction covered. In the [next post]({% post_url 2015-06-27-text-classification-3 %}) we'll start comparing our extracted features to build a working classifier!