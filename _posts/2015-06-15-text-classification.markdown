---
layout: post
title:  "Simple text classification with a little F# (Part 1)"
date:   2015-06-15 09:13
tags:	machine learning text classification
---
In this series I'll describe a simple recipe for text classification. Some of the basic techniques here can still be applied and built upon for more sophisticated classification techniques.

Get all the F# code for this post [here](https://github.com/taylorwood/ADRDemo).

> Disclaimer: There are many great [off-the-shelf](https://code.google.com/p/word2vec/) [packages](http://scikit-learn.org/stable/tutorial/text_analytics/working_with_text_data.html) available for machine learning and text classification, and you'd be better served using those rather than rolling your own. This post is mostly intended to be an easy-to-understand tutorial on text classification.

## How I learned to teach a computer something

At a previous job, I was faced with a problem of [document classification](https://en.wikipedia.org/wiki/Document_classification). Workers were poring over huge files, page by page, manually building a table of contents for each one. It took a long time to train workers on how to classify these documents, and even the best workers were inconsistent. There was no good way to scale the operation.

We needed a system that could ingest large files, each with thousands of pages of text, and automatically classify each individual document they contained. I knew there had to be a way to teach a computer how to recognize this stuff. This was a new frontier for me, so I started reading a lot of wikipedia articles and thought really hard.

## Training data

There were a few hundred types of documents we needed to identify. We had hundreds of pre-labeled samples of each document type. I knew I needed to leverage those samples to build something that could read a document it's never seen and tell me which category it fell under. Those samples would be my [training data](https://en.wikipedia.org/wiki/Training_set).

> It's often the case, at least with *supervised* machine learning, that your training set plays a much more important role in your classifier's accuracy than which algorithm or model you use.

We'll revisit this topic once we get a working prototype and want to improve its accuracy.

### Boiling it down

I needed to grind up those samples so I could derive some useful information from their contents. Firstly, I needed to discard any noise: punctuation, symbols, etc. Let's define some functions for sanitizing our samples:
{% highlight fsharp %}
let inline isCharAllowed c =
    Char.IsLetterOrDigit c || Char.IsWhiteSpace c || c = '-'
    
let sanitizeText (text: string) =
    let cleanChars = text |> Seq.filter isCharAllowed |> Seq.toArray
    new String(cleanChars)
{% endhighlight %}

### Tokenization

Tokenization is the process of breaking down the text into its constituent words (or tokens). We'll simply break the text up into individual words by splitting on whitespace. Then we can optionally combine those individual words into [n-grams](https://en.wikipedia.org/wiki/N-gram).

We're using a simple [bag of words](http://en.wikipedia.org/wiki/Bag-of-words_model) model. Imagine you've ripped a page from a book and very meticulously cut out each word with a tiny pair of scissors and put them in a bag. You'd have a bag of unigrams.

> An individual word is a unigram, a pair of words is a bigram, and so on. Extracting n-grams larger than a unigram can be done using a "sliding window". For example, the string `these pretzels are making me thirsty` would produce the following trigrams: `these pretzels are`, `pretzels are making`, `are making me`, `making me thirsty`.

{% highlight fsharp %}
let getNGrams n words =
    let join (items: string[]) = String.Join(" ", items)
    words |> Seq.windowed n |> Seq.map join
    
let getWords gramSize (text: string) =
    let words = Regex.Split(text, @"\s+")
    words |> getNGrams gramSize |> Seq.toList
{% endhighlight %}

### Term frequency

Regardless of which n-gram size we choose, we still end up with a bag of them. What information can we derive from the bag's contents? We've lost any information about the original order of the words, but we *can* count how many times we find each n-gram in the bag.

> If your document domain is fairly homogenous, larger n-grams might help differentiate between similar documents. However, using too large an n-gram may cause your classifier to only recognize documents that are very similar to your training set.

For efficiency's sake, we'll map each n-gram to an integer, e.g. `cat` will always map to 1, `dog` will always map to 2, etc. We could technically do without this, but working with the original strings throughout our classifier would waste memory and CPU cycles.
{% highlight fsharp %}
type Term = int
{% endhighlight %}

We'll define a term frequency tuple that pairs a Term with the number of times it appears in a document:
{% highlight fsharp %}
type TermFrequency = Term * int
{% endhighlight %}

We'll define a record type for representing each document we're using as a training sample. It holds the path to the physical file and its set of term frequencies:
{% highlight fsharp %}
type TrainingSample = {Path: string; Frequencies: Set<TermFrequency>}
{% endhighlight %}

We'll define a map to organize training samples into categories (or classes):
{% highlight fsharp %}
type Category = string
type TrainingData = Map<Category, TrainingSample list>
{% endhighlight %}

We're going to map each of our n-grams to a Term, so that `cat` always maps to the same value regardless of which document it came from:
{% highlight fsharp %}
let gramTokenMap = new Dictionary<string,Term>()
{% endhighlight %}

And finally, a function to calculate the term frequencies for a collection of n-grams, i.e. a document:
{% highlight fsharp %}
let getTermFreqs words = words |> Seq.countBy (fun w -> w)
{% endhighlight %}

## Take a breather

In the next post we'll discuss why term frequencies alone may not be sufficient, and how we can supplement them. Then we'll start putting this training data to work!