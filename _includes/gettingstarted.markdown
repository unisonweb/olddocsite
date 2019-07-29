# A tour of Unison

This guide assumes you've already gone through the steps in [the quickstart guide](quickstart.html). We recommend going through that guide before continuing along here.

The source for this document is [on GitHub][on-github]. Feedback and improvements are most welcome!

[repoformat]: todo
[on-github]: todo
[roadmap]: todo
[quickstart]: quickstart.html
[langref]: languagereference.html

### 🧠 The big idea

If there is one motivating idea behind Unison, it's this: the technology for creating software should be _thoughtfully crafted_ in all aspects. Needless complexity and difficulties should be stripped away, leaving only that exhilerating creative essence of programming that made many of us want to learn this subject in the first place! Or at the very least, if we can't have this, let's have programming be _reasonable_. "Well, it was done this way in the 70s and no one's really bothered to really revisit it" is not a good reason for continuing to do something that makes programming worse.

> 🧐 That said, it's sensible to make decisions about when and where to innovate and not try to innovate All The Things right now. But let's be honest that this is what we're doing... and not forget to keep making things better later!

Okay, but if there is one big _technical_ idea behind Unison, explored in pursuit of this overall goal, it's this: __Unison definitions are identified by content.__ Each Unison definition is some syntax tree, and by hashing this tree in a way that incorporates the hashes of all that definition's dependencies, we obtain the Unison hash which uniquely identifies that definition. This is the basis for some serious improvements to the programmer experience: it eliminates builds and most dependency conflicts, allows for easy dynamic deployment of code, typed durable storage, and lots more.

But this one technical idea is also a bit weird. The implications of it are weird! Consider this: if definitions are identified by their content, there's no such thing as changing a definition, there's only introducing new definitions. What can change is how we map definitions to human-friendly names. e.g. `x -> x + 1` (a definition) vs `Int.increment` (a name we associate with it for the purposes of writing and reading other code that references it). An analogy: Unison definitions are like stars in the sky. We can discover new stars and create new star maps that pick different names for the stars, but the stars exist independently of what we choose to call them.

The longer you spend with this weird idea, the more the niceness of it starts to take hold of you. You start seeing it everywhere: "wow, this would be so much easier with the Unison approach". And you start wanting to see the implications of it worked out in detail...

It does raise lots of questions, though: like even if definitions themselves are unchanging, we certainly may change which definitions we are interested in and give nice names to. How does that work? How do I refactor or upgrade code? Is the codebase still just a mutable bag of text files, or do we need something else?

We __do__ need something else to make working with content-addressed code nice. In Unison we call this something else the _Unison codebase manager_.

### 👋 to the Unison codebase manager

When first launching Unison in a new directory, we get a message like:

```
No codebase exists here so I'm initializing one in:
.unison/v1
```

What's happening here? This is the Unison _codebase manager_ starting up and initializing a fresh codebase. We're used to thinking about our codebase as a bag of text files that's mutated as we make changes to our code, but in Unison the codebase is represented as a collection of serialized syntax trees, identified by a hash of their content and stored in a collection of files in that `.unison/v1` directory.

The Unison codebase format has a few key properties:

* It is _append-only_: once a file in the `.unison` directory is created, it is never modified, and files are always named uniquely and deterministically based on their content.
* As a result, a Unison codebase can be versioned and sync'd with Git or any similar tool and will never generate a conflict in those tools. (Though diverging concurrent edits are still a fact of life, they don't show up as a Git conflict and are surfaced by Unison via a separate `todo` command in the codebase manager.)

> 🐘 Remember that `pull git@github.com:unisonweb/unisonbase.git` we used in the [quickstart guide][quickstart]. That used git behind the scenes to sync new definitions from the remote Unison codebase to the local codebase.

Because of the append-only nature of the codebase format, we can cache all sorts of interesting information about definitions in the codebase and _never have to worry about cache invalidation_. For instance, Unison is a statically typed language and we know the type of all definitions added to the codebase (the codebase is always in a well-typed state). So one thing that's useful and easy to maintain is an index that lets us query for definitions in the codebase by their type. Try out the following commands (new syntax is explained below):

__unison__
```
.> find : [a] -> [a]

  1. base.Heap.sort : [a] -> [a]
  2. base.List.distinct : [a] -> [a]
  3. base.List.reverse : [a] -> [a]
  4. base.Heap.sortDescending : [a] -> [a]

.> view 3

  base.List.reverse : [a] -> [a]
  base.List.reverse as =
    use base.List cons
    base.List.foldl (acc a -> cons a acc) [] as
```

Here, we did a type-based search for functions of type `[a] -> [a]`, got a list of results, and then used the `view` command to look at the nicely formatted source code of one of these results. Let's introduce some Unison syntax:

* `base.List.reverse : [a] -> [a]` is the syntax for giving a type signature to a definition. We sometimes pronounce the `:` symbol as "has type", as in "reverse has the type `[a] -> [a]`".
* `[Nat]` is the syntax for the type which is lists of natural numbers (terms like `[0,1,2]` and `[3,4,5]`, and `[]` will have this type), and more generally `[Foo]` is the type of lists whose elements have type `Foo`.
* Any lowercase variable in a type signature is assumed to be _universally quantified_, so `[a] -> [a]` really means and could be written `forall a . [a] -> [a]`, which is the type of functions that take a list whose elements are any type, and return a list of elements of that same type.
* `base.List.reverse` takes one parameter, called `as`. The stuff after the `=` is called the _body_ of the function, and here it's a _block_, which is an indentation demarcated  gives one parameter.
* The `acc a -> ..` is the syntax for an anonymous function.
* Function arguments are separated by spaces and function application binds tighter than any operator, so `f x y + g p q` parses as `(f x y) + (g p q)`. You can always use parentheses to control grouping more explicitly.
* `use base.List cons` lets us reference `base.List.cons` using just `cons`. Import statements like this can be placed in any Unison block; they don't need to go at the top of your file.

> Try doing `view base.List.foldl` if you're curious to see how it's defined.

### Names are stored separately from definitions so renaming is fast and 100% accurate

The Unison codebase, in its definititon for `reverse`, doesn't store names for the definitions it depends on (like the `foldl` function); it references these definitions via their hash. As a result, changing the name(s) associated with a definition is easy as pie.

Let's try this out. `reverse` is defined using `List.foldl`. Geez, "foldl", what the heck is that?? Down with pointless abbreviations! Let's rename that to `List.foldLeft`. Try out the following command (you can use tab completion here to help if you like):

```
.> move.term base.List.foldl base.List.foldLeft

  Done.

.> view base.List.reverse

  base.List.reverse : [a] -> [a]
  base.List.reverse as =
    use base.List cons
    base.List.foldLeft (acc a -> cons a acc) [] as
```

What's happening here? Notice that `view` shows the `foldLeft` name now, so the rename has taken effect. Nice!

To make this happen Unison just changed the name associated with the hash of `foldl` _in one place_. The `view` command just looks up the names for the hashes on the fly, right when it's printing out the code.

This is important: Unison __isn't__ doing a bunch of text munging on your behalf, updating possibly thousands of files (sounds hard!), generating a huge textual diff, and also breaking a bunch of downstream library users of yours that are still expecting that definition to be called the old name. That would be crazy, right?

So rename and move things around as much as you want. Don't worry about picking the perfect name at first. Give the same definition multiple names if you want, it's all good!

> ☝️ Using `alias.term` instead of `move.term` introduces a new name for a definition without removing the old name(s).

> 🤓 If you're curious to learn about the guts of the Unison codebase format, you can check out the [v1 codebase format specification][repoformat].

Okie dokie, let's learn more about Unison's interactive way of writing and editing code:

### Unison scratch files are like spreadsheets and replace the usual read-eval-print-loop

The codebase manager lets you make changes to your codebase and explore the definitions it contains, but it also listens for changes to any file ending in `.u` in the current directory (including any subdirectories). When any such file is saved (which we call a "scratch file"), it parses and typechecks that file. Let's try this out.

Keep your `unison` terminal running and open up a file, `scratch.u` (or `foobar.u`, or whatever you like) in your preferred editor, and put the following in your scratch file:

**scratch.u**
```
twice x = x * 2

--- anything below this is ignored by Unison
```

This defines a function called `twice`. It takes an argument called `x` and it returns `x` multiplied by two.

When you save the file, Unison replies:

**unison**
```
✅

I found and typechecked these definitions in ~/Dropbox/projects/unison/scratch.u. If you do an
`add` or `update` , here's how your codebase would change:

  ⍟ These new definitions are ok to `add`:

    twice : Nat -> Nat

Now evaluating any watch expressions (lines starting with `>`)... Ctrl+C cancels.
```

It typechecked `twice` and inferred that it takes a natural number and returns a natural number, so it has the type `Nat -> Nat`. It also tells us that `twice` is "ok to `add`". We'll do that shortly, but first, let's try calling our function right in the `scratch.u` file, just by starting a line with `>`:

**scratch.u**
```
twice x = x * 2

> twice 47
```

And Unison replies:

**unison**
```
3 | > twice 4
      ⧩
      8
```

The line `> twice 4` starting with a `>` is called a "watch expresion", and Unison uses these watch expressions instead of having a separate read-eval-print-loop (REPL). The code you are editing can be run interactively, right in the same spot as you are doing the editing, with a full text editor at your disposal, with the same imports all in scope, all without needing to switch to a separate tool.

__Question:__ do we really want to reevaluate all watch expressions on every file save? What if they're expensive? Luckily, Unison keeps a cache of results for expressions it evaluates, keyed by the hash of the expression (you can clear this cache at any time without ill-effects). If it has a result for a hash, it returns that instead of evaluating the expresion again. So you can think of and use your Unison `.u` scratch files a bit like spreadsheets, which only recompute the minimal amount when dependencies change!

> 🤓 There's one more ingredient that makes this work  effectively, and that's functional programming. When an expression has no side effects, its result is deterministic and you can cache it as long as you've got a good key to use for the cache, like the Unison content-based hash. Unison's type system won't let you do I/O inside one of these watch expressions or anything else that would make the result change from one evaluation to the next.

> 😤 How many times have you started doing some exploration in a REPL only to become irritated as you realize "geez, I shoulda just put this in a file". No more!

Let's try out a few more examples:

**scratch.u**
```
-- A comment, ignored by Unison

use base.List -- wildcard import
-- use base.List reverse size foldLeft -- specific imports

> reverse [1,2,3,4]
> 4 + 6
> 5.0 / 2.0
> not true
```

**unison**
```
✅

~/unisoncode/scratch.u changed.

Now evaluating any watch expressions (lines starting with
`>`)... Ctrl+C cancels.

  6 | > reverse [1,2,3,4]
        ⧩
        [4, 3, 2, 1]

  7 | > 4 + 6
        ⧩
        10

  8 | > 5.0 / 2.0
        ⧩
        2.5

  9 | > not true
        ⧩
        false
```

### Testing

Let's add add a test for our `twice` function:

**scratch.u**
```
twice x = x * 2

use test.v1 expect run

test> tests.twice.ex1 = run (expect (4 == 8))

--- Unison ignores anything below here
```

Save the file, and Unison comes back with:

**unison**
```
7 | test> tests.twice.ex1 = run (expect (twice 4 == 8))

✅ Passed : Passed 1 tests.
```

Some syntax notes:

* We're using explicit imports here (importing just the `expect` and `run` functions), but `use test.v1` with no symbols after would be a "wildcard" import and would let us access all the definitions starting with `test.v1` without needing a qualified name.
* The `test>` prefix tells Unison that what follows is a test watch expression.
* The `---` is called "the fold". Unison will ignore any code below the fold.

The `expect` function has type `Boolean -> Test`. It takes a `Boolean` expression and gives back a `Test`, which can be `run` to produce a list of test results, of type `[base.Test.Result]` (try `view base.Test.Result`). In this case there was only one result, and it was a passed test.

Let's test this a bit more thoroughly. `twice` should have the property that `twice a + twice b == twice (a + b)` for all choices of `a` and `b`. The testing library supports writing property-based tests like this. There's some new syntax here, explained afterwards:

**scratch.u**
```
twice x = x * 2

use test.v1 expect run nat

test> tests.twice.ex1 = run (expect (twice 4 == 8))

test> tests.twice.prop1 =
  go _ = a = !nat
         b = !nat
         expect (twice a + twice b == twice (a + b))
  runs 100 go
```

**Unison**
```
8 |   go _ = a = !nat

✅ Passed : Passed 100 tests. (cached)
```

This will test our function with a bunch of different inputs. Syntax notes:

* The Unison block which begins after an `=` begins a Unison block, which can have any number of _bindings_ (like `a = ...`) all at the same indentation level, terminated by a single expression (here `expect (twice ..)`), which is the result of the block.
* You can call a function parameter `_` if you just plan to ignore it. Here, `go` ignores its argument; it's just a delayed computation that will be evaluated multiple times by the `runs` function.
* `!expr` means the same thing as `expr ()`, we say that `!expr` _forces_ the delayed computation `expr`.
* Note: there's nothing special about the names `tests.twice.ex1` or `tests.twice.prop1`; we could call those bindings anything we wanted. Here we just picked some uncreative names just based on the function being tested. Use whatever naming convention you prefer.

`nat` is imported from `test.v1` - `test.v1.nat`. It's a _generator_ of natural numbers. `!nat` samples one of these numbers.

The `twice` function and the tests we've written for it are not yet part of the codebase. So far they only exists in our scratch file. Let's add it now. Switch to the Unison console and type `add`. You should get something like:

```
.> add

  ⍟ I've added these definitions:

    tests.twice.ex1    : [base.Test.Result]
    tests.twice.prop1  : [base.Test.Result]
    twice              : base.Nat -> base.Nat
```

You've just added a new function and some tests to your Unison codebase. Try typing `view twice` or `view tests.twice.prop1`. Notice that Unison inserts precise `use` statements when rendering your code. `use` statements aren't part of your code once it's slurped up into the codebase. When rendering code, a minimal set of imports is inserted automatically by the code printer, so you don't have to be so precise with your imports when entering in the code - use wildcard imports as much as you wish.

If you type `test` at the Unison prompt, it will "run" your test suite:

**Unison**
```
.> test

  Cached test results (`help testcache` to learn more)

  ◉ tests.twice.ex1       : Passed 1 tests.
  ◉ tests.twice.prop1     : Passed 100 tests.

  ✅ 2 test(s) passing

  Tip:  Use view tests.twice.ex1 to view the source of a test.
```

But actually, it didn't need to run anything! All the tests had been run previously and cached according to their Unison hash. In a purely functional language like Unison, tests like these are deterministic and can be cached and never run again. No more running the same tests over and over again!

### The Unison namespace, paths, and imports

Now that we've added our `twice` function to the codebase, how do we reference it elsewhere?

The _Unison namespace_ is the mapping from names to definitions. Names in Unison look like this: `math.sqrt`, `.base.Int`, `base.Nat`, `base.Nat.*`, `++`, or `frobnicate`. That is: an optional `.`, followed by one or more segments separated by a `.`, with the last segment allowed to be an operator name like `*` or `++`.

We often think of these names as forming a tree, much like a directory of files, and names are like file paths in this tree. _Absolute_ names (like `.base.Int`) start with a `.` and are paths from the root of this tree and _relative_ names (like `math.sqrt`) are paths starting from the "current path", which you can set using the `path` command:

```
.> path mylibrary

  ☝️  The namespace .mylibrary is empty.

.mylibrary>
```

Notice the prompt changes to `.mylibrary>`, indicating your current path is now `.mylibrary`. When editing scratch files, any relative names not locally bound in your file will be resolved by prefixing them with the current absolute path of `.mylibrary`. And when you issue an `add` command, the definitions are put directly underneath this path. For instance, if we added `x = 42` to our scratch file and then did `.mylibrary> add`, that would create the definition `.mylibrary.x`.

> You can use `path .` to move back to the root.

When we added `twice`, we were at the root, so `twice` and its tests are directly under the root. To keep our namespace a bit tidier, let's go ahead and move our definitions into the `mylibrary` namespace:

```
.mylibrary> move.term .twice twice

  Done.

.mylibrary> find

  1.  twice  : .base.Nat -> .base.Nat

.mylibrary> move.path .tests tests

  Done.
```

We're using `.twice` to refer to the `twice` definition directly under the root, and then moving it to the _relative_ name `twice`. When you're done shuffling some things around, you can use `find` with no arguments to view all the definitions under the current namespace:

```
.mylibrary> find

  1.  tests.twice.ex1  : [.base.Test.Result]
  2.  tests.twice.prop1  : [.base.Test.Result]
  3.  twice  : .base.Nat -> .base.Nat
```

Also notice that we don't need to rerun our tests after this reshuffling. The tests are still cached:

```
.mylibrary> test

  Cached test results (`help testcache` to learn more)

  ◉ tests.twice.ex1       : Passed 1 tests.
  ◉ tests.twice.prop1     : Passed 100 tests.

  ✅ 2 test(s) passing

  Tip:  Use view tests.twice.ex1 to view the source of a test.
```

We get this for free because the test cache is keyed by the hash of the test, not by what the test is called.

> ☝️  The `use` statement can do absolute imports as well, for instance `use .base.List map`.

When you're starting out writing some code, it can be nice to just put it in a temporary namespace, perhaps called `temp` or `scratch`. Later, without breaking anything, you can move that namespace or bits and pieces of it elsewhere, using the `move.term`, `move.type`, and `move.namespace` commands.

## Modifying an existing definition

Instead of starting a function from scratch, often you just want to slightly modify something that already exists. Here we'll make a trivial change to our `twice` function. Try doing `edit twice` from your prompt (note you can use tab completion):

```
.mylibrary> edit twice
  ☝️

  I added these definitions to the top of ~/scratch.u

    twice : .base.Nat -> .base.Nat
    twice x =
      use .base.Nat *
      x * 2

  You can edit them there, then do `update` to replace the definitions currently in this branch.
```

This copes the pretty-printed definition of `twice` into you scratch file above the fold. Let's edit `twice` and instead define it as `x + x`.

**scratch.u**
```Haskell
twice : .base.Nat -> .base.Nat
twice x = x + x
```

> Notice Unison has put the correct type signature on `twice`. The absolute names `.base.Nat` look a bit funny. We will often do `use .base` at the top of our file to refer to all the basic functions and types in `.base` without a fully qualified name.

**Unison**
```
✅

I found and typechecked these definitions in ~/Dropbox/projects/unison/scratch.u. If you do an
`add` or `update` , here's how your codebase would change:

  ⍟ These new definitions will replace existing ones of the same name and are ok to `update`:

    twice  : .base.Nat -> .base.Nat

Now evaluating any watch expressions (lines starting with `>`)... Ctrl+C cancels.
```

Notice the message says that `twice` is "ok to `update`". Let's try that:

```
.mylibrary> update

  ⍟ I've updated to these definitions:

    twice  : .base.Nat -> .base.Nat
```

If we rerun the tests, the tests won't be cached this time, since one of their dependencies has actually changed:

```
.mylibrary> test

TODO: fixme - update doesn't propagate
```

Notice the message indicates that the tests weren't cached. If we do `test` again, we'll get the newly cached results.

## Publishing code

Unison code is published just by virtue of it being pushed to github; there's no separate publication step. You might choose to make a copy of your namespace. Let's go ahead and do this:

```
.mylibrary> path .
.> copy.path mylibrary mylibrary.releases.v1

  Done.

.> cd mylibrary.releases.v1
.mylibrary.releases.v1> find

  1.  tests.twice.ex1  : [.base.Test.Result]
  2.  tests.twice.prop1  : [.base.Test.Result]
  3.  twice  : .base.Nat -> .base.Nat
```

But this is just a naming convention, there's nothing magic happening here.

Now let's publish our `mylibrary` to a fresh Unison repo. First, fork the Unison base library, using the button below (you'll need a GitHub account to do this). This creates a valid minimal Unison codebase repo that you can push to:

<iframe src="https://ghbtns.com/github-btn.html?user=unisonweb&repo=unisonbase&type=fork&count=true&size=large" frameborder="0" scrolling="0" width="158px" height="30px"></iframe>

> ☝️ There's nothing special about using GitHub here; you can also host your Unison git repos elsewhere. Just use whatever git URL you'd use on your git hosting provider for a `git push`.

After you've forked the base repo, you can push to it:

**Unison**
```
.mylibrary.releases.v1> path .
.> push git@github.com:<yourgithubuser>/unisonbase.git
```

You'll see some git logging output. Your code is now live on the internet!

### Installing libraries written by others

This section under construction.

From the root, do:

```
.> pull git@github.com:<github-username>/unisonbase.git temp
..
.> move.path temp.myfirstlibrary.releases.v1 myfirstlibrary
.> delete.path temp
```

The namespace you created is now available under `.myfirstlibrary`, so `.myfirstlibrary.twice` will resolve to the function you wrote.

### What next?

* [The core language reference][langref] describes Unison's core language and current syntax in more detail.
* TODO: writing a more interesting library
