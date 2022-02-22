# Partial function application in Erlang
This document details an idea that could be implemented either as a parse
transform or as a native Erlang feature, namely how <N arguments could be
applied to an arity N function.

A lot of the contents of this document is from
[this thread](https://erlangforums.com/t/is-partial-function-application-in-erlang-a-good-idea/473)
on the Erlang forums, where I got much valuable input and different opinions.

# Abstract
I think the language would benefit from having a special syntax for partial
function application not only for the current 0 arguments applied, but for
an arbitrary amount of arguments, to allow for more expressive use of
higher-order functions such as `lists:map/2`, but also for shorthand forms
of function calls where one or more arguments are constant.

My proposal to do this while also keeping the significant arity as part of the
function signature is to add special syntax for "missing arguments", as such:
```erlang
%%% As shorthand of a function with one constant argument
Listen = fun gen_tcp:listen(_, [{active, false}, {packet, 2}]),
HttpListener = Listen(80),
HttpsListener = Listen(443).

%%% As shorthand in a higher-order function
lists:filter(fun lists:prefix("hello", _), ["hello world", "goodbye world"]).
%%% ["hello world"]
```
Here, `_` is used as a special "missing argument" marker.

# Short intro to partial function application
Partial function application is a concept where a function call can be done
without providing all arguments. The resulting return value would be a new
function which accepts the remaining arguments.

In curried languages such as Haskell, partial function application is a
prerequisite for how the language works; there is no meaningful distinction
between a function that takes two arguments, and a function that takes one
argument and returns another function that takes one argument.

## Example from Haskell
A trivial example from Haskell is shown below.
```haskell
increment = (+) 1
```
Since `+` is a function that takes two arguments, but only a `1` was applied,
`increment` will be a function that takes a number and returns another number.
In other words, `increment 100` would return `101`.

# Arity and partial function application
Since arity is part of the function signature in Erlang, unlike in curried
languages, simply omitting arguments to achieve partial function application
would not make sense - is `string:trim("   hello   ")` the string "hello", or
is it the function:
```erlang
fun(Dir, Characters) -> string:trim("   hello   ", Dir, Characters) end.
```

## Special syntax
A special marker would be used to mark any missing arguments. I have chosen
`_` for this purpose, since I couldn't think of any other time it would be used
on the right-hand side of a pattern match, but any marker would do.

Furthermore, to be consistent with the rest of the language, the `fun` keyword
will be added before partially applied functions.
```erlang
TrimNewline = fun string:trim(_, trailing, "\r\n").
```
## Semi-useful side-effects
As a curious side-effect "partially applied" functions with _all_ arguments
already supplied can be used to construct arity 0 functions this way.
```erlang
Receive = fun gen_tcp:recv(LSocket, 0, infinity),
Receive().
%%% {ok, Packet}

CountUp = fun ets:update_counter(SomeTable, counter, 1),
CountUp().
%%% 1
CountUp().
%%% 2
```
Or, with _no_ arguments supplied as an alternative way of writing `fun m:f/a`.
```erlang
lists:filter(fun string:is_empty(_), ["", "", "", "nope"]).
%%% ["", "", ""]
```

# Motivation
## More expressive higher-order function usage
Oftentimes when using `map`, `fold` or `filter` functions, the function
argument often becomes very long due to the added `fun`, `->`, `end`
and repetition of placeholder variables when defining a simple wrapper
such as the `TrimNewline` example above.

For lists, it's often cleaner to use list comprehension for the `lists:map/2`
and `lists:filter/2` cases, but there's no clean way for `fold`s, nor can list
comprehension be used to solve the same problem for other data structures'
higher-order functions, e.g., `maps:map/2`, `sets:filter/2`.

## Allow for flexible pipe-operator syntax in the future
The pipe operator is not a feature in Erlang, at least not yet, but since Elixir
introduced an OCaml-like pipe operator `|>`, there has been a lot of talk about
wanting to add one to Erlang as well, not least by Joe Armstrong in
[this blog post](https://joearms.github.io/published/2013-05-31-a-week-with-elixir.html)

As is discussed in the comment thread of that post, introducing a pipe operator
into Erlang would come with a set of problems. First of all, Elixir's standard
library was carefully designed with the pipe operator in mind, so that what is
most likely the thing to be passed around from function to function, is the
_first_ argument in every function.

Erlang's standard library was not designed like this, and even if it was,
it's inevitable that at some point we will encounter a chain of functions
where we wish we could've piped the second argument instead, no matter how
carefully we design our libraries. Joe discusses this in length in the comments
of the blog post, proposing syntax such as `|0>` for piping to the first
argument, `|1>` for piping to the second argument, `|-1>` for piping to the last
argument, and so on.

My proposed solution for a pipe operator, if ever implemented, is for the
operator to _only accept arity 1 funs_ as the right-hand operand, and any Erlang
term as its left-hand operand. The argument position problem would disappear,
and with clever use of partial function application it would, in my opinion at
least, look much better than the positional piping syntax Joe was writing about.

```erlang
shouting_snake_case(Text) ->
    Text
    |> fun string:tokens(_, " ,.:") % Pipe to position 1
    |> fun lists:join("_", _) % Pipe to position 2
    |> fun string:uppercase/1. % Good old fun m:f/a notation also works.
```

The `fun` keyword is repeated quite a lot. `|>` could, as syntactic sugar,
add an implicit `fun`. It would look nicer, though it may not be a good idea
to make these `fun`s look different from other `fun`s.

The above example is equivalent to
```erlang
shouting_snake_case(Text) ->
    string:uppercase(lists:join("_", string:tokens(Text, " ,.:"))).
```

# Comparison to current Erlang
To be clear, adding partial function application would not be ground-breaking
and allow for never before seen applications of Erlang.
It is simply a shorthand form of wrapping a single function call in a `fun`.
Today, we already have the shorthand form `fun name/arity` of a function
where _zero_ arguments have been applied:
```erlang
lists:map(fun string:uppercase/1, ["hello", "world"]).
%%% ["HELLO","WORLD"]
```

But if more than zero but fewer than all arguments are provided, the only
options currently are to wrap the whole thing in a `fun`, or in a wrapper
function that reduces the amount of arguments. A wrapper is a perfectly valid
approach for helper functions that are used in several places, but wrapping
a single function call inside a function just so it can be used in one
`lists:foldl/3` in one module might be overkill.

```erlang
%%% With a wrapper function
get_greetings_wrapper(Messages) ->
    lists:filter(fun is_greeting/1, Messages).

is_greeting(Message) ->
    lists:prefix("hello", Message).

%%% With a conventional lambda function
get_greetings_lf(Messages) ->
    lists:filter(fun(Message) -> lists:prefix("hello", Message) end, Messages).

%%% With partial function application
get_greetings_pfa(Messages) ->
    lists:filter(fun lists:prefix("hello", _), Message).
```

The extra function layer in the first option adds some cognitive load,
especially for examples larger than this toy example where `is_greeting/1`
may not be immediately visible on the very next line in the text editor.

The lambda function is unwieldy even though it does a very simple thing
and it's hard to fit such function calls on a single line.

I would argue that the last example shows intent with less clutter
and lower cognitive load.

All the extra keywords and dummy parameters can be unwieldy when used in a
higher-order function.
```erlang
sets:map(string:trim(_, trailing, "\r\n"), Lines).
sets:map(fun(Ln) -> string:trim(Ln, trailing, "\r\n") end, Lines).
```

# Real-life use cases
Some examples in OTP that could have been clearer, more concise or just nicer
in general with the use of partial function application.

Todo:
* Find more examples
* Write equivalent code with PFA
* Also find examples that could use some piping (variables X0, X1, X2, X3...)
* Update the `partial_application` repo and either add a link here, or just
  add the examples straight to this repo so they can be cloned and tried.
  Needs to be updated with the `fun` keyword, to remove ambiguity
  mentioned in the forums.

## `diameter_service.erl`
* https://github.com/erlang/otp/blob/98b6ec065782ed17010fb2548764a72f540ef9ab/lib/diameter/src/base/diameter_service.erl#L628
* https://github.com/erlang/otp/blob/98b6ec065782ed17010fb2548764a72f540ef9ab/lib/diameter/src/base/diameter_service.erl#L636
* https://github.com/erlang/otp/blob/98b6ec065782ed17010fb2548764a72f540ef9ab/lib/diameter/src/base/diameter_service.erl#L1770
* https://github.com/erlang/otp/blob/98b6ec065782ed17010fb2548764a72f540ef9ab/lib/diameter/src/base/diameter_service.erl#L1776
* https://github.com/erlang/otp/blob/98b6ec065782ed17010fb2548764a72f540ef9ab/lib/diameter/src/base/diameter_service.erl#L2054
