# Introducing abilities

TODO: doc purpose and context

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

It defines one *request*, `systemTime`, which returns the clock reading.  We'll come back to the term 'request' later.  

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

The `{SystemTime}` is an *ability list* - in this case a list of just one ability.  It's saying that `tomorrow` _requires_ the `SystemTime` ability - that ability needs to be available in functions that call `tomorrow`.  And it's also saying that the `SystemTime` ability is available for use within the definition of `tomorrow` itself.  

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

#### `IO`

The main reason for having abilities is to give our programs a way of having an effect on the world outside the program.  There is a special ability, called `IO` (for Input/Output), which lets us do this.  It's built into the language and runtime, so it's not defined and implemented in the normal way, but we can take a look at its ability declaration.

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
>
> ⚙️ Interestingly, you can define your own handler for `IO`!  (In theory this includes using the 'real' `IO` from within your handler, but note Unison issue [#697](https://github.com/unisonweb/unison/issues/697)).

#### `Abort`

TODO i.e. demonstrating that abilities can affect control flow

Maybe do choice/nondeterminism as well?

### More on abilities in type signatures

TODO somewhere that detail about {A} X being a subtype of X.

TODO abilities can be inferred even where a type signature is given.  Surprise about them being inferred as concrete effects, ticket 691.

TODO abilities only mattering on signatures for lambdas (see slack discussion and ability typechecking doc)

#### Ability lists on each argument

TODO
--including thinking of the {} as joined to the ->

#### Pure functions

TODO

#### Higher-order functions, ability polymorphism, and ability inference

TODO

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

## Recap - worked example

TODO Maybe Send/Receive + Log ?