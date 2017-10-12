---
layout: post
title:  "Working with ALGLIB's Random Decision Forests in F#"
date:   2016-04-03 12:00:00
tags:   random forest decision tree alglib fsharp
---
My team just finished a [Kaggle competition](https://www.kaggle.com/c/home-depot-product-search-relevance) to help Home Depot improve their search results. We tried a few machine learning algorithms along the way: starting with linear & logistic regression, moving on to [random forests](https://en.wikipedia.org/wiki/Random_forest), and experimenting with [XGBoost](https://xgboost.readthedocs.org/en/latest/). In this post I'll focus on random forests because they're very easy to use and perform well on a variety of problems. We were able to place in the top 5% using a random forest model, but top finishers used a combination of models.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">The <a href="https://twitter.com/hashtag/fsharp?src=hash">#fsharp</a> <a href="https://twitter.com/hashtag/datascience?src=hash">#datascience</a> group finished in the top 5% of the $40K Home Depot <a href="https://twitter.com/hashtag/Kaggle?src=hash">#Kaggle</a> competition! <a href="https://twitter.com/brandewinder">@Brandewinder</a> and <a href="https://twitter.com/squeekeeper">@squeekeeper</a> rock!!!</p>&mdash; Jamie Dixon (@jamie_dixon) <a href="https://twitter.com/jamie_dixon/status/724771906222698497">April 26, 2016</a></blockquote> <script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

## Random Forests
My one-liner explanation of random forests is "a bunch of [decision trees](https://en.wikipedia.org/wiki/Decision_tree) grown by using a random subsample of your training data per tree." That's a gross oversimplification but I won't go into detail here---instead I recommend reading [Mathias Brandewinder's](https://twitter.com/brandewinder) much better explanation in [this post](http://www.clear-lines.com/blog/post/Random-Forest-classification-in-F-first-cut.aspx) that also includes a sample implementation.

Random forests have several appealing properties:

1. Conceptually, they're relatively simple to reason about. It's a combination of basic concepts that form a versatile algorithm. As a bonus, you don't need to understand how they work to get good results out of them.
1. They can be less prone to [overfitting](https://en.wikipedia.org/wiki/Overfitting) than other algorithms, thanks in part to random sampling per tree.
1. Since each tree is grown using random subsampling of the complete training set, the model's performance can be assessed during training using "[out-of-bag](https://en.wikipedia.org/wiki/Out-of-bag_error)" samples. This basically removes the need for a separate [cross validation](https://en.wikipedia.org/wiki/Cross-validation_(statistics)) process.
1. They can be used for regression, classification, and other tasks.
1. The training process can be really fast, especially compared to more complex models like neural networks.

## ALGLIB's Implementation
The [ALGLIB](http://www.alglib.net) library provides a variety of tools for machine learning. It's free to use under GPL and targets several languages/platforms. The official site provides some [background](http://www.alglib.net/dataanalysis/decisionforest.php) and [documentation](http://www.alglib.net/translator/man/manual.csharp.html#unit_dforest) regarding its Random Decision Forest implementation.

The general workflow is as follows:

1. **Construct an array of training input:**
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
I picked a simple data set about subjective [wine qualities](http://archive.ics.uci.edu/ml/datasets/Wine+Quality) from U.C. Irvine's Machine Learning Repository. There are twelve data points for each sample:

1. Fixed acidity 
1. Volatile acidity 
1. Citric acid 
1. Residual sugar 
1. Chlorides 
1. Free sulfur dioxide 
1. Total sulfur dioxide 
1. Density 
1. pH 
1. Sulphates 
1. Alcohol
1. Quality *(a score from 1 to 10 as judged by human tasters, this is what we'll be training our model to predict based on the first 11 variables.)*

The problem can be solved with regression or classification because the quality scores are discrete values from one to ten, but we'll go with regression for simplicity. Classification would require mapping the output classes from `[1 .. 10]` to `[0 .. 9]` for input, and vice versa for output, and would also predict individual probabilities for each class rather than a single, continuous variable.

### Raw Data
Download the [white wine data file](http://archive.ics.uci.edu/ml/machine-learning-databases/wine-quality/winequality-white.csv) and take a look. We'll use [F# Data's](http://fsharp.github.io/FSharp.Data/) `CsvProvider` to read the file:

```ocaml
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

`loadTrainingData` converts every column to `float` (required by ALGLIB) and separates the input variables from the output variable, returning a tuple.

### Working with ALGLIB
Next we'll write a function to wrap ALGLIB's random forest training function:

``` ocaml
let trainForest (features: float[][]) (outputs: float[]) trees subsample =
    let inputArray =
        Seq.zip features outputs
        |> Seq.map (fun (d,o) -> Array.append d [|o|])
        |> array2D
    let featureCount = inputArray.GetLength(1) - 1
    let mutable info = 0
    let forest = alglib.dforest.decisionforest()
    let report = alglib.dforest.dfreport()
    alglib.dforest.dfbuildrandomdecisionforest(
        inputArray,
        npoints = features.Length,
        nvars = featureCount,
        nclasses = 1, // 1 for regression, otherwise would be number of output classes
        ntrees = trees,
        r = subsample,
        info = &info,
        df = forest,
        rep = report)
    match info with
    | 1  ->
        let predictor features =
            let mutable predictions : float[] = [||]
            alglib.dforest.dfprocess(forest, features, &predictions)
            predictions
        predictor, report.oobrmserror
    | -1 -> failwith "Incorrect parameters"
    | -2 -> failwith "Invalid class data"
    | _  -> failwith "Unknown error"
```

This one's pretty messy due to ALGLIB's C-style API, but let's break it down:

* The inputs are `features` - an array of feature value arrays, `outputs` - an array of output values, `trees` - the number of trees to grow in the forest, and `subsample` - the percentage of training data to sample for each tree.
* ALGLIB wants the features and the output variables combined in a two-dimensional array `inputArray`, with the output variable last. This is the only use of F#'s `array2D` I've ever seen...
* For whatever reason it doesn't infer the number of features from the input, so we need to pass that in `featureCount`.
* The training function has three output variables: `info` is a return code, `forest` is the trained Random Forest, and `report` is a data structure with various model statistics. In Xamarin/Mono, I wasn't able to automatically destructure those into a 3-tuple in a `let` binding, even though that works in Visual Studio 2015.
* Finally, we `match` on the return code. On success we return a `predictor` function that takes a `features` array and returns a predicted quality value, and the out-of-bag [RMSE](https://en.wikipedia.org/wiki/Root-mean-square_deviation). The lower the RMSE, the better.

### Growing the Random Forest
We'll want to split our data set in two, using most of it for training data and the rest of it as test data. This function will use a RNG to randomly partition the `data` array according to a given `ratio`. I've chosen to use 90% of the data set as training and the rest for testing.

``` ocaml
let rnd = System.Random(1234) // static seed for reproducable runs
let partitionData ratio data =
    data |> Array.partition (fun _ -> rnd.NextDouble() < ratio)

let allData = loadTrainingData ()
let trainSet, testSet = allData |> partitionData 0.9
let trainFeatures, trainOutputs = trainSet |> Array.unzip
let testFeatures, testOutputs = testSet |> Array.unzip
```

Now we train the forest using 75 trees with 50% subsampling:

``` ocaml
let predictor, oob = trainForest trainFeatures trainOutputs 75 0.5
printfn "Out-of-bag RMSE = %f" oob
```

It only takes a second, then we're ready to make predictions for all samples in the test set:

``` ocaml
let predictions =
    testFeatures
    |> Seq.map (predictor >> Array.head)
```

Let's define a function to print the predictions along with the actual scores, and the differences between them. `printTestPrediction` has some extra logic to print `*` characters next to bad predictions.

``` ocaml
let printTestPrediction (actual, predicted) =
    let delta =
        match predicted - actual with
        | x when abs x > 1. ->
            sprintf "%4.1f %s" x (System.String('*', floor (abs x) |> int))
        | x ->
            sprintf "%4.1f" x
    printfn "Actual = %.0f; Predicted = %.1f; Delta = %s" actual predicted delta

// print actual vs. predicted qualities
Seq.zip testOutputs predictions
|> Seq.iter printTestPrediction
```

You should see output like this when running the program:

```
Out-of-bag RMSE = 0.617041
Actual = 6; Predicted = 5.7; Delta = -0.3
Actual = 6; Predicted = 5.8; Delta = -0.2
Actual = 5; Predicted = 5.6; Delta =  0.6
Actual = 6; Predicted = 6.1; Delta =  0.1
Actual = 6; Predicted = 5.6; Delta = -0.4
Actual = 5; Predicted = 5.1; Delta =  0.1
Actual = 8; Predicted = 7.2; Delta = -0.8
Actual = 6; Predicted = 5.9; Delta = -0.1
Actual = 4; Predicted = 5.0; Delta =  1.0 *
Actual = 6; Predicted = 6.1; Delta =  0.1
Actual = 6; Predicted = 6.4; Delta =  0.4
Actual = 5; Predicted = 5.3; Delta =  0.3
Actual = 5; Predicted = 5.4; Delta =  0.4
Actual = 6; Predicted = 5.4; Delta = -0.6
Actual = 7; Predicted = 6.4; Delta = -0.6
Actual = 6; Predicted = 5.5; Delta = -0.5
Actual = 5; Predicted = 5.7; Delta =  0.7
Actual = 5; Predicted = 5.5; Delta =  0.5
Actual = 5; Predicted = 5.1; Delta =  0.1
...
```

## Summary
You can see the Random Forest has acquired a reasonably sophisticated taste for white wine, and we've barely put forth any effort. It only knows about a few chemical variables of each wine but is able to match humans' opinions fairly well. You could experiment with different numbers of trees and subsampling ratios to see if there's room for improvement, or try approaching the problem using classification.

I hope you've enjoyed this tutorial! If not, have a glass of 🍷 and try to forget about it.
