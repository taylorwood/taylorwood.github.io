---
layout: post
title:  "Working CryptoPals Set 2 challenges in F#"
date:   2016-10-23 12:00:00
tags:   crypto fsharp
---
My [previous post]({% post_url 2016-10-21-cryptopals-pt1 %}) covered approaches to challenges in Set 1 of CryptoPals challenges. This post will be more of the same for Set 2.

## [Challenge 9](https://cryptopals.com/sets/2/challenges/9)

### Implement PKCS#7 padding

```ocaml
let pad bytes length =
    let diff = length - Array.length bytes
    let pad = Seq.initInfinite (fun _ -> byte diff)
    Seq.append bytes pad |> Seq.truncate length

let unpadded = "YELLOW SUBMARINE" |> stringToBytes

let padded = pad unpadded 20
```

Since `unpadded` is 16 bytes, 4 less than the desired length of 20, we pad the end of it with byte values equal to 4. If `unpadded` was 15 bytes, we'd be 5 short and would need to paid with byte values of 5.

## [Challenge 10](https://cryptopals.com/sets/2/challenges/10)

### Implement CBC mode

We'll need to implement [CBC mode](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Cipher_Block_Chaining_.28CBC.29) by hand using the ECB approach from the previous set. CBC mode differs from ECB in that the ciphertext output of each block is XOR'd with the plaintext input of the next block. Since the first plaintext block to be encrypted has no prior ciphertext block, an Initialization Vector (IV) used to bootstrap the process.

```ocaml
open System.Security.Cryptography
let cryptAlg key =
    let aes = new AesManaged()
    aes.Mode <- CipherMode.ECB
    aes.Key <- key
    aes.Padding <- PaddingMode.None
    aes

let blockSize = 16

let encrypt (bytes: byte[]) key =
    use aes = cryptAlg key
    let encryptor = aes.CreateEncryptor(aes.Key, aes.IV)
    [| for block in bytes |> Array.chunkBySize blockSize do
        let padded = pad block blockSize |> Array.ofSeq
        let output = Array.create padded.Length 0uy
        encryptor.TransformBlock(padded, 0, padded.Length, output, 0) |> ignore
        yield! output |]

encrypt (stringToBytes "YELLOW SUBMARINE") (stringToBytes "YELLOW SUBMARINE")
```

Above, we've defined the plain EBC `encrypt` function again. This will be used in our CBC encrypt function below. Notice `prevBlock` is used to store the IV initially, and then the ciphertext of each successive block, after it's XOR'd with the plaintext input.

```ocaml
let encryptCBCWithIV (iv: byte[]) (bytes: byte[]) key =
    let mutable prevBlock = iv
    [| for block in bytes |> Array.chunkBySize blockSize do
        let padded = pad block blockSize |> Array.ofSeq
        let xorBytes = xorBytes padded prevBlock |> Array.ofSeq
        let blockCipher = encrypt xorBytes key
        prevBlock <- blockCipher
        yield! blockCipher |]
```

And some very similar functions for decrypting.

```ocaml
let decrypt (aesBytes: byte[]) key =
    use aes = cryptAlg key
    let decryptor = aes.CreateDecryptor(aes.Key, aes.IV)
    let output = Array.create aesBytes.Length 0uy
    decryptor.TransformBlock(aesBytes, 0, aesBytes.Length, output, 0) |> ignore
    output

let decryptCBC (bytes: byte[]) key =
    let mutable prevBlock : byte[] = Array.create blockSize 0uy
    [| for block in bytes |> Array.chunkBySize blockSize do
        let padded = pad block blockSize |> Array.ofSeq
        let blockCipher = decrypt padded key
        let xorBytes = xorBytes blockCipher prevBlock |> Array.ofSeq
        prevBlock <- padded
        yield! xorBytes |]
```

Now we should be able to decrypt the challenge text using CBC mode. I think it's just more Vanilla Ice lyrics.

```ocaml
let cbcBytes = readBase64Bytes (__SOURCE_DIRECTORY__ + "/Challenge10.txt")
decryptCBC cbcBytes (stringToBytes "YELLOW SUBMARINE") |> bytesToString |> printfn "%s"
```

## [Challenge 11](https://cryptopals.com/sets/2/challenges/11)

### An ECB/CBC detection oracle

This one asks you to create an encrypt function that pads the plaintext and randomly chooses ECB/CBC mode. Then write a function to determine which mode it's using. The idea here is to use big empty plaintext buffers to cause ECB mode to produce identical cipher blocks, then look for those.

```ocaml
let rnd = Random()
let coinflip () = rnd.NextDouble() < 0.5
let genAesKey () =
    let bytes = Array.create 16 0uy
    rnd.NextBytes bytes
    bytes

let crazyCrypt bytes =
    let leftPad = Array.create (rnd.Next(5,10)) 0uy
    let rightPad = Array.create (rnd.Next(5,10)) 0uy
    let bytes = Array.concat [| leftPad; bytes; rightPad |]
    let rndKey = genAesKey ()
    if coinflip ()
    then encrypt bytes rndKey
    else
        let iv = Array.create blockSize 0uy
        rnd.NextBytes iv
        encryptCBCIV iv bytes rndKey

let isECB encrypt =
    let bytes = encrypt (Array.create (blockSize * 6) 0uy) // use zeroed buffers to expose ECB repetition
    let blocks = bytes |> Array.chunkBySize blockSize
    blocks.Length > (blocks |> Array.distinct |> Array.length) // if there were duplicate blocks, ECB was used

isECB crazyCrypt
```

## [Challenge 12](https://cryptopals.com/sets/2/challenges/12)

### Byte-at-a-time ECB decryption

I won't repeat the official description, but you should read it. Essentially you want an encrypt function like `AES-128-ECB(your-string || unknown-string, random-key)` where `unknown-string` is used inside the function but not _known_ to the attacker.

>It turns out: you can decrypt `unknown-string` with repeated calls to the oracle function!

```ocaml
module Oracle12 =
    let append = Convert.FromBase64String "Um9sbGluJyBpbiBteSA1LjAKV2l0aCBteSByYWctdG9wIGRvd24gc28gbXkgaGFpciBjYW4gYmxvdwpUaGUgZ2lybGllcyBvbiBzdGFuZGJ5IHdhdmluZyBqdXN0IHRvIHNheSBoaQpEaWQgeW91IHN0b3A/IE5vLCBJIGp1c3QgZHJvdmUgYnkK"
    let rndKey = genAesKey ()
    let encrypt bytes = encrypt (Array.append bytes append) rndKey
```

The first thing we need to know is the block size produced by the function. We kinda already know this, but it's important to figure it out manually. This is a pretty goofy way of finding it:

```ocaml
let growingKeys = [for i in 1 .. 16 do yield Array.create i 0uy]
let lenDiffs = // look for ciphertext size diffs given diff inputs
    growingKeys
    |> List.map (Oracle12.encrypt >> Array.length)
    |> List.pairwise
    |> List.map (fun (x,y) -> abs(x-y))
    |> List.where (fun x -> x > 0)
    |> List.distinct
```

Per instructions, we should check the mode of the `encrypt` function even though we know what it is.

```ocaml
isECB Oracle12.encrypt
```

Now for the interesting part. We know the block size, and we know that the `unknown-string` gets appended to `your-string` in the encrypt function. What would happen if we sent a plaintext input that was one shorter than the block size? The first character of secret string would be appended to the end of the first block.

Using a short buffer we can ensure the last character of the plaintext input will be the first character of the secret string -- we can force it to be the last character of an otherwise "empty" block -- so we can now brute force the first block with every byte, and compare it to the encrypted block of the "short" buffer. If the two blocks match, we've identified the first character of the secret string!

```ocaml
// how many blocks of secret text are there? find out by encrypting an empty buffer
let numBlocks = (Oracle12.encrypt [||] |> Array.length) / blockSize

let getBlock blockNum = Array.chunkBySize blockSize >> Array.tryItem blockNum

let secret = ResizeArray()
for blockNum in 0 .. numBlocks - 1 do
    for blockOffset in 0 .. blockSize - 1 do
        let shorty = Array.create (blockSize - (blockOffset + 1)) 0uy
        let bruteCryptBlocks = seq {
            for b in Byte.MinValue .. Byte.MaxValue do
                let plaintext = Array.concat [| shorty; secret.ToArray(); [|b|] |]
                let crypted = Oracle12.encrypt plaintext
                yield b, getBlock blockNum crypted }
        let shortyCryptBlock = shorty |> Oracle12.encrypt |> getBlock blockNum
        bruteCryptBlocks
        |> Seq.tryPick (fun (guessByte, cipherBlock) ->
            if cipherBlock = shortyCryptBlock
            then Some guessByte
            else None)
        |> Option.iter (secret.Add)

secret.ToArray() |> bytesToString |> printfn "%A"
```

I took an iterative approach to this problem, maybe easier to understand than purely functional. If a brute forced block matches the ciphertext of the short block, we have a match. For each offset of the block size, we shorten the input plaintext and accumulate the guessed key characters -- and this is repeated for each block. Each block yields 16 characters, except when padded.

## More to come

Check back later for updates.
