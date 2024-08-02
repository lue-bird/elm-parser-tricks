Unlike most other aspects of elm, making your parser faster will likely make it less readable
and harder to understand.

I have not found improving performance to be all that intuitive at all.
Most things listed below have exceptions listed (and likely others I haven't encountered or noticed).

### The key to success is not `succeed`

#### ~`succeed (\) |= first`~ `map (\) first`

Yes you heard it right. `|=`, `|.` etc look pretty nice but
since they are a `map2` underneath.
This means
We first construct a new parser with `succeed`, case on it,
case on the incoming parser, apply.
If we instead use `map` directly on the parser, we
just case on it, and apply a given function.

#### ~`succeed (\) |. first |= second`~ `first |> continueWith (map (\) second)`

with
```elm
continueWith next before =
    before |> Parser.andThen (\() -> next)
```
which allows us to not worry about pre-defining the "next" parser (see [section "constructing parsers dynamically is evil"](constructing-parsers-dynamically-is-evil)),
we can just as above avoid a `map2`.

This might be minimally faster compared to
```elm
map (\() secondResult -> ...)
    first
    |= second
```
TODO but I've not benchmarked this


### nested map is bad

Imagine e.g.
```elm
functionDeclaration =
    ...
    |= oneOf
        [ map Just typeSignature
        , succeed Nothing
        ]
    |= implementation

typeSignature =
    succeed (\name type_ -> ...)
        |= ...
```
Intermediate summarizing of information is poison.

There are roughly 2 solutions for this:
  - use `Just` directly in `typeSignature` (this can be rough if you need it multiple times, and some times without the `Just`, e.g. in the port signature)
  - add a map parameter to `typeSignature` that transforms the result (this tends to be ever so slightly slower for the `identity` case)

There are exceptions to this rule, though very few ones. E.g.
```elm
map f p |. andThen (\y -> predefinedSucceedUnit) ...
```
is evil but
```elm
andThen (\y -> \x -> succeed (f ...)) p |= ...
```
is slightly worse. See also: [section "constructing parsers dynamically is evil"](constructing-parsers-dynamically-is-evil)

A final tip: use `mapChompedString (\() string -> )` instead of `map (\string -> ) (getChompedString ...)` wherever you can.

### nested `oneOf` is not great

E.g.
```elm
oneOf
    [ symbol "()"
    , symbol "(" |> continueWith ...
    , ...
    ]
```
is usually faster than
```elm
oneOf
    [ symbol "("
        |> continueWith
            (oneOf
                [ symbol ")"
                , ...
                ]
            )
    , ...
    ]
```

unless
- the shared token is more expensive
- there are much more than 2 with the same shared start
- there are a lot of likely possibilities below

### constructing parsers dynamically is evil

It is really convenient and easy to reach for `andThen`
when you want to continue after parsing existing shared data.

```elm
TODO
```

There are roughly 3 solutions for this:
  - aggressive reliance on pre-defined parsers in `andThen`
  - apply style trick
  - intermediate union type

TODO show each example

*Exceptions:

When using `Parser.loop` with a step that's a single `oneOf` (do keep an eye out for optimizations there, though).

Also, if the parser to construct dynamically is very small
and not dynamically constructing it would result in an extra step
```elm
Parser.map
    (\column ->
        \indent ->
            if indent < column then
                succeedUnit

            else
                problemPositivelyIndented
    )
    Parser.getCol
    |= Parser.getIndent
    |> Parser.andThen identity
```
which you can also write as
```elm
Parser.getCol
    |> Parser.andThen
        (\column ->
            Parser.andThen
                (\indent ->
                    if column > indent then
                        Parser.succeed ()

                    else
                        Parser.problem "must be positively indented"
                )
                Parser.getIndent
        )
```
Lookaheads and lookbehinds usually benefit from this pattern.

### `variable` is faster than non-empty chomping
even when you discard the result implicitly,
`chompIf ... | chompWhile` or `symbol ... |. chompWhile`
are slower than
```elm
variable { start = \c -> ..., inner = \c -> ..., reserved = Set.empty }
```

TODO verify this is also the case for only `chompWhile |> getChompedString`

The one exception is if you actually need to `map` the resulting string.
There `variable |> map f` will be slower than `chompWhile |> mapChompedString (\() s -> f s)`

TODO verify this is also the case for only `chompIf |. chompWhile |> getChompedString`

### `keyword` is fast
It's only minimally slower than `symbol`, so if you need to check for whitespace after a character,
`keyword` should be a no-brainer for well... keywords because it allows you to not explicitly handle "non-empty" whitespace.

### avoid `|= getPosition` etc whenever you can
rely on the information TODO

### use backtracking parsers as little as possible

This one is less obvious than you might think!
Parsers like `getCol`, `getRow`, `getIndent`, `getPosition`, `getOffset`, `getSource`, `succeed`, `end`, `oneOf` with no possibilities, `loop` that succeeds immediately, etc. need to be backtrackable
because they don't consume characters.

So if we e.g. have
```elm
oneOf
    [ map (\) getPosition |. symbol "(" |= ...typeA
    , map (\) getPosition |. symbol "[" |= ...typeB
    , map (\) getPosition |. symbol "{" |= ...typeC
    ]
```
The `getPosition` parser will be called three times in the worst case.

There are 2 ways to fix this:
  - move the `getPosition` one level higher out of the `oneOf` (less general and less performant)
  - move the `getPosition` after the identifying start symbol (more general but error prone since you have to calculate the actual start position)

Note that 1 will actually perform better if a common possibility parser is ambiguous and needs to backtrack anyways (like `Parser.number`).

### shortcut to end

This might be the most obvious but it's easy to miss.

Blocks of non-trivially-separated elements (usually with whitespace in between) that have a fixed end symbol can make use of this to exit element checks early.
Examples of such an end symbols:
  - `in` in `let {declarations} in` (like in e.g. elm)
  - `=` in `functionName {arguments} =` (like in e.g. elm)
  - `)` in `functionName({arguments})` (like in e.g. java)
  - `}` in `{{statements without semicolons}}` (like in e.g. java)

```elm
until : Parser () -> Parser a -> Parser (List a)
until end element =
    let
        step : List a -> Parser (Parser.Step (List a) (List a))
        step itemsSoFar =
            Parser.oneOf
                [ Parser.map (\() -> Parser.Done (List.reverse itemsSoFar)) end
                , Parser.map (\el -> Parser.Loop (el :: itemsSoFar)) element
                ]
    in
    Parser.loop [] step
```

This advice does not apply if your elements are cheap to check for,
like `symbol "," |> continueWith element`.

It also (for the most part) doesn't apply it the end is itself not trivial to check for (e.g. `finalExpression` in `{variable = value\n}finalExpression`).
If you know fewer elements are way more common, it might make sense, though.
