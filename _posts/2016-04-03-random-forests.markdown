---
layout: post
title:  "Working with ALGLIB's Random Decision Forests in F#"
date:   2016-04-03 12:00:00
tags:   random forest decision tree alglib fsharp
---
Lately I've been working on a [Kaggle competition](https://www.kaggle.com/c/home-depot-product-search-relevance) to help Home Depot improve their search results. My team has tried a variety of machine learning algorithms along the way, starting with linear & logistic regression, moving on to [random forests](https://en.wikipedia.org/wiki/Random_forest), as well as experimenting with XGBoost. In this article I'll focus on random forests because they're very easy to use and perform well on a variety of problems.

## Random Forests
My one-liner explanation of random forests is "a bunch of [decision trees](https://en.wikipedia.org/wiki/Decision_tree) grown by using a random subsample of your training data per tree." That's a gross oversimplification but I won't go into detail here -- instead I recommend reading [Mathias Brandewinder's](https://twitter.com/brandewinder) much better explanation in [this post](http://www.clear-lines.com/blog/post/Random-Forest-classification-in-F-first-cut.aspx) that also includes a sample implementation.

Random forests have several appealing properties:

1. Conceptually, they're relatively simple to reason about. It's a combination of basic concepts that form a versatile algorithm. As a bonus, you don't need to understand how they work to get good results out of them.
1. They can be less prone to [overfitting](https://en.wikipedia.org/wiki/Overfitting) than other algorithms, thanks in part to random sampling per tree.
1. Since each tree is grown using random subsampling of the complete training set, the model's performance can be assessed during training using "[out-of-bag](https://en.wikipedia.org/wiki/Out-of-bag_error)" samples. This basically removes the need for a separate [cross validation](https://en.wikipedia.org/wiki/Cross-validation_(statistics)) process.
1. They can be used for regression, classification, and other tasks.
1. The training process can be really fast, especially compared to complex models like neural networks.

## ALGLIB's Implementation
The [ALGLIB](http://www.alglib.net) library provides a variety of tools for machine learning. It's free to use under GPL and targets several languages/platforms. The official site provides some [background](http://www.alglib.net/dataanalysis/decisionforest.php) and [documentation](http://www.alglib.net/translator/man/manual.csharp.html#unit_dforest) regarding its Random Decision Forest implementation.

The general workflow is as follows:

1. **Construct a two-dimensional array of training input:**
    * **Extract some features from your training data.** Your features will need to be converted to an array of floating point numbers. If one of your features is something like a category of `[red, green, yellow]` then you'll need to map those values by identifying the distinct set of classes across the entire data set and mapping each value to a number, e.g. `[red = 1, green = 2, yellow = 3]`.
    * **Isolate your training data output variable:** If your problem is **regression**, your output variable (the property you want your model to predict) will likely already be numeric. If your problem is **classification**, you may need to map the classes to numeric values as well. ALGLIB expects the training data output/class to be the last column in the input array.
1. **Pick some training parameters.** ALGLIB allows the following parameters to be adjusted:
    * **Number of trees:** the documentation recommends 50-100 trees, but don't be afraid to go higher. 600+ trees performed much better for our Kaggle problem.
    * **Random subsample ratio:** the documentation recommends less than 66% or `0.66` for this value. Experiment to find the best value, but I'd guess you usually want something lower depending on the "noise" level of your training data.
    * Optionally specify number of variables to use when determining best split.
1. **Train the model.** Check your email, get a drink, wonder if you used the right parameters.
1. **Use the trained model to make predicitions about new data.** This is when you find out whether your model is good or not! If it's for a Kaggle competition, you'd make predictions against test data, write the predictions out to file, upload it, and see your score.
    * If you're using the model for **regression**, the output array will only contain one number for each prediction.
    * For **classification** the array will contain the probability for each class, e.g. if you had three classes mapped to `[0;1;2]`, the array will contain three probabilities (values between zero and one where one is absolute certainty). If you want to predict just one class, simply pick the one with the highest probability.
    
## Algorithmic Wine Tasting
I picked a simple data set about subjective [wine qualities](http://archive.ics.uci.edu/ml/datasets/Wine+Quality) from U.C. Irvine's Machine Learning Repository. There are 11 feature variables and one output variable (quality). The data can be used for regression or classification because the quality scores are discrete values between one and ten, but we'll go with regression.

Download the [white wine data file](http://archive.ics.uci.edu/ml/machine-learning-databases/wine-quality/winequality-white.csv) and take a look. There are 12 columns and the last column is the output variable, which is what we'll be predicting. We'll use [F# Data](http://fsharp.github.io/FSharp.Data/) `CsvProvider` to read the file:

```fsharp
[<Literal>]
let WineDataPath = __SOURCE_DIRECTORY__ + "/../Data/winequality-white.csv"
type WineDataCsv = CsvProvider<WineDataPath, ";">

let loadTrainingData () =
    WineDataCsv.GetSample().Rows
    |> Seq.map (fun r ->
        [| r.Alcohol
           r.Chlorides
           r.``Citric acid``
           r.Density
           r.``Fixed acidity``
           r.``Free sulfur dioxide``
           r.PH
           r.``Residual sugar``
           r.Sulphates
           r.``Total sulfur dioxide``
           r.``Volatile acidity`` |]
           |> Array.map float,
        r.Quality |> float)
    |> Array.ofSeq
```

*To be continued...*
