Unlike most other aspects of elm, making your parser faster will likely make it less readable
and harder to understand.

I have not found improving performance to be all that intuitive.
Most advice below has exceptions listed (and likely others I haven't encountered or noticed).

### The key to success is not `succeed`

#### replace `succeed f |= first` by `map f first`

`|=` and `|.` look pretty but
since they are a `map2` underneath, we first construct a new parser with `succeed`, case on it,
case on the incoming parser, apply.

If we instead use `map` directly on the parser, we
just case on it and apply a given function.

#### replace `succeed f |. first |= second` by `map (\() -> f) first |= second`

This avoids a `map2` just as above.

### replace `succeed identity |= kept |. ignored` by `kept |. ignored`
`succeed identity |=` is unnecessary.

### replace `succeed identity |. ignored |= kept` by `ignored |> continueWith kept`
using
```elm
{-| Like `Parser.andThen (\() -> ...)` but circumvents laziness
-}
continueWith : Parser b -> Parser () -> Parser b
continueWith b a =
    a |> Parser.andThen (\() -> b)
```
To learn about why we need to "circumvent laziness", see [section "constructing parsers dynamically is evil"](constructing-parsers-dynamically-is-evil)

### nested map is bad

E.g.
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
andThen (\y x -> succeed (f ...)) p |= ...
```
is slightly worse. See also: [section "constructing parsers dynamically is evil"](constructing-parsers-dynamically-is-evil)

A final tip: use `mapChompedString (\() string -> )` instead of `map (\string -> ) (getChompedString ...)` wherever you can.


### constructing parsers dynamically is evil

Sometimes (and that's somewhat bad design from the format you're parsing I'd say),
the exact structure you parse into is not determined after parsing
a first piece of data.

E.g. with `public static final @Export` you could continue with `class`, a function name etc.
```elm
modifiers
    |> Parser.andThen
        (\modifiersResult ->
            Parser.oneOf
                [ classWithModifiers modifiersResult
                , functionWithModifiers modifiersResult
                ]
        )

classWithModifiers : Modifiers -> Parser Class
classWithModifiers modifiers =
    succeed (\rest -> { modifiers = modifiers, ...rest })
        |= ...
-- same for functionWithModifiers
```
It is really convenient and easy to reach for `andThen`
when you want to continue after parsing existing shared data.

This will however construct parsers for `classWithModifiers` and `functionWithModifiers`
again and again even though their parser shape never actually changes,
and that's really not free.

There are roughly 3 solutions for this:
  - aggressive reliance on pre-defined parsers in `andThen` (you can use [`elm-review-predefine`](https://github.com/lue-bird/elm-review-predefine) to find spots that could be problematic)
  - apply style trick
  - intermediate union type

With the apply style trick:
```elm
Parser.succeed (\modifiersResult withModifiers -> withModifiers modifiersResult)
    |= modifiers
    |= Parser.oneOf
        [ classWithModifiers
        , functionWithModifiers
        ]

classWithModifiers : Parser (Modifiers -> Class)
classWithModifiers =
    Parser.succeed (\rest modifiers {-as the last argument-} -> { modifiers = modifiers, ...rest })
        |= ...
-- same for functionWithModifiers
```
It does the job.
I have a personal distaste for it because it hands the ability to construct the full thing to a sub-parser which is kinda weird.

With an intermediate union type:
```elm
Parser.succeed
    (\modifiersResult afterModifiers ->
        case afterModifiers of
            ClassAfterModifiers classAfterModifiers ->
                Class { modifiers = modifiersResult, ...classAfterModifiers }
            
            FunctionAfterModifiers functionAfterModifiers ->
                Function { modifiers = modifiersResult, ...functionAfterModifiers }
    )
    |= modifiers
    |= Parser.oneOf
        [ classAfterModifiers
        , functionAfterModifiers
        ]

type AfterModifiers
    = ClassAfterModifiers ...
    | FunctionAfterModifiers ...

classAfterModifiers : Parser AfterModifiers
classAfterModifiers =
    Parser.succeed (\...afterModifiers -> { ...afterModifiers })
        |= ...
-- same for functionWithModifiers
```
I find this the option that's easiest to understand among the 3.
It also doesn't compromise on speed.
Note: you'll get some small extra performance there if each variant has the same number of arguments:
[article "Improving the performance of Custom Types" by Robin Heggelund Hansen](https://medium.com/bekk/improving-the-performance-of-custom-types-39f7e2a1d8e1)


Exceptions:

Using `loop`, a step that's a single `oneOf`
with cheap parsers tends to be surprisingly fast still,
compared to e.g. a step which is a `map` on a pre-defined parser
(do keep an eye out for optimizations like that, though).

In most cases, if the parser to construct dynamically is cheap
and not dynamically constructing it would result in an extra step
```elm
Parser.map
    (\column indent ->
        if column > indent then
            predefinedSucceedUnit

        else
            predefinedProblemPositivelyIndented
    )
    Parser.getCol
    |= Parser.getIndent
    |> Parser.andThen identity
```
you can rewrite it to the faster
```elm
Parser.getCol
    |> Parser.andThen
        (\column ->
            Parser.andThen
                (\indent ->
                    if column > indent then
                        predefinedSucceedUnit

                    else
                        predefinedProblemPositivelyIndented
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
Sometimes locations can be inferred from what you've parsed.
E.g.
  - an `if` expression already knows when the `else` branch expression ends
  - an unqualified function name can derive the start/end from the location of one side and it's length
  - a single-line comment only spans one line (what!?) so you only need `getCol` to calculate the end location
  - a type declaration must not be indented, so you only need `getRow` to calculate the start location   

### use backtracking parsers as little as possible

This one is less obvious than you might think!
Parsers like `getCol`, `getRow`, `getIndent`, `getPosition`, `getOffset`, `getSource`, `succeed`, `end`, `oneOf` with no possibilities, `loop` that succeeds immediately, etc. are always backtrackable
because they don't consume characters.

So if we e.g. have
```elm
Parser.oneOf
    [ ... Parser.getPosition |. Parser.symbol "(" |= ...typeA
    , ... Parser.getPosition |. Parser.symbol "[" |= ...typeB
    , ... Parser.getPosition |. Parser.symbol "{" |= ...typeC
    ]
```
The `getPosition` parser and the map on it will be called three times in the worst case.

There are 2 ways to fix this:
  - move the `getPosition` one level higher out of the `oneOf` (less general and less performant)
  - move the `getPosition` after the identifying start symbol (more general but error prone since you have to calculate the actual start position)

Note that 1 will actually perform better if a common possibility parser is ambiguous and needs to backtrack anyways (like `number`).

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

It also (for the most part) doesn't apply it the end is itself not trivial to check for (e.g. `finalExpression` in `{variable = value\n}finalExpression`, where `finalExpression` needs to be backtrackable).
If you know fewer elements are way more common, it might make sense, though.

### `Parser` is fine, except for `Parser.loop`

`Parser.x` helpers are usually direct or very cheap calls to `Parser.Advanced.x`,
so switching won't do much.

The only exception is `Parser.loop` which you should replace by `Parser.Advanced.loop`.
Since `Parser.Parser` is an alias of the advanced type,
the only thing you have to change is the `Step` and `loop` qualification.

This change saves a `Parser.map` for every loop step internally.

### nested `oneOf` is not great

E.g.
```elm
Parser.oneOf
    [ Parser.symbol "()"
    , Parser.symbol "(" |> continueWith ...
    , ...
    ]
```
is usually faster than
```elm
Parser.oneOf
    [ Parser.symbol "("
        |> continueWith
            (Parser.oneOf
                [ Parser.symbol ")"
                , ...
                ]
            )
    , ...
    ]
```

The impact of this optimization will only be small
but especially if one of the top-level possibilities is itself a `oneOf` under the hood,
flattening it out is a good idea.

You should probably not do it if
  - the shared token is more expensive
  - there are much more than 2 possibilities with the same shared start
  - there are a lot of likely possibilities below

### `lazy` is pretty good

You might have seen code like
```elm
succeed () |> map (\() -> Done (List.reverse items))
```
to avoid calling `List.reverse` at every step.

There's a function for that that's faster because it doesn't need to case on the result like `map`:
```elm
lazy (\() -> succeed (Done (List.reverse items)))
```
This technically violates [section "constructing parsers dynamically is evil"](constructing-parsers-dynamically-is-evil)
but since this will only be called once, it'll be fine!

I can't tell you at which point it's better to use
`lazy` than `succeed` directly.
Be suspicious around maybe "three field record".
