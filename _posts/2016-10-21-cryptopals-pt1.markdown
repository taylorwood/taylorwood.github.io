---
layout: post
title:  "Working CryptoPals Set 1 challenges in F#"
date:   2016-10-21 12:00:00
tags:   crypto fsharp
---
The [CryptoPals challenges](https://cryptopals.com) are a great hands-on introduction to cryptography and [cryptanalysis](https://en.wikipedia.org/wiki/Cryptanalysis). This post will outline some of my approaches to the challenges, written in F#.

The code is available on [GitHub](https://github.com/taylorwood/FsCryptoPals).

My solutions for Set 2 challenges are in [this post]({% post_url 2016-10-23-cryptopals-pt2 %}).

## [Challenge 1](https://cryptopals.com/sets/1/challenges/1)

### Convert hexadecimal to base 64

This one is "easy" if you know [how hexadecimal representation works](https://en.wikipedia.org/wiki/Hexadecimal#Using_0.E2.80.939_and_A.E2.80.93F). We'll go from hex to bytes, then from bytes to base 64.

This function converts one hex `char` to a `byte`. There are BCL methods to do that for you, but redundancy can be educational.

```ocaml
let hexToByte = function
    | '0' -> 0uy | '1' -> 1uy | '2' -> 2uy | '3' -> 3uy | '4' -> 4uy | '5' -> 5uy | '6' -> 6uy | '7' -> 7uy | '8' -> 8uy | '9' -> 9uy
    | 'a' -> 10uy | 'b' -> 11uy | 'c' -> 12uy | 'd' -> 13uy | 'e' -> 14uy | 'f' -> 15uy
    | _ -> failwith "Invalid hexadecimal character"
```

Astute observers might say "that's only using half the `byte`, stop wasting my electricity" but that's why we have this function:

```ocaml
let hexToBytes (hexChars: string) : byte array =
    [| for i in 0 .. 2 .. hexChars.Length - 1 do
        let hi = hexToByte hexChars.[i]
        let lo = hexToByte hexChars.[i+1]
        yield (hi <<< 4) ||| lo |]
```

A byte is 8 bits. A hex digit represents 4 bits, affectionately referred to as a nibble. You can fit two hex digits in a byte by storing one hex digit on the left 4 bits, and another on the right. Shifting `hi <<< 4` scoots the bits over four places, e.g. `0000 1010` becomes `1010 0000`. OR'ing with the other 4 bits simply sets the low bits, e.g. `1010 0000 ||| 0000 1100` becomes `1010 1100`. Can you *smell* the silicon yet? I didn't bother implementing base 64 conversion.

```ocaml
let bytes = hexToBytes "49276d206b696c6c696e6720796f757220627261696e206c696b65206120706f69736f6e6f7573206d757368726f6f6d"
Convert.ToBase64String bytes // expected: SSdtIGtpbGxpbmcgeW91ciBicmFpbiBsaWtlIGEgcG9pc29ub3VzIG11c2hyb29t
```

## [Challenge 2](https://cryptopals.com/sets/1/challenges/2)

### XOR two hexadecimal strings

This challenge calls for XOR'ing two equal-length buffers. Pretty simple function here, just zipping two `byte seq` and applying the XOR operator to each pair. Define a helper function to print a byte buffer in hexadecimal, and apply our functions.

```ocaml
let xorBytes b1 b2 =
    Seq.zip b1 b2 |> Seq.map (fun (x,y) -> x ^^^ y)

let printBytesHex (bytes: byte seq) = bytes |> Seq.map (sprintf "%02x") |> String.concat "" |> printfn "%s"

let hexBytesA = hexToBytes "1c0111001f010100061a024b53535009181c"
let hexBytesB = hexToBytes "686974207468652062756c6c277320657965"
printBytesHex (xorBytes hexBytesA hexBytesB)
```

## [Challenge 3](https://cryptopals.com/sets/1/challenges/3)

### Decipher a single-character XOR cipher

Here's where it gets interesting! Given a hexadecimal string, find a character that correctly decrypts the ciphertext.

```ocaml
let cryptBytes = hexToBytes "1b37373331363f78151b7f2b783431333d78397828372d363c78373e783a393b3736"

let xorBytesWith (bytes: byte[]) value = bytes |> Array.map (fun b -> b ^^^ value)

let brute cryptBytes = seq {
    for b in 0uy .. 255uy do
        yield xorBytesWith cryptBytes b |> bytesToString }
```

`xorBytesWith` XOR's each byte in `bytes` with a given byte `value`. We'll use that in `brute` which produces a sequence of XOR'd buffers, one for each possible `byte` value as the key. One of those XOR'd buffers will be our decrypted ciphertext, but how can we identify which one?

>Devise some method for "scoring" a piece of English plaintext. Character frequency is a good metric. Evaluate each output and choose the one with the best score.

```ocaml
let isLikelyMatch (str: string) =
    let isPunctuation = function ',' | ''' | '.' | '?' | '!' -> true | _ -> false
    let isTypical c = Char.IsLetter c || c = ' ' || isPunctuation c
    let letters = str |> Seq.where Char.IsLetter |> Seq.length
    let lowers = str |> Seq.where Char.IsLower |> Seq.length
    let ratio check =
        let n = str |> Seq.where check |> Seq.length
        float n / float str.Length
    (ratio isPunctuation) < 0.2 &&
    (ratio isTypical) > 0.9 &&
    (float letters / float str.Length) > 0.5 &&
    (float lowers / float letters) > 0.5
```

This function uses some heuristics on ratios of character types in a string, and returns true if the string seems to be composed of human readable text: it should be mostly alphabetical, and mostly lower case, leaving very little margin for the kinds of "special" characters that you'll see in improperly decrypted ciphertext.

Now we can find any likely decryptions, with one obvious winner. We could make our `isLikelyMatch` function much smarter, especially by adding some analysis of letter-frequency and whitespace, but this is good enough to move on.

```ocaml
brute cryptBytes |> Seq.where isLikelyMatch |> Seq.iter (printfn "%s")
```

```
Cooking MC's like a pound of bacon                                                      
Bnnjhof!LB&r!mhjd!`!qntoe!ng!c`bno                                                      
Dhhlni`'JD t'knlb'f'whric'ha'efdhi
```

## [Challenge 4](https://cryptopals.com/sets/1/challenges/4)

### Find the encrypted string

We can solve this challenge just by reusing that code to find encrypted lines in a larger file.

```ocaml
let cryptStrings = IO.File.ReadAllLines(__SOURCE_DIRECTORY__ + "/Challenge4.txt")
let possibleDecrypts =
    cryptStrings
    |> Seq.map (hexToBytes >> brute)
    |> Seq.indexed
    |> Seq.map (fun (i,cs) -> i, cs |> Seq.where isLikelyMatch |> Array.ofSeq)
    |> Seq.where (fun (i,cs) -> cs.Length > 0)
possibleDecrypts |> Seq.iter (printfn "%A")
```

This will brute force each line in the file, looking for lines that are likely readable, producing just one line as expected: `(170, [|"Now that the party is jumping"|])`

## [Challenge 5](https://cryptopals.com/sets/1/challenges/5)

### Implement repeating-key XOR

We need to implement stronger XOR encryption using a multi-character key. We're given the plaintext and key. The key will be repeated as many times as necessary to XOR each character in the plaintext.

```ocaml
let stanza = """Burning 'em, if you ain't quick and nimble
I go crazy when I hear a cymbal"""
let stanzaBytes = stringToBytes stanza
let key = "ICE".ToCharArray() |> Array.map byte
```

We need a function to repeat the key to desired length, e.g. `ICE` repeated to length 7 becomes `ICEICEI`.

```ocaml
let repeatKey length (key: 'a array) = seq {
    for i in 0 .. length - 1 do yield key.[i % key.Length] }
let repetitiveKey = repeatKey stanzaBytes.Length key

let encryptedStanza = Seq.zip stanzaBytes repetitiveKey |> Seq.map (fun (x,y) -> x ^^^ y)
printBytesHex encryptedStanza
```

Finally `zip` the plaintext bytes with the key bytes, XOR each pair, and we've rolled our own weak crypto.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">when you roll your own crypto <a href="https://t.co/nRdwPhGpeG">pic.twitter.com/nRdwPhGpeG</a></p>&mdash; nasty hombre 🔑 (@mshelton) <a href="https://twitter.com/mshelton/status/733484058752221184">May 20, 2016</a></blockquote> <script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

## [Challenge 6](https://cryptopals.com/sets/1/challenges/6)

### Commencing cryptanalysis

If you intercept some repeating-key XOR ciphertext, how might you attack it? One clever approach exploits the nature of repeating keys: we can analyze repeating patterns in the ciphertext to make educated guesses as to the _length_ of the key.

As advised by CryptoPals, we'll first define a function to get the [Hamming distance](https://en.wikipedia.org/wiki/Hamming_distance) between two bytes, i.e. the number of bits that _differ_ in the same positions in both bytes, another place where XOR comes in handy.

```ocaml
let bitDist (x: byte, y: byte) =
    let countBits b =
        let mutable set = 0
        for i in 0 .. 7 do
            let isSet = (b >>> i) &&& 1uy = 1uy
            if isSet then set <- set + 1
        set
    countBits (x ^^^ y)
```

If we think the key might be _N_ bytes, using this function we can measure the distance between the first _N_ bytes of the ciphertext against the second _N_ bytes. We'll do the same for many key lengths/values of _N_.

```ocaml
let keySizes = [2 .. 40] // guessing at range of key lengths

let getAverageBlockDists keySize buffer =
    buffer
    |> Seq.chunkBySize keySize // blocks of key size
    |> Seq.chunkBySize 2 // two-block pairs to compare
    |> Seq.truncate 6 // compare 7 pairs at most
    |> Seq.map (fun bp -> Array.zip bp.[0] bp.[1] |> Array.sumBy bitDist |> float)
    |> Seq.average

let repeatKeyCryptBytes =
    IO.File.ReadAllLines(__SOURCE_DIRECTORY__ + "/Challenge6.txt")
    |> String.concat ""
    |> Convert.FromBase64String

let keySizeBlockDists = keySizes |> List.map (getAverageBlockDists repeatKeyCryptBytes)
let blockDistsNorm = // fst is key size, snd is normalized block distance
    List.zip keySizes keySizeBlockDists
    |> List.map (fun (ks,dist) -> ks, dist / float ks) // normalize distances by key size
    |> List.sortBy snd
```

>The KEYSIZE with the smallest normalized edit distance is probably the key. You could proceed perhaps with the smallest 2-3 KEYSIZE values. Or take 4 KEYSIZE blocks instead of 2 and average the distances.

In hindsight, I found their initially suggested approach of taking the edit distance of just the first two blocks didn't produce accurate results. After taking their advice of averaging successive blocks' edit distance, I found the more successive chunks I included the lower the normalized edit distance of the _correct_ key size. I'm not sure if the initial suggestion is intentionally weak, or if there's something subtly wrong with my code!

Anyway, we now have some guesses as to the _length_ of a possible key. Here are our top guessed key lengths, sorted by normalized edit distance descending. A 29-character key seems most likely.

```[(29, 2.683908046); (5, 2.766666667); (15, 2.966666667); (31, 3.010752688);]```

Again, the idea is to exploit the nature of a repeating key by taking "slices" of the ciphertext for each key character offset. We think our key is 29 characters, so in order to guess the _first_ character of the key we take the first and every 29th character from the ciphertext then analyze them together. To guess the _second_ character of the key, we'd take the second and every 29th character from there, etc. If our key length guess is right, the same character will have been used to XOR each of those plaintext characters at the repsective offsets. Even when looking at a decrypted "slice" the resulting text should still resemble a typical distribution of character types---hopefully so that `isLikelyMatch` will still recognize it!

```ocaml
let findLikelyKeyCombos buffer keySize =
    let chunks = buffer |> Array.chunkBySize keySize
    let transpose (chunks: 'a [][]) offset =
        chunks |> Array.where (fun c -> offset < c.Length) |> Array.map (fun c -> c.[offset])
    let transposedChunks = Array.init keySize (transpose chunks)
    let findLikelyKeyChars transposed =
        transposed
        |> brute'
        |> Seq.where (snd >> isLikelyMatch)
        |> Seq.map (fun (key,text) -> (bytesToString [|key|]))
        |> List.ofSeq
    transposedChunks
    |> Seq.map findLikelyKeyChars
    |> Seq.where (not << List.isEmpty)
    |> List.ofSeq
```

The `transpose` function is doing the "slicing" of the buffer for a given key offset. `findLikelyKeyChars` is brute forcing all the transposed/sliced chunks and returning any key characters that produce "good looking" plaintext.

```
[[["T"]; ["e"]; ["r"]; ["m"]; ["i"]; ["n"]; ["a"]; ["t"; "u"]; ["o"]; ["r"];  [" "]; ["X"]; [":"]; [" "]; ["B"]; ["r"]; ["i"]; ["n"]; ["g"]; [" "]; ["t"]; ["h"; "i"; "o"]; ["e"]; [" "]; ["n"]; ["o"]; ["i"; "n"]; ["s"]; ["e"]]]
```

Each element in the list represents one key offset, and each element is itself a list that represents guesses of what the character value might be at that offset. If you squint hard enough, you can already see the key even though a few characters have two or three potential values.

Using this helpful [cartesian product function](http://www.fssnip.net/2A), we can flatten that list into all possible combinations of those lists.

```ocaml
let possibleKeyCombos =
    keySizes
    |> Seq.map (findLikelyKeyCombos repeatKeyCryptBytes)
    |> Seq.where (not << List.isEmpty)

let printPossibleKeyCombos possibleKeys =
    for possibleKeyCombos in possibleKeys do
        cartesian possibleKeyCombos |> List.map (String.concat "") |> Seq.iter (printfn "%s")

printPossibleKeyCombos possibleKeyCombos
```

And print out all likely key phrases:

```
Terminator X: Bring the nonse
Terminauor X: Bring the nonse
Terminator X: Bring tie nonse
Terminauor X: Bring tie nonse
Terminator X: Bring toe nonse
Terminauor X: Bring toe nonse
Terminator X: Bring the noise
Terminauor X: Bring the noise
Terminator X: Bring tie noise
Terminauor X: Bring tie noise
Terminator X: Bring toe noise
Terminauor X: Bring toe noise
```

Assuming Terminator isn't bringing _toe_ noise, there's only one good candidate in that list so let's try decrypting with it.

```ocaml
let keyBytes = "Terminator X: Bring the noise".ToCharArray() |> Array.map byte
let repeatedKey = repeatKey repeatKeyCryptBytes.Length keyBytes
let decrypted =
    xorBytes repeatKeyCryptBytes repeatedKey
    |> Array.ofSeq
    |> Text.Encoding.ASCII.GetString
```

That should look familiar, it's basically the same thing we did in Challenge 5. XOR'ing the ciphertext bytes _again_ with the correct key bytes gives us the original values, which are unfortunately the lyrics to a Vanilla Ice song.

## [Challenge 7](https://cryptopals.com/sets/1/challenges/7)

### Figure out how to use an AES implementation

```ocaml
open System.Security.Cryptography
let decrypt aesBytes key =
    use aes = new AesManaged()
    aes.Mode <- CipherMode.ECB
    aes.Key <- stringToBytes key
    let decryptor = aes.CreateDecryptor(aes.Key, aes.IV)
    use mem = new IO.MemoryStream(aesBytes)
    use decryptStream = new CryptoStream(mem, decryptor, CryptoStreamMode.Read)
    use readStream = new IO.StreamReader(decryptStream)
    readStream.ReadToEnd()
    
let aesBytes = IO.File.ReadAllLines(__SOURCE_DIRECTORY__ + "/Challenge7.txt") |> String.concat "" |> Convert.FromBase64String
decrypt aesBytes "YELLOW SUBMARINE"
```

## [Challenge 8](https://cryptopals.com/sets/1/challenges/8)

### Detect AES ECB encrypted text

Because in ECB mode the same plaintext input per-block will produce the same ciphertext output, we'll assume that the presence of "duplicate" blocks indicates AES ECB encrypted text.

```ocaml
let hexLines = IO.File.ReadAllLines(__SOURCE_DIRECTORY__ + "/Challenge8.txt") |> Array.map hexToBytes
hexLines
|> Array.map (Array.chunkBySize 16 >> Array.distinct >> Array.length)
|> Seq.iteri (printfn "%d: %A") // line 133 has only 7/10 distinct blocks
```

## More challenges

My solutions for Set 2 challenges are in [this post]({% post_url 2016-10-23-cryptopals-pt2 %}).