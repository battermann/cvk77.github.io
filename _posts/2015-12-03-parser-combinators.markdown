# Parser combinators

When tasked to extract information from a string, most seasoned developers - especially the ones with some kind of Linux background - will resort to regular expressions. While regular expressions are incredibly powerful and very well suited for most parsing jobs, with increasing complexity they tend to become very cryptic or even unreadable.

In functional programming, it's common to use parser combinators that combine small basic parsers to build a complex ruleset. Erik Meijers functional programming MOOC at edX recently covered the topic, closely following Graham Hutton's great book "Programming in Haskell".

Let's see how parser combinators can be used in an (admittedly easy) context:

One of our customers runs a fascinating webservice: Using a straightforward URL scheme, you can remotely control a rendering engine that can effectively combines hundreds of different car parts in order to render a photorealistic image of a specific car configuration.

We will use parser combinators to write a parser that extracts the configuration-specific data from such a webservice call.

## The webservice

As mentioned before, the URL format is quite straightforward, but at first glance it seems to be more developer-friendly than parser-friendly, with one weird quirk. It consists of the following parts:

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

We will be using Haskell's Parsec library, as it seems well-suited for this task: It's as straightforward as something that calls itself "industrial strength, monadic parser combinator" can be and there are many clones and implementations in mainstream languages like Java, C#, Python or F#.

Don't worry if you don't know Haskell - the parsers are so tiny that you should be able to understand what's going on without knowing the language. But in the footnotes you'll find an example written for  JParsec in Java.

As mentioned before, parser combinators build up complex parsers by combining simple building blocks. It's fascinating that all parsers are functions and can be executed independently, so everything is easily maintainable and testable.

## Writing the parser

As we're dealing with an URL it's quite obvious that even without looking at the example above, we will encounter "things delimited by slashes", so let's first define "things" and write a parser for that and - for lack of a better name - let's call them `value` and `part`:

```
value = do
    many1 $ choice [alphaNum, char '+', char '-']

part = do
    char '/'
    value
```

Let's ignore the `do` for now and just assume it means "expect this in sequential order".
Now `value` parses one or more (`many1`) letters, numbers (`alphaNum`), plusses (`char '+'`) or minuses (`char '-'`) and `part` is defined as a parser that reads the character "/" followed by a value, returning the latter.

The parser functions return either an error message if the parser didn't match or the extracted value. Haskell has an datatype called `Either` that represents values with two possibilities. By convention, Left is used to hold an error value and Right is used to hold a correct value.

Let's confirm in Haskell's REPL that our parsers work:

```
> parse part "" "/foo+bar-42"
Right "foo+bar-42"

> parse part "" "quux"
Left (line 1, column 1):
unexpected "q"
expecting "/"
```

Nice! Now let's start to do some real work and write a parser that reads the first part of the URL including the country code and return that. We can reuse our `part`-parser here:

```
country = do
    string "http"
    optional (char 's')
    string "://example.org"
    part
```

We'll skip the vehicle part for now and implement the parsers for the car customization first. Options, accessories and packs are all quite similar: a key followed by a slash and a comma separated list of values, so we can implement them at once.

```
commaDelimited = do
    value `sepBy` (char ',')

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

This will obviously not work as it will read beyond the vehicle information and into the customization, so we need to tell the parser where to stop.

```
customization = do
    choice [try packs, try accessories, try options]

vehicle = do
    string "/vehicle"
    manyTill part (lookAhead customization)
```

Let's skip the rest.

```
theRest = do
    skipMany anyChar
```

```
parser = do
    country <- country
    vehicle <- vehicle
    customization <- many1 customization
    foo <- theRest
    eof
    return (country, vehicle, customization, foo)
```

```
> parse parser "https://example.org/de/vehicle/trabant/5-doors/0815+universal/options/1,4711,815/packs/p7/accessories/a,b,c/width/1024/height/768/exterior-45.jpg"
Right ("de",["trabant","5-doors","0815+universal"],[["1","4711","815"],["p7"],["a","b","c"]])
```
