---
layout: post
title:  "Generating Markov text from YouTube comments"
date:   2015-12-17 09:08:00
tags:	markov chain text generator YouTube comments web scraping
---
Let's build on my post from a few months ago about [generating weird text with Markov chains]({% post_url 2015-07-04-markov-text %}). This post will focus on a purely [GIGO](https://en.wikipedia.org/wiki/Garbage_in,_garbage_out) workflow — instead of feeding our Markov text generator with literary classics, we're going to feed it [YouTube comments](http://www.newyorker.com/humor/daily-shouts/comments-feeling-youtube).

*I've left some uninteresting but important bits of code out for brevity. All the code for this post can be found on [GitHub](https://github.com/taylorwood/YouTudes).*

## Request for comments
Our plan is to scrape the comment text from videos that are returned for a given search query. We can use the very convenient HTTP functionality in [F# Data](http://fsharp.github.io/FSharp.Data/) for working with [YouTube.com](https://www.youtube.com).

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

`HtmlDocument` allows easy access to the response's DOM elements. We can find the container element for each video result and their respective video URLs with the following code:
{% highlight fsharp %}
let videoContainers = searchDoc.Descendants (fun n -> n.HasClass("yt-lockup-title"))
let videoUrls =
    videoContainers
    |> Seq.collect (fun n -> n.Elements("a"))
    |> Seq.choose (fun n -> n.TryGetAttribute("href"))
    |> Seq.map (fun attr -> host + attr.Value())
    |> Seq.where (fun url -> url.Contains("/watch?"))
    |> Seq.distinct
{% endhighlight %}

YouTube's tasteless comments are loaded via AJAX request after the video page loads. The request needs to be POSTed with some particular values or it won't work. The following function handles that and will return the raw string JSON response:
{% highlight fsharp %}
let getCommentJson videoUrl jsonUrl =
    let body =
        FormValues [("session_token", sessionToken); ("client_url", videoUrl)]
    let headers =  [("Referer", videoUrl)
                    ("Content-Type", "application/x-www-form-urlencoded")
                    ("DNT", "1")
                    ("Origin", host)]
    Http.RequestString(jsonUrl, httpMethod = "POST", body = body, headers = headers, cookieContainer = cookies)
{% endhighlight %}

But before we can call that function, we need the `jsonUrl`, which can be simply formatted like this:
{% highlight fsharp %}
let getCommentUrl videoId = sprintf "%s/watch_fragments_ajax?v=%s&tr=time&distiller=1&frags=comments&spf=load" host videoId
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
    |> Seq.map getVideoCommentJson
    |> Seq.map JsonValue.Parse
    |> Seq.map
        (fun j ->
            maybe {
                let! body = j.TryGetProperty("body")
                let! html = body.TryGetProperty("watch-discussion")
                return html
            })
    |> Seq.choose id
{% endhighlight %}

The JSON response are basically just containers for embedded HTML. We can use `HtmlDocument` again to parse those fragments, get at the comment containers, then extract their inner text:
{% highlight fsharp %}
let getCommentText (body: JsonValue) =
    let commentHtml = HtmlDocument.Parse(body.AsString())
    commentHtml.Descendants (fun n -> n.HasClass "comment-text-content")
    |> Seq.map HtmlNodeExtensions.InnerText

let comments = commentBodies |> Seq.map getCommentText |> Seq.collect id
{% endhighlight %}

The `comments` value is now a sequence of all comments for all videos we found.

## Garbage compactor
Now that we have all this inane text, let's use it to generate some *insane* text.

> You'll need the code from my previous post for this, which can be found on [GitHub](https://github.com/taylorwood/FsMarkov).

We'll combine all our comments into a big string and feed it into our `MarkovTextBuilder`:
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

My [FsMarkov](https://github.com/taylorwood/FsMarkov) project makes some assumptions about english syntax, i.e. sentences start with a capitalized letter and end with punctuation. YouTube comments don't always follow those rules, so I may do some refactoring to be a little more lenient. Probably not. Sorry! 