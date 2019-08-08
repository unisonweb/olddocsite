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

The reason we can omit the name of the ability being defined from its requests' signatures is that, logically, it always needs to be there.  So we say it 'goes without saying', which saves us a bit of syntactic noise.

> 🤔 An ability declaration is the only place you'll see an ability list in a signature that doesn't follow the rule that 'ability requests exist in the context of a function processing one of its arguments' - i.e. the only place you can have a `{SystemTime}` not following a `->`.

### Examples of abilities

-- IO (sidebar on why it's special; idea of one ability implemented in terms of another), control flow (Abort, Send/Receive), at least some of: State/Store, Console IO, abilities with type parameters, abilities that are just sets of commands to emit, random/choice. Comment about effects with pure handlers.

-- Backing up the motivation, giving the picture of how programs should end up looking in the large (or later)

### More on abilities in type signatures

TODO somewhere that detail about {A} X being a subtype of X.

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