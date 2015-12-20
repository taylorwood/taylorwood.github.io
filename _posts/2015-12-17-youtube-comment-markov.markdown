---
layout: post
title:  "Generating Markov text from YouTube comments"
date:   2015-12-27 12:00:00
tags:	markov chain text generator YouTube comments web scraping fparsec xpath
---
Let's build on my post from a few months ago about [generating weird text with Markov chains]({% post_url 2015-07-04-markov-text %}). This post will focus on a purely [GIGO](https://en.wikipedia.org/wiki/Garbage_in,_garbage_out) workflow — instead of feeding our Markov text generator with literary classics, we're going to feed it [YouTube comments](http://www.newyorker.com/humor/daily-shouts/comments-feeling-youtube).

We'll also use [FParsec](http://www.quanttec.com/fparsec/) to implement a subset of [XPath](https://en.wikipedia.org/wiki/XPath) for [F# Data's](http://fsharp.github.io/FSharp.Data/) `HtmlDocument`.

*I've left some uninteresting but important bits of code out for brevity. All the code for this post can be found on [GitHub](https://github.com/taylorwood/YouTudes).* 

## Request for comments
Our plan is to scrape the comment text for a given search query's videos. We can use the very convenient HTTP functionality in [F# Data](http://fsharp.github.io/FSharp.Data/) for working with [YouTube.com](https://www.youtube.com).

> Google provides a [YouTube Data API](https://developers.google.com/youtube/v3/docs/) for things like this, but we'll take the brute approach and depend on specific markup in YouTube's HTML that may break suddenly. Use their API if you're serious about harvesting garbage comments on an ongoing basis.

First, we'll issue a search `query`, parse the session token from the response, then load the response in a `HtmlDocument`:
{% highlight fsharp %}
let searchHtml = Http.RequestString(searchUrl, cookieContainer = cookies)
let sessionToken = // required for comment JSON requests
    match getSessionToken searchHtml with
    | Some t -> t
    | None -> failwith "Failed to parse session token from HTML"
let searchDoc = HtmlDocument.Parse searchHtml
{% endhighlight %}

### And now for something completely different
I figured an unnecessary and fun diversion for this task would be to implement XPath selection to make our scraper more concise. `HtmlDocument` enables easy access to the response's DOM elements, but might be unwieldy if the element(s) you're after are deeply nested and/or vaguely decorated. An XPath query can get us pretty far in just one line (`//table[@id='results']/tbody/tr//a[@href]`), and most browsers will generate XPaths for you from their developer tools.

We'll need two things to pull this off:

1. A parser to turn that XPath string into a useful structure, and...
2. Some code to evaluate that structure against a `HtmlDocument`.

#### Path parsing
Let's see how easy it is to define a parser for a simple subset of XPath. First, we'll define the types we want our parser to produce:
{% highlight fsharp %}
type XPathPart = Axis * NodeTest * Predicate list
and Axis =
    | Child
    | DescendantOrSelf
and NodeTest = string
and Operator =
    | Equals
    | NotEquals
and Predicate = {
    Attribute: string
    Filter: (Operator * string) option }
{% endhighlight %}

Given a valid XPath, our parser should give us a list of `XPathPart` -- one for each XPath segment.

FParsec defines many helpful primitives for building a parser, on top of which we'll build our own primitives. The following parser segments correspond to the `Axis`, `NodeTest`, and `Predicate` that constitute an `XPathPart`:
{% highlight fsharp %}
// single or double slash separator
let separator = stringReturn "/" Child
let descOrSelf = stringReturn "//" DescendantOrSelf
let axis = choice [descOrSelf; separator; preturn Child]
// node name or asterisk (all nodes)
let tagName = many1Satisfy (fun c -> isAsciiLetter c || isDigit c)
let nodeName = choice [tagName; pstring "*"]
// predicate with optional value test, e.g. [@disabled] or [@id='hey']
let equal = stringReturn "=" Equals
let unequal = stringReturn "!=" NotEquals
let operator = spaces >>. choice [equal; unequal] .>> spaces
let predicateName = many1Satisfy (fun c -> isAsciiLetter c || isDigit c || c = '-')
let quoted = between (pstring "'") (pstring "'")
let comparison = operator .>>. choice [quoted predicateName; predicateName]
let predicate = between (pstring "[@") (pstring "]") (predicateName .>>. opt comparison)
{% endhighlight %}
I hope that code is *mostly* self-explanatory. The `>>` operators combine two parsers, and the `.` on either side specifies which parser value to keep, e.g. the `spaces` in `operator` are allowed/parsed but discarded. The `.>>.` in `comparison` combines two parsers and keeps both of their values.

We can assemble our component parsers into a unified XPath parser like this:
{% highlight fsharp %}
// parses one XPath segment ("/div[@id='hey'][@disabled]")
let xPathPart =
    let predicates = many predicate
    pipe3 axis nodeName predicates getXPathPart
// parses an entire XPath comprised of 1-to-many segments ("/div/ol/li/*")
let xPathParts = (many1 xPathPart) .>> eof
{% endhighlight %}
`pipe3` combines our three primary component parsers and pipes their output to the unlisted `getXPathPart`, which just constructs an `XPathPart` from their values. Adding the `eof` parser on the end ensures the entire string can be parsed as a XPath.

Even though we've written no error handling code, FParsec gives nice error messages for bad input:
{% highlight ruby %}
Failure: Error in Ln: 1 Col: 9
div/ul/*[💩]
        ^
Expecting: end of input, '*', '/', '//' or '[@'
{% endhighlight %}

#### Walking the path
Now that we can parse a simple XPath, we can evaluate it. The entry point is the `evaluate` function below. It recursively evaluates a `XPathPart list` from the context of a given `HtmlNode` (usually the root), returning any elements that satisfy the XPath's requirements.

{% highlight fsharp %}
let satisfiesPredicate (n: HtmlNode) pred =
    let attr = n.TryGetAttribute(pred.Attribute)
    match attr with
    | Some a ->
        match pred.Filter with
        | Some (op, comp) -> // perform comparison
            let value = a.Value()
            match op with
            | Equals -> compareValues n pred.Attribute value comp
            | NotEquals -> not <| compareValues n pred.Attribute value comp
        | None -> true // attr found, no comparison
    | None -> false // attr not found

let evaluate' part (n: HtmlNode) =
    let axis, name, preds = part
    let searchNodes =
        match axis with
        | Child ->
            if name <> "*" then n.Elements(name) else n.Elements()
            |> Seq.ofList
        | DescendantOrSelf ->
            if name <> "*" then n.DescendantsAndSelf(name) else n.DescendantsAndSelf()
    let isMatch n = preds |> List.forall (satisfiesPredicate n)
    searchNodes |> Seq.where isMatch

let evaluate xPath node =
    let rec loop parts nodes =
        match parts with
        | [] -> nodes
        | p::ps ->
            let ns = nodes |> Seq.map (evaluate' p) |> Seq.collect id |> List.ofSeq
            loop ps ns
    loop xPath [node]
{% endhighlight %}

### Back to scraping
Now we can find each video's URL with the following code:
{% highlight fsharp %}
let videoUrls =
    searchDoc.Select("//*[@class=yt-lockup-title]/a[@href]")
    |> Seq.map (fun a -> host + a.AttributeValue("href"))
    |> Seq.where (fun url -> url.Contains("/watch?"))
{% endhighlight %}

YouTube's tasteless comments are loaded via AJAX request after the video page loads. The request needs to be POSTed with some particular values or it won't work. The following function handles that and will return the raw string JSON response:
{% highlight fsharp %}
let getCommentJson videoUrl jsonUrl =
    let body = FormValues [("session_token", sessionToken); ("client_url", videoUrl)]
    let headers = [("Referer", videoUrl); ("Origin", host)
                    ("Content-Type", "application/x-www-form-urlencoded"); ("DNT", "1")]
    printfn "Requesting comments for %s..." jsonUrl
    Http.RequestString(jsonUrl, httpMethod = "POST", body = body, headers = headers, cookieContainer = cookies)
{% endhighlight %}

All of that can be tied together with this function:
{% highlight fsharp %}
let getVideoCommentJson videoUrl =
    match getVideoId videoUrl with
    | Some id -> getCommentUrl id |> getCommentJson videoUrl
    | None -> failwithf "Failed to parse video ID from video URL %s" videoUrl
{% endhighlight %}

Now we have everything we need to request and parse the JSON comment responses for each video:
{% highlight fsharp %}
let commentBodies =
    videoUrls
    |> Seq.map (getVideoCommentJson >> JsonValue.Parse)
    |> Seq.choose
        (fun j ->
            maybe {
                let! body = j.TryGetProperty("body")
                let! html = body.TryGetProperty("watch-discussion")
                return html
            })
{% endhighlight %}

The JSON response are basically just containers for embedded HTML. We can use `HtmlDocument` again to parse those fragments, get at the comment containers, then extract their inner text:
{% highlight fsharp %}
let getCommentText (body: JsonValue) =
    let commentHtml = HtmlDocument.Parse(body.AsString())
    commentHtml.Select("//*[@class=comment-text-content]")
    |> Seq.map HtmlNodeExtensions.InnerText

let comments = commentBodies |> Seq.map getCommentText |> Seq.collect id
{% endhighlight %}

The `comments` value is now a sequence of all comments for all videos we found.

## Garbage compactor
Now that we have all this inane text, let's use it to generate some *insane* text. We'll combine all the comments and feed it to `MarkovTextBuilder`:

> You'll need the [FsMarkov](https://github.com/taylorwood/FsMarkov) code from my previous post for this.
{% highlight fsharp %}
let nGramSize = 3 // try different values of n
let corpus = comments |> String.concat Environment.NewLine
let nGrams = getWordPairs nGramSize corpus
let map = buildMarkovMap nGrams
let generator = MarkovTextBuilder(map)

// print a few sentences out
generator.GenerateSentences 10
|> joinWords
|> printfn "%A"
{% endhighlight %}

What happens if we feed it with comments from `mariah carey christmas`?

> WHEN YOU'RE WATCHING IT, YOU'RE GOING TO DIE ANYWAY﻿ These kids under 15 have nothing else better to do amazing at this next christmas concert shes having i want for Christmas is FOOODDDDD :D﻿ Lol﻿ TRUUUU﻿ ▬▬▬▬▬▬▬▬▬▬ஜ۩۞۩ஜ▬▬▬▬▬▬▬▬▬▬▬▬▬▬ Who's watching this in DECEMBER ▬▬▬▬▬▬▬▬▬▬ஜ۩۞۩ஜ▬▬▬▬▬▬▬▬▬▬▬▬▬▬﻿ Im spicy.....﻿ +U 1.27 go away, don't copy and paste this comment.﻿ ▬▬▬▬▬▬▬▬▬▬ஜ۩۞۩ஜ▬▬▬▬▬▬▬▬▬▬▬▬▬▬ Who's watching this on August... LEARN HOW TO read because Mariah slays you and silent night is seconds but its was first can some one help me out here and chubby people can be just as beautiful as people who are not consistent (as you said). JB about this! AIWFCIY... it's Silent Night.... Mariah Carey, Merry Christmas to everyone, Jesus is the coolest guy in the beginning...but what happened lmao﻿ If you are the best, second is Sesame Street, third is Madonna

Interesting.

## Mashups

My [YouTudes](https://github.com/taylorwood/YouTudes) project can be run as a console application that asks for a query, scrapes any resulting video comments, then writes them all to text file. You can make your own Markov text mashups by running it for a few different queries and combining them into `corpus` like this:
{% highlight fsharp %}
let corpus =
    Directory.GetFiles("/tmp/garbage/comments", "*.txt")
    |> Seq.map File.ReadAllText
    |> String.concat Environment.NewLine
{% endhighlight %}

What happens if we feed it with a mashup of `donald trump` and `more than a feeling`?

> ESTABLISHMENT?? There goes CNN again. My sister was very close with him﻿ played it in your lap.﻿ I'm half expecting Will Ferrell to appear mid-song with more cowbell.﻿ +random666777 NO, why spoil a good time.﻿ I think you are going to win big ! Maybe the rest of US repugnant morality and hypocrisy if youd like; basically however, the USA split into 50 separate nations so ppl would finally stop moaning about "this cancer in Washington" and get the best". I picked up guitar during a temporary ban until we crash and burn because we become so powerful and many Americans will be classic Sina - SUPERB!﻿ +sina-drums Looking forward to hearing your album too. They will always keep them forever safe & in my heart.... Sammy Hagar opened for them.

We'll stop here.

### Afterword

My [FsMarkov](https://github.com/taylorwood/FsMarkov) project makes some assumptions about english syntax, i.e. sentences start with a capitalized letter and end with punctuation. YouTube comments don't always follow those rules, so I may do some refactoring to be more accommodating. Probably not. Sorry!
