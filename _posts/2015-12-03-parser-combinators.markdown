# Parser combinators

When tasked to extract information from a string, most seasoned developers - especially the ones with some kind of Linux background - will resort to regular expressions. While regular expressions are incredibly powerful and very well suited for most parsing jobs, with increasing complexity they tend to become very cryptic or even unreadable.

In functional programming, it's common to use parser combinators that combine small basic parsers to build a complex ruleset. Erik Meijers [functional programming MOOC at edX](https://courses.edx.org/courses/DelftX/FP101x/3T2014/) recently covered the topic, closely following Graham Hutton's great book [Programming in Haskell](http://www.cs.nott.ac.uk/~pszgmh/book.html).

Let's see how parser combinators can be used in an (admittedly simple) context.

One of our customers runs a fascinating webservice: Using a easy-to-understand URL scheme, you can remotely control a rendering engine that can effectively combines hundreds of different car parts in order to render a photorealistic image of a specific car configuration.

We will use parser combinators to write a parser that extracts the configuration-specific data from such a webservice call.

## The webservice

As mentioned before, the URL format is quite straightforward, but at first glance it seems to be more human-friendly than machine-friendly, but more about that later. It consists of the following parts:

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

As you can see, there is quite a bit of inconsistency: the country code doesn't have a qualifier, the car model, body and grade info are delimited by slashes, the other fields are delimited by commas.

## The library

The basic idea of a parser is that it takes an input, reads ("consumes") it until it’s either _done_ or _fails_, in both cases  returning the _result_ and the _remaining unparsed string_, which might be fed into subsequent parsers. In this example we will be using the [Parsec library](https://github.com/aslatter/parsec) to manage the grunt work for us. It seems well-suited for this task, it's quite readable and as straightforward as something that calls itself "industrial strength, monadic parser combinator" can be. 

The library is written in the functional programming language Haskell, but there are many Parsec-clones in mainstream languages like Java, C#, Python or F#. To understand the examples in this post, you don't really need to understand Haskell though. The parsers itself are so tiny any readable that you should be able to understand what's going on without knowing the language.

## Writing the parser

As mentioned earlier, parser combinators are built up from simple building blocks combined to complex parsers. It's fascinating that all parsers are independent and can be executed on their own, so everything is easily maintainable and testable.

We're dealing with an URL, so even without knowing the specs, we can guess that we will encounter "things delimited by slashes". According to the webservice specifications, these "things" can only consist of characters and numbers. We'll start with a parser that reads these symbols between the slashes and call it `value`.

```
value = do
    many1 alphaNum
```

Let's ignore the `do` for now and just assume it means "expect things in sequential order". In reality it has to do something with the [M-word](https://en.wikipedia.org/wiki/Monad_(functional_programming)), but that would go vastly beyond the scope of this article.

When run, `value` parses one or more (`many1`) letters or numbers (`alphaNum`). If the input matches these characters it will succeed, if it encounters any other symbol, it will fail. The `return` is implied: The result of the last line in our `do`-block is returned automatically.

The webservice specifications also allow the use of "+" and "-" in values, so we'll need to add these to our parser using the `<|>`-operator that basically just means "or":

```
value = do
    many1 (alphaNum <|> char '+' <|> char '-')
```

Now we have everything we need to combine this parser to another parser that reads the separate parts of our URL (you remember: things delimited by slashes). For lack of a better name, let's call it `part`.

```
part = do
    char '/'
    value
```

This does exactly what you might expect: It reads the character "/" and then a `value` as defined before.

I mentioned that parser functions either return the value or fail. In imperative programming languages, failing usually means that some kind of error or exception will be thrown. Our parsers work a bit differently: Haskell has an datatype called `Either` that represents values with two possibilities called `Left` and `Right`. By convention, `Left` is used to hold an error value and `Right` is used to hold a correct value. You'll see in a minute how that looks.

Let's confirm in Haskell's REPL that our parsers work. For this we'll use the `parse` command which expects the following parameters: the parser to execute, a name that we'll just leave empty as we don't need it and the input string. You probably already noticed that Haskell doesn't need braces `()` around and commas between function parameters.

```
> parse part "" "/foo+bar-42"
Right "foo+bar-42"

> parse part "" "quux"
Left (line 1, column 1):
unexpected "q"
expecting "/"
```

Nice! Now let's start to do some real work and write a parser that reads the first part of the URL including the country code and return that. We can reuse our `part`-parser again here and will introduce two new parsers: `string`, which reads a string (as opposed to `char` that only reads a single character) and `optional` that (surprise!) marks a parser as optional.

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
    value `sepBy` (char ',')
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

We need a way to tell the parser where to stop. That can be done with the `manyTill` parser that runs a parser until another parser matches. The problem is that the parser works greedily - so any character that matches a parser and is therefor already consumed stays consumed, even if the parser fails later. We can circumvent this by using the `try` statement that makes the parser only consume input when it's fully executed.

```
customization = do
    choice [try packs, try accessories, try options]

vehicle = do
    string "/vehicle"
    manyTill part (lookAhead customization)
```

That's it, we don't need the rest (`skipMany`):

```
theRest = do
    skipMany anyChar
```

These are all the building blocks we need to parse the url, but we have to combine them. Up to this point, all parsers did something and returned the result after running all parsers, but what if we want the results of all parsers?

Let's take a step back first and write a parser that reads exactly _two_ alphanumeric characters and then expects the input to end (`eof`):

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

That's nice, but how do we get to the first `x`? As each parser returns a result itself, we can just store that in a variable using the left arrow `<-` and return a tuple containing all variables. You don't have to return a tuple, you can use your favourite data structure.

```
twoChars = do
    x <- alphaNum
    y <- alphaNum
    eof
    return (x, y)
```

You might remember me writing that the last line is always returned automatically, so what is this explicit `return` doing there? In Haskell, `return` doesn't work like return in most languages, it takes a value and wraps it in a context. If you actually want to know what's going on, I can recommend the excellent book [Learn You a Haskell For Great Good](http://learnyouahaskell.com/) by Miran Lipovača. If you're just here for the parsers, you can safely ignore it.

Wrapping it all up, here's our parser:

```
parser = do
    country <- country
    vehicle <- vehicle
    customization <- many1 customization
    rest <- theRest
    eof
    return (country, vehicle, customization, rest)
```

Taking it for a test run:


```
> parse parser "https://example.org/de/vehicle/trabant/5-doors/0815+universal/options/1,4711,815/packs/p7/accessories/a,b,c/width/1024/height/768/exterior-45.jpg"
Right ("de",["trabant","5-doors","0815+universal"],[["1","4711","815"],["p7"],["a","b","c"]])
```

Nice.

## What's next?

This barely touches the surface of what Parsec is able to do. If you want to read more, an excellect starting point is chapter 16 of [Real World Haskell](http://book.realworldhaskell.org/read/using-parsec.html) by Bryan O'Sullivan, Don Stewart, and John Goerzen. It goes into much more detail, but expects quite a bit of Haskell knowledge. Also you can [Write yourself a Scheme in 48 Hours](https://en.wikibooks.org/wiki/Write_Yourself_a_Scheme_in_48_Hours/Parsing) using Parsec.

As mentioned earlier, there are parser combinator libraries for many languages, here's a short and in no way complete list:
* [JParsec](https://github.com/jparsec/jparsec) for Java
* [FParsec](http://www.quanttec.com/fparsec/) for F#
* [PyParsing](http://pyparsing.wikispaces.com/) for Python

If you want to know more about the details of the whole `do`-notation and `return`-stuff or just impress your friends with [Zygohistomorphic prepromorphisms](https://wiki.haskell.org/Zygohistomorphic_prepromorphisms), [Learn You A Haskell](http://learnyouahaskell.com/) is an excellent start. At the time of writing, Christopher Allen and Julie Moronuki are about 90% done with their [Haskell Book](http://haskellbook.com/). I'm sure it will be awesome.
