# Introducing abilities

TODO: doc purpose and context

TODO: collect all the examples in a code repo

TODO: exercises?

[langref]: languagereference.html

See also [this][langref#abilities-and-ability-handlers].

TODO: contents list

## Motivation

TODO
 - quote the langref
 - (inspiration from idris paper https://eb.host.cs.st-andrews.ac.uk/drafts/effects.pdf)
 - including the fact that we're separating declaration of the ability from the handler which knows how to provide it
 - why unison has them, why you haven't heard of abilities before
 - benefits: separation of interface from implementation; easy to have pure test handlers, testability; control flow control; abilities compose; constraining type signatures

## Using abilities

OK, let's get started.  Here's the declaration of an ability called `SystemTime`, which will let's us write code that can read the system clock.

``` haskell
ability SystemTime where
  -- Number of seconds since the start of 1970.
  systemTime : .base.Nat
```

It defines one **request**, `systemTime`, which returns the clock reading.  We'll come back to the term 'request' later.  

TODO/question: should I use the word 'request' or 'ability constructor' (or 'method', which I'd prefer...)

Let's use this to write some code.

``` haskell
tomorrow = '(SystemTime.systemTime + 24 * 60 * 60)
```

The only non-obvious thing here is the delay, that is, the use of the `'`.  

> 🐘 Remember that while `42` is a `Nat`, `'42` is a `() -> Nat` - that is, a function which takes an argument of the unit type, and returns a `Nat`.  See [delayed computations][langref#delayed-computations] for more detail.

We're using the `'` to turn `tomorrow` into a function rather than just a value.  Even though values are in a sense just functions that take zero arguments, as far as abilities are concerned they are different creatures.  Here's the key point: 

👉 Ability requests exist in the context of a function processing one of its arguments.

It's in the process of a function doing some computation that it makes sense to make a request using an ability.  A computed value can't do it - it's just sitting there, with no more computation to do.  

So the following wouldn't make sense:

``` haskell
-- wrong
tomorrow = SystemTime.systemTime + 24 * 60 * 60
```

On the one hand, this code would be saying we just want `tomorrow` to be a computed value, just a `Nat` we've got in the bag.  But on the other it's saying we want to compute it with the help of the system clock.  When do we want that computation to happen - when we first add it to the codebase?  That wouldn't make sense.  

So we need use add the `'` delay, to turn `tomorrow` into function, so we can trigger the computation at the right moment (and with the help of a handler, which we'll come to later). 

### Type signatures of functions using abilities

If we add `tomorrow` to the codebase, Unison tells us it's inferred the following signature:

``` haskell
tomorrow : '{SystemTime} .base.Nat
```
> 🐘 Again, that `'` is syntactic sugar for delayed function types - this signature is equivalent to `() ->{SystemTime} .base.Nat`. 

> 🐞 You may see a `∀` in the signature, due to Unison issue #689.

The `{SystemTime}` is an **ability list** - in this case a list of just one ability.  It's saying that `tomorrow` _requires_ the `SystemTime` ability - that ability needs to be available in functions that call `tomorrow`.  And it's also saying that the `SystemTime` ability is available for use within the definition of `tomorrow` itself.  

Suppose you're writing a function `foo` which should call `tomorrow`.  There are two ways of making the `SystemTime` ability available:
1. Put an ability list containing `SystemTime` in the signature of `foo`, the same as with the signature of `tomorrow`.  Indeed, if you leave the signature of `foo` unspecified, this ability list will be inferred again.  In this way the `SystemTime` requirement propagates up the function call graph.  
2. Use a handler.  This is how we stop that propagation.  We'll get on to handlers later.  

So now we've seen another key point about abilities:

👉 The requirement for an ability propagates from one function's signature to all its callers' signatures (until terminated by a handler).

This propagation is supported through the ability type inference mechanism as we've seen.  It's also _enforced_ by the type-checker - you can't get away with omitting one of these abilities if it's in fact required in your function (or a function it calls).  The typechecker makes sure that our ability lists faithfully reflect what's going on.  

#### Type signatures in ability declarations

Let's revisit the ability declaration we started with.

``` haskell
ability SystemTime where
  systemTime : .base.Nat
```

There's a significant piece of information that's been elided here for brevity.  The full and unabridged version of this declaration would be the following.

``` haskell
ability SystemTime where
  systemTime : {SystemTime} .base.Nat
```

Note the ability list that's appeared in the request signature.  Just as `tomorrow` has this ability in its signature, which therefore propagates to up to `foo` (in the example from the previous section), so `systemTime` did even beforehand, and it was this which propagated to `tomorrow` in the first place.  

The reason we can omit the name of the ability being defined from its requests' signatures is that, logically, it always needs to be there - using an ability's request requires that ability.  So as far as Unison is concerned, this bit of an ability's type signature 'goes without saying'.  You can write it either way.  

> 🤔 An ability declaration is the only place you'll see an ability list in a signature that doesn't follow the rule that 'ability requests exist in the context of a function processing one of its arguments' - i.e. the only place you can have a `{SystemTime}` not following a `->` or a `'`.

### Examples of abilities

#### `Store`

The following ability lets us write programs that have access to mutable state.

``` haskell
ability Store v where
  get : v
  put : v -> ()
```

💡 Notice that this ability has a type parameter, `v`.  Abilities can have these, just like type declarations can.  

The `Store` ability can be implemented using handlers, even though Unison does not offer mutable state as a language primitive - we'll see the implementation later.  

Here's an example, using `Store` to help label a binary tree with numerical indices, in left-to-right ascending order.

``` haskell
type Tree a = Branch (Tree a) a (Tree a) | Leaf

labelTree : Tree a ->{Store .base.Nat} Tree (a, .base.Nat)
labelTree t =
  use Tree Branch Leaf
  case t of
    Branch l v r ->
      use .base.Nat +
      l' = labelTree l
      n = Store.get
      Store.put (n + 1)
      r' = labelTree r
      Branch l' (v, n) r'
    Leaf -> Leaf
```

💡 Observe how the first branch of the `case` statement includes four side-effecting statements - the two lines with recursive calls to `labelTree`, and the lines in between.  Unison supports these **blocks** of statements, and handles the statements in sequence, because order of execution is important when running side-effecting code.  Note that the last line is in this case a non-side-effecting expression - the value of the block is just the value of this final expression.

> Note, the `'` in the identifiers `l'` and `r'` here are just part of the names - nothing to do with delay syntax this time.

#### `IO`

The main reason for having abilities is to give our programs a way of having an effect on the world outside the program.  There is a special ability, called `IO` (for 'Input/Output'), which lets us do this.  It's built into the language and runtime, so it's not defined and implemented in the normal way, but we can take a look at its ability declaration.

``` haskell
.base.io> view IO

  ability IO where
    send_ :
      Socket
      -> base.Bytes
      ->{IO} base.Either Error base.()
    getLine_ :
      Handle ->{IO} base.Either Error base.Text
    openFile_ :
      FilePath
      -> Mode
      ->{IO} base.Either Error Handle
    throw : Error ->{IO} a
    fork_ :
      '{IO} a ->{IO} base.Either Error ThreadId
    systemTime_ : {IO} (base.Either Error EpochTime)

    -- ... and many more requests, omitted here for brevity.
```

The `IO` ability spans many different types of I/O - the snippet above shows sockets, files, exceptions, and threads, as well as the system clock.  

> Typically you access these requests via the helper functions in the `.base.io` namespace, e.g. `.base.IO.systemTime : '{IO} EpochTime`.

So, since all the ways in which we can interact with the world are captured in the `IO` ability, why do we ever need any other abilities?  There are several reasons.
1. We don't want to write all our code in terms of low-level concepts like files and threads.  We want higher-level abstractions, for example persistent distributed stores for typed data, and stream-based concurrency.  The low-level stuff is what we're used to from traditional programming environments, but we want to hide it behind powerful libraries, written in Unison, that expose better abstractions.  
2. We don't want `{IO}` to feature too often in the type signatures of the functions we write, because it doesn't tell us much.  Since `IO` contains so many different types of I/O, it leaves the behavior of our functions very unconstrained.  We want to use our type signatures to document and enforce the abiity requirements of our functions in a more fine-grained way.  For instance, it's useful that we know, just by looking at its signature, that `tomorrow : '{SystemTime} .base.Nat` isn't going to write to file or open a socket.  If we instead had `tomorrow : '{IO} .base.Nat`, then we'd have no such guarantee, without going and inspecting the code.  
3. Some things can be expressed well using abilities, but *don't* require interaction with the outside world. `Store` is an example.  

This leads us to a common pattern: 

👉 Typically, one ability is implemented by building on top of another.  And often, when we get down to the bottom of the pile, we'll find `IO`.  

For example, the handler for our `SystemTime` ability is going to require the `IO` ability, and it's going to call `.base.io.systemTime`.

In terms of the architecture of our programs, this typically means that the top level entry points for our 'business logic' are annotated with all the fine-grained abilities our program can use, like this:

``` haskell
placeOrder : Order ->{Database, Log, TimeService, AuthService} OrderConfirmation
```

And then we have one or more functions to wrap that logic, invoking handlers to collapse the signature down to one using only `IO`, like this:

``` haskell
orderServer : ServerConfig ->{IO} ()
```

> ##### Executing a Unison program
> 
> Here's the help for `ucm`'s `execute` command.
> ```
> .> help execute
> 
>   `execute foo` evaluates the Unison expression `foo` of type `()` with access to the `IO` ability.
> ```
>
> This shows us that we *need* to collapse our functions down to something like `orderServer`, so Unison knows how to run them.  

#### `Log`

Here's an example of an ability to let us append text to a log - for example a log file kept on disk.

``` haskell
ability Log where
  log : .base.Text -> ()
```

You could imagine the handler decorating the text with timestamps and other useful contextual information.  

The `Log` ability is typical of the class of abilities which let the program emit a sequence of data or commands.  The information flow is purely *out* of the program using the ability.  Contrast this with the `Store` and `IO` abilities, in which the flow of information is two-way, both in and out of the program.  

Most abilities are concerned with emitting and/or receiving data from/to the program.  However, abilities can do more than that: they can affect the program's control flow in ways that a regular function can't, as shown in the next example.  

#### `Abort` and `Exception`

The `Abort` ability lets us write programs which can terminate early.  

``` haskell
ability Abort where
  abort : a
```

Here's `Abort` in action:

``` haskell
use .base

getName : Input ->{Abort} Text
getName i = name = if not (valid i)
                   then Abort.abort
                   else extract "name" i
            canonicalName name

handleInput : Input ->{Abort} ()
handleInput i = name = getName i
                handleRequest name i
```

Suppose we're running `handleInput`, and we hit the `not (valid i)` error case inside `getName`: then we call `Abort.abort` and exit immediately.  Execution resumes from after the first enclosing `Abort` handler.  So, in this case, we exit both `getName` and `handleInput` immediately, since there's no handler in between the two.

> Note that the `abort` request has polymorphic type, `abort : a`.  This means it can be used in any context, and still typecheck.  It doesn't actually need to be able to return an `a`, because computation is not going to contine after the call to `abort`.  In `getName`, `abort` is being used where a `Text` is required, so `a` is instantiated to `Text`.  

There's a variant of `Abort`, which lets you provide a value to describe what's happened - this is analogous to the exception handling provided in some other languages.  

``` haskell
ability Exception e where
  throw : e -> a
```

😎 The ability mechanism is sufficiently general and powerful that what might otherwise be a whole separate single-purpose language feature, exception handling, instead becomes a few lines of library code.  Isn't that cool??

#### `Choice`

Here's another example - shown here to demonstrate further the idea of an ability affecting control flow.  

``` haskell
ability Choice where
  choose : .base.Boolean
```

There's a handler for this ability (which we'll see later), which gives the program not just one Boolean value after a call to `choose`, but both.  It then tries continuing the program under *both* conditions.  Each successive call to `choose` is a fork in the tree of possibilities.  The handler collects all the results from all the possible execution paths.  

This trick can be neat for exhaustively exploring a space of possibilities, for example to optimize some decision.  

### More on abilities in type signatures

#### Pure functions vs inferred abilities

Empty ability lists are used for declaring pure functions.  For example:

``` haskell
inverse : Matrix ->{} Matrix
```

The typechecker then enforces that `inverse` does not require any abilities.  

👉 The signature `A ->{} B` is different from the signature `A -> B`.

The former is the type of a pure function.  When you write the latter, you're asking for the ability list to be inferred by the Unison type-checker.  

👉 On code that you write, the signature `A -> B` doesn't mean 'no abilities', but rather that Unison will determine the ability list itself.  

This is an important distinction, and easy to forget, because the signature `A -> B` doesn't contain any visual cues to think about abilities.  

We'll learn more about the ability inference mechanism shortly, in [ability inference and generalization](#ability-inference-and-generalization).

#### Ability lists can appear before each function argument

So far we've seen functions whose types include one ability list, like so:

``` haskell
orderServer : ServerConfig ->{IO} ()
```
But the following is also possible:

``` haskell
orderServer' : ServerConfig ->{Log} '{IO} ()
```

> 🐘 This type signature is equivalent to `ServerConfig ->{Log} () ->{IO} ()`.

`orderserver'` is a function which, when partially applied to its first (`ServerConfig`) argument, can produce log messages, before finally yielding a function of type `'{IO} ()`.  That second function can then be forced, i.e. applied to `()`, to actually carry out some `IO`.  

The first application requires (only) the `Log` ability, and the second requires (only) the `IO` ability.  This is a useful distinction.  In this case, it tells us we can set up an order server, involving inspecting some configuration, in a computation that does some logging but is otherwise pure.  Only the second stage might do unrestricted `IO`.  

See [enclosing lambdas](#enclosing-lambdas) for how to _define_ a function like `orderServer'`.
TODO fix link

It's useful to keep the following in mind when reading type signatures.  

👉 Every `->` and `'` has an ability list `{…}` logically attached to it, describing the abilities required for applying the function to the preceding argument.  

'Logically', because as we've seen, the `{…}` can be left unspecified when writing code, to ask Unison to infer what it should say.  This process is described in the next section.  

#### Ability inference and generalization

Unison can do two levels of type inference for you.  The first is to infer the complete signature of your definition.

``` haskell
retries = 3
-- inferred type: .base.Nat
```

The second is to infer ability lists, wherever you have left them unspecified.  

```haskell
use .base

incrementP : Nat -> Nat
incrementP x = io.printLine "incrementP"
               x + 1
-- inferred type: Nat ->{io.IO} Nat               
```

Unison can see, from the use of `io.printLine`, that `incrementP` requires the `IO` ability.  

🚧 It's arguably surprising that Unison may infer a concrete ability for a function for which you provided a `Nat -> Nat` signature.  In future Unison will emit a message to say that it's done this.  ([#717](https://github.com/unisonweb/unison/issues/717))

When you do `add incrementP`, Unison will report the actual inferred type, `Nat ->{io.IO} Nat`.

So what does a plain `->` or `'` mean, when you see it after doing an `add`?  In this context it *does* mean a pure function - it's equivalent to `->{}` or `'{}`.

👉 When you *give* Unison a plain `->` or `'` (with no `{…}`) you're asking it to infer some abilities.  When Unison gives *you* a plain `->` or `'`, it means `->{}` or `'{}`.

So in particular, this means that if you type `->{}`, Unison can render it back to you as just `->`.  

> 🐞 Note, Unison can currently sometimes fail to output its inferred abilities when you do `view` or `edit` (although it does correctly output them at the `ucm` command line when you typecheck your code or do `add`/`update`.)  This is due to Unison issue [#703](https://github.com/unisonweb/unison/issues/703).  However, it will re-run its inference again when you next add the code.

##### Higher-order functions and ability polymorphism

Here's how Unison infers the type of `.base.List.map`, a higher-order function:

``` haskell
base.List.map : (a -> b) -> [a] -> [b]
-- inferred type: (a ->{𝕖} b) -> [a] ->{𝕖} [b]
```

It's added ability lists including a type variable, `𝕖`, in a process called **ability generalization**.  This is saying that, whatever the required abilities of the input function, the overall invocation of `map` will have the same requirements.

So for example, `'(base.List.map base.io.printLine ["Hello", "world!"])` has type `'{IO} [()]` - it requires `IO`, because it calls `printLine` which requires `IO`.  

We say that `base.List.map` is **ability polymorphic**: even though the function itself is in a sense pure, it can be used in a side-effecting way, depending on the ability requirements of its argument.  

The generalization process can work in tandem with inferring concrete abilities - for example:

``` haskell
applyP f x = .base.io.printLine "applyP"
             f x
-- inferred type: (i ->{𝕖, .base.io.IO} o) -> i ->{𝕖, .base.io.IO} o
```

This is saying that `applyP` requires `IO`, combined with whatever other abilities (`𝕖`) are required by its first argument.  (The combination process is a set union, so if `𝕖` also includes `IO`, then `IO` still only appears once in the resulting type.)

#### Enclosing lambdas  TODO fix title

TODO abilities only mattering on signatures for lambdas (see slack discussion and ability typechecking doc)

not getting confused when interpreting local definition signatures

TODO somewhere that detail about {A} X being a subtype of X, so beware ascribing your own ability signatures - where can you put too many abilities and it pass OK? 

how to define orderServer'

## Invoking handlers

TODO

(does this require understanding the type signature of the handler? Hopefully not)
Example of a test handler vs a ‘real’ handler
Examples of converting abilities out into regular types, eg Abort to Maybe
Getting IO provided at the top level
stacked handlers
providing the initial state
choose and revisit running examples
...

## Writing handlers

TODO
Matching on the requests, and a pure case
Continuations
Tail position
Being able to use the continuation 0 or 2+ times.  
[Performance/implementation/optimization futures?]
...

> ⚙️ Interestingly, you can define your own handler for `IO`!

proxy handler pattern

## Recap - worked example

TODO Maybe Send/Receive + Log ?