# Parser combinators

When tasked to extract information from a string, most seasoned developers - especially the ones with some kind of Linux background - will resort to regular expressions. While regular expressions are incredibly powerful and very well suited for most parsing jobs, they don't scale very well. With increasing complexity they tend to become very cryptic, if not unreadable.

Another alternative is to use parser combinators that combine small atomic parsers to build a complex ruleset.

Let's see how parser combinators can be used in an (admittedly simple) context. One of our customers runs a fascinating webservice: Using a easy-to-understand URL scheme, you can remotely control a rendering engine that can effectively combines hundreds of different car parts in order to render a photorealistic image of a specific car configuration. One task in a recent project was to extract car-configuration-specific data from a webservice URL and I solved it in just a few lines of easily maintainable code using JParsec.

Our customer's API is confident, so I have to anonymize the URLs.

## The webservice

As mentioned before, the URL scheme is quite straightforward, but it seems to be more human-friendly than machine-friendly, but more about that later. It consists of the following parts:

1. The country version's iso code
1. Car metadata (model is mandatory, body and grade are optional)
1. Optional codes for
    1. option codes
    1. equipment packs
    1. accessories
1. Image metadata like resolution, background colour and car angle.

```
https://example.org
    /de
    /vehicle/trabant/5-doors/0815+universal
    /options/1,4711,815
    /packs/p7
    /accessories/a,b,c
    /width/1024
    /height/768
    /exterior-45.jpg
```

As you can see, there is quite a bit of inconsistency that might cause some inconvenience while parsing: the country code doesn't have a qualifier, the car model, body and grade info are delimited by slashes, the other fields are delimited by commas. Also there's some information encoded in the filename, but we can ignore that as we're only parsing the car configuration, not the image metadata.

## The library

The basic idea of any parser combinator is that it takes an input, reads ("consumes") it until it’s either _done_ or _fails_, in both cases  returning the _result_ and the _remaining unparsed string_, which might be fed into subsequent parsers. In this example we will be using the [Parsec library](https://github.com/aslatter/parsec) to manage the grunt work for us. It seems well-suited for this task, it's quite readable and as straightforward as something that calls itself "industrial strength, monadic parser combinator" can be.

The library is written in the pure functional programming language Haskell, but there are many Parsec-clones in mainstream languages like Java, C#, Python or F#. To understand the examples in this post, you don't really need to understand Haskell, though. I chose Haskell for the code examples because the parsers contain barely any language-specific syntax, all the handling is nicely kept away from us, making the code almost look like pseudo code.

## Writing the parser

As mentioned earlier, parser combinators are built up from simple building blocks combined to complex parsers. There are quite a lot predefined parsers for common tasks that we can re-use and combine. It's fascinating that all parsers are independent and can be executed on their own, so everything is easily maintainable and testable.

> Functions in Haskell are defined by just assigning a value to a name. Types are inferred by the compiler and type annotations are recommended but optional.

We're dealing with an URL, so even without knowing the specs, we can guess that we will encounter "things delimited by slashes". According to the webservice specification, these "things" can only consist of characters and numbers. We'll start with a parser that reads these symbols between the slashes and call it `value`.

```
value = do
    many1 alphaNum
```

`many1` and `alphaNum` are parsers already defined in the Parsec library. When run, our combined parser expects one or more (`many1`) alphanumeric symbols, i.e. letters or numbers (`alphaNum`). If the input matches these characters it will succeed, if it encounters any other symbol, it will fail. The result of the last line in our `do`-block is returned automatically.

> Let's ignore the fact that the function doesn't have an explicit input value and just assume that the `do` means "read something from somewhere, expect input in sequential order and spare me the details". In reality it has to do something with the [M-word](https://en.wikipedia.org/wiki/Monad_(functional_programming)), but that would go vastly beyond the scope of this article.
> You might have noticed that Haskell doesn't always need parentheses around and commas between function parameters. Function application is left-associative, so this won't work: `print 1 + 2` as it would try to add `2` to the return value of `print 1`. You will need parentheses here: `print (1 + 2)`.

I lied when I said that values can consist only of alphanumeric characters. Actually the webservice specifications also allow the use of "+" and "-", so we'll need to add these to our parser. A nice way to achieve this is to use the `<|>`-operator that basically just means "or":

```
value = do
    many1 (alphaNum <|> char '+' <|> char '-')
```

Another option would be to use `... <|> oneOf "+-"`.

With this out of the way we have everything we need to combine this parser to another parser that reads the separate parts of our URL (you remember: things delimited by slashes). For lack of a better name, let's call it `part`.

```
part = do
    char '/'
    value
```

This does exactly what you might expect: It reads the character "/" and then a `value` as defined before.

I mentioned earlier that parser functions either return the value or fail. In most imperative programming languages, failing usually means that some kind of error or exception will be thrown. Our parsers work a bit differently: Haskell has an datatype called `Either` that represents values with two possibilities called `Left` and `Right`. By convention, `Left` is used to hold an error value and `Right` is used to hold a correct value. You'll see in a minute how that looks.

Let's confirm in Haskell's REPL that our parsers work. For this we'll use the `parse` helper function that expects the following parameters: the parser to execute, a context name (that we'll just leave empty) and the input string. 

```
> parse part "" "/foo+bar-42"
Right "foo+bar-42"

> parse part "" "quux"
Left (line 1, column 1):
unexpected "q"
expecting "/"
```

The results should be self explanatory: The first parser succeeded, returning a `Right` with the parsed string. The second call failed, returning a `Left` with a detailed error message. 

Now let's start to do some real work and write a parser that reads the first part of the URL including the country code and return that. The protocol can be either "http" or "https", so we need to take care of both options, for the rest we can reuse our `part`-parser. 

This parser introduces two new predefined parsers: `string`, which reads any given string (as opposed to `char` that only reads a single character) and `optional` that (surprise!) marks a parser as optional.

```
country = do
    string "http"
    optional (char 's')
    string "://example.org"
    part
```

In action:

```
> parse country "" "https://example.org/de/otherStuff"
Right "de"

> parse country "" "http://not-the-example.org/de"
Left (line 1, column 5):
unexpected "n"
expecting "://example.org"
```

We'll skip the vehicle part for now and implement the parsers for the car customization first. Options, accessories and packs are all quite similar: a key followed by a slash and a comma separated list of values. Let's start with an interesting parser: `sepBy` splits a string using a separator and returns the results. The output won't be a string now, but a list of strings instead:

```
commaDelimited = do
    sepBy value (char ',')
```

In the REPL this looks like this:

```
> parse commaDelimited "" "1,2,3,4"
Right ["1","2","3","4"]
```

The rest is easy:

```
packs = do
    string "/packs/"
    commaDelimited

options = do
    string "/options/"
    commaDelimited

accessories = do
    string "/accessories/"
    commaDelimited
```

With that out of the way we can concentrate on the most interesting part now: The vehicle information that can consist of one or more `part`s. Let's start with a quite naive approach:

```
vehicle = do
    string "/vehicle"
    many1 part
```

This will obviously not work as it will read beyond the vehicle information and into the customization:

```
> parse vehicle "" "/vehicle/ford/prefect/packs/pack307"
Right ["ford","prefect","packs","pack307"]
```

We need a way to tell the parser where to stop. That can be done with the `manyTill` parser that runs a parser until another parser matches. If we want to check for more than one parser, we can use `choice` which expects a list of parsers and tries them consecutively. The problem is that parsers works greedily - so any character that matches a parser and is therefor already consumed stays consumed, even if the parser should eventually fail. We can circumvent this by using `try` which makes the parser only consume input when it's fully executed. 

The tricky part is: We don't want that either. We only want to check, _if_ a parser will succeed, but not actually consume anything. The `lookAhead` parser does exactly that.

```
customization = do
    choice [try packs, try accessories, try options]

vehicle = do
    string "/vehicle"
    manyTill part (lookAhead customization)
```

This will first expect the string "/vehicle" and then run the `part` parser for as long as possible until our `customization` parser succeeds, but it will only consume the results of `parse`.

That's it, we don't need the rest (`skipMany`), but we should still read it:

```
theRest = do
    skipMany anyChar
```

These are all the building blocks we need to parse the url, but we have to combine them. Up to this point, all parsers did something and returned the last result _after_ running all parsers. What if we want the results of _all_ parsers, i.e. the in-betweens?

Let's take a step back first and write a parser that reads _exactly two_ alphanumeric characters and then expects the input to end (`eof`):

```
twoChars = do
    alphaNum
    alphaNum
    eof
```

```
> parse twoChars "" "xy"
Right 'y'
```

That's nice, but how do we get to the first `x`? As each parser returns a result itself, we can just pull it out and store it in a variable using the left arrow `<-` and return a tuple containing all variables. You don't have to return a tuple, you can return whatever data type holds your values and matches your domain.

```
twoChars = do
    x <- alphaNum
    y <- alphaNum
    eof
    return (x, y)
```

> You might remember me writing that the last line is always returned automatically, so what is this explicit `return` doing there? In Haskell, `return` doesn't work like return in most languages: it takes a value and wraps it in a context.
> If you actually want to know what's going on, I can recommend the excellent book [Learn You a Haskell For Great Good](http://learnyouahaskell.com/) by Miran Lipovača. If you're just here for the parsers, you can safely ignore it.

Wrapping it all up, here's our complete parser:

```
parser = do
    c  <- country
    v  <- vehicle
    cs <- many1 customization
    theRest
    eof
    return (c, v, cs)
```

Taking it for a test run:


```
> parse parser "" "https://example.org/de/vehicle/trabant/5-doors/0815+universal/options/1,4711,815/packs/p7/accessories/a,b,c/width/1024/height/768/exterior-45.jpg"
Right ("de",["trabant","5-doors","0815+universal"],[["1","4711","815"],["p7"],["a","b","c"]])
```

Nice.

You can find the complete source code here: [parser.hs](https://gist.github.com/cvk77/244bcdf7eb4f0f049221).

## What's next?

This barely touches the surface of what Parsec is able to do. If you're interesting in learning more, an excellect starting point is chapter 16 of [Real World Haskell](http://book.realworldhaskell.org/read/using-parsec.html) by Bryan O'Sullivan, Don Stewart, and John Goerzen. It goes into much more detail, but expects quite a bit of Haskell knowledge. Also you can [Write yourself a Scheme in 48 Hours](https://en.wikibooks.org/wiki/Write_Yourself_a_Scheme_in_48_Hours/Parsing) using Parsec.

If you want to understand parser combinators in general, not necessarily only the Parsec library, check out Graham Hutton's great book [Programming in Haskell](http://www.cs.nott.ac.uk/~pszgmh/book.html). Erik Meijers [functional programming MOOC at edX](https://courses.edx.org/courses/DelftX/FP101x/3T2014/) also covers the topic, closely following Hutton's book.

As mentioned earlier, there are parser combinator libraries for many languages, here's a short and in no way complete list:
* [JParsec](https://github.com/jparsec/jparsec) for Java
* [FParsec](http://www.quanttec.com/fparsec/) for F#
* [PyParsing](http://pyparsing.wikispaces.com/) for Python

If you want to know more about the details of the whole `do`-notation and `return`-stuff or just impress your friends with [Zygohistomorphic prepromorphisms](https://wiki.haskell.org/Zygohistomorphic_prepromorphisms), [Learn You A Haskell](http://learnyouahaskell.com/) is an excellent start. At the time of writing, [Chris Allen](https://twitter.com/bitemyapp) and [Julie Moronuki](https://twitter.com/argumatronic) are about 90% done with their [Haskell Book](http://haskellbook.com/). I'm sure it will be awesome.
