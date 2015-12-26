---
layout: post
title:  "Generating Markov text from YouTube comments"
date:   2015-12-27 12:00:00
tags:	markov chain text generator YouTube comments web scraping fparsec xpath
---
Let's build on my post from a few months ago about [generating weird text with Markov chains]({% post_url 2015-07-04-markov-text %}). This post will focus on a purely [GIGO](https://en.wikipedia.org/wiki/Garbage_in,_garbage_out) workflow — instead of feeding our Markov text generator with literary classics, we're going to feed it [YouTube comments](http://www.newyorker.com/humor/daily-shouts/comments-feeling-youtube).

We'll also use [FParsec](http://www.quanttec.com/fparsec/) to parse basic [XPath](https://en.wikipedia.org/wiki/XPath) queries for evaluation against [FSharp.Data's](http://fsharp.github.io/FSharp.Data/) `HtmlDocument`.

Many thanks to [Sergey Tihon](https://twitter.com/sergey_tihon) for organizing another great [F# Advent Calendar](https://sergeytihon.wordpress.com/2015/10/25/f-advent-calendar-in-english-2015/) this year! Check out the [#FsAdvent](https://twitter.com/hashtag/FsAdvent?src=hash) tag on Twitter for more posts.

*I've left some uninteresting bits of code out for brevity, but it can all be found in my [YouTudes](https://github.com/taylorwood/YouTudes) on GitHub.* 

## Request for comments

Our plan is to scrape the comment text for a given search query's videos. We can use the very convenient HTTP functionality in [FSharp.Data](http://fsharp.github.io/FSharp.Data/) for working with [YouTube.com](https://www.youtube.com).

> Google provides a [YouTube Data API](https://developers.google.com/youtube/v3/docs/) for things like this, but we'll take the dumb approach and depend on specific markup in YouTube's HTML that may break suddenly. Use their API if you're serious about harvesting garbage comments on an ongoing basis.

First, we'll issue the search `query`, parse the session token from the response, then load the response in a `HtmlDocument`:
{% highlight fsharp %}
let searchHtml = Http.RequestString(searchUrl, cookieContainer = cookies)
let sessionToken = // required for comment JSON requests
    match getSessionToken searchHtml with
    | Some t -> t
    | None -> failwith "Failed to parse session token from HTML"
let searchDoc = HtmlDocument.Parse searchHtml
{% endhighlight %}

### And now for something completely different

I figured an unnecessary and fun diversion for this task would be to implement simple XPath queries to make our scraper more concise. `HtmlDocument` enables easy access to the response's DOM elements, but might be unwieldy if the element(s) you're after are deeply nested and/or vaguely decorated. A single XPath query can get us pretty far in just one line, e.g. `//table[@id='results']/tbody/tr//a[@href]`, and most browsers will generate XPaths for you from their developer tools.

We'll need two things to pull this off:

1. A parser to turn that XPath string into a useful structure
2. Some code to evaluate that structure against a `HtmlDocument` and return any matching elements

#### Path parsing

Let's see how easy it is to define a parser for a simple subset of XPath. First, we'll define the types we want our parser to produce:
{% highlight fsharp %}
type XPathPart = Axis * NodeTest * Predicate list
type Axis =
    | Child
    | DescendantOrSelf
type NodeTest = string
type Operator =
    | Equals
    | NotEquals
type Predicate = {
    Attribute: string
    Filter: (Operator * string) option }
{% endhighlight %}

Given a valid XPath, our parser should give us a list of `XPathPart` -- one for each XPath segment.

[FParsec](http://www.quanttec.com/fparsec/) provides many general [parser primitives](http://www.quanttec.com/fparsec/reference/primitives.html) that we can compose into larger primitives for our needs. The following sections of code build up `Axis`, `NodeTest`, and `Predicate` parsers that will ultimately constitute a `XPathPart` parser:
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

We can compose those parsers into a unified XPath parser like this:
{% highlight fsharp %}
// parses one XPath segment ("/div[@id='hey'][@disabled]")
let xPathPart =
    let predicates = many predicate
    pipe3 axis nodeName predicates getXPathPart
// parses an entire XPath comprised of 1-to-many segments ("/div/ol/li/*")
let xPathParts = (many1 xPathPart) .>> eof
{% endhighlight %}
`pipe3` combines our three primary component parsers and pipes their output to the unlisted `getXPathPart` function that simply constructs a `XPathPart` from their values. Adding the `eof` parser on the end ensures the entire string can be parsed as a XPath; no extraneous funny business allowed.

Even though we've written no error handling code, FParsec gives nice error messages for bad input:
{% highlight ruby %}
Failure: Error in Ln: 1 Col: 9
div/ul/*[💩]
        ^
Expecting: end of input, '*', '/', '//' or '[@'
{% endhighlight %}

#### Walking the path

Now that we can parse a simple XPath, we can evaluate it against a `HtmlDocument`. The entry point is the `evaluate` function below. It folds over a `XPathPart list` from the context of a given `HtmlNode`, each step evaluating one `XPathPart` and passing on any nodes that meet the requirements.

{% highlight fsharp %}
let satisfiesPredicate (n: HtmlNode) pred =
    match n.TryGetAttribute(pred.Attribute) with
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
        | DescendantOrSelf ->
            if name <> "*" then n.DescendantsAndSelf(name) else n.DescendantsAndSelf()
            |> List.ofSeq
    let isMatch n = preds |> List.forall (satisfiesPredicate n)
    searchNodes |> List.where isMatch

let evaluate xPath node =
    let folder nodes part = nodes |> List.collect (evaluate' part)
    xPath |> Seq.fold folder [node]
{% endhighlight %}

### Back to scraping

Now we can get each video's URL with a handy XPath query:
{% highlight fsharp %}
let videoUrls =
    searchDoc.Select("//*[@class='yt-lockup-title']/a[@href]")
    |> Seq.map (fun a -> host + a.AttributeValue("href"))
    |> Seq.where (fun url -> url.Contains("/watch?"))
{% endhighlight %}

YouTube's tasteless comments are loaded via AJAX request after the video page loads. The server requires certain values be POSTed, otherwise it won't give us the goods. The following function issues the request and returns the raw JSON string response:
{% highlight fsharp %}
let requestComments videoUrl jsonUrl =
    let body = FormValues [("session_token", sessionToken); ("client_url", videoUrl)]
    let headers = [("Referer", videoUrl); ("Origin", host)]
    printfn "Requesting comments for %s..." jsonUrl
    Http.RequestString(jsonUrl, httpMethod = "POST", body = body, headers = headers, cookieContainer = cookies)
{% endhighlight %}

Next we need a function to parse the JSON response and give us the part we're interested in:
{% highlight fsharp %}
let getComments videoUrl =
    let getCommentUrl videoId = sprintf "%s/watch_fragments_ajax?v=%s&tr=time&distiller=1&frags=comments&spf=load" host videoId
    match getVideoId videoUrl with
    | Some id ->
        let json = getCommentUrl id |> requestComments videoUrl |> JsonValue.Parse
        maybe {
            let! body = json.TryGetProperty("body")
            let! html = body.TryGetProperty("watch-discussion")
            return html
        }
    | None -> failwithf "Failed to parse video ID from video URL %s" videoUrl
{% endhighlight %}

The JSON property contains the comments as embedded HTML. We can use `HtmlDocument` again to parse the HTML fragments, get at the comment containers, and extract their inner text:
{% highlight fsharp %}
let parseComments (body: JsonValue) =
    let commentHtml = HtmlDocument.Parse(body.AsString())
    commentHtml.Select("//*[@class='comment-text-content']")
    |> Seq.map HtmlNode.innerText

let comments = videoUrls |> Seq.choose getComments |> Seq.collect parseComments
{% endhighlight %}

The `comments` value is now a sequence of all comments for all videos we found.

## Garbage fountain

Now that we have all this inane text, let's use it to generate some insane text. We'll concatenate all the comments and feed them to `MarkovTextBuilder`. You'll need the [FsMarkov](https://github.com/taylorwood/FsMarkov) code from my [previous post]({% post_url 2015-07-04-markov-text %}) for this.

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

What happens if we feed it comments from **mariah carey christmas** videos?

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

What happens if we feed it a mashup of comments from **donald trump** and **more than a feeling** videos?

> ESTABLISHMENT?? There goes CNN again. My sister was very close with him﻿ played it in your lap.﻿ I'm half expecting Will Ferrell to appear mid-song with more cowbell.﻿ +random666777 NO, why spoil a good time.﻿ I think you are going to win big ! Maybe the rest of US repugnant morality and hypocrisy if youd like; basically however, the USA split into 50 separate nations so ppl would finally stop moaning about "this cancer in Washington" and get the best". I picked up guitar during a temporary ban until we crash and burn because we become so powerful and many Americans will be classic Sina - SUPERB!﻿ +sina-drums Looking forward to hearing your album too. They will always keep them forever safe & in my heart.... Sammy Hagar opened for them.

We'll stop before things get out of hand.

## Afterthoughts

My [FsMarkov](https://github.com/taylorwood/FsMarkov) project makes some simplifying assumptions about english syntax, i.e. sentences start with a capitalized letter and end with punctuation. YouTube comments don't always follow those rules, so the output may not reflect the full range of human emotion they contain.
