# A tour of Unison

This guide assumes you've already gone through the steps in [the quickstart guide](quickstart.html). We recommend going through that guide before continuing along here.

The source for this document is [on GitHub][on-github]. Feedback and improvements are most welcome!

[repoformat]: todo
[on-github]: todo
[roadmap]: todo

### 🧠 The big idea

If there is one motivating idea behind Unison, it's this: the tools for making software should be _thoughtfully crafted_ in all aspects, with the goal of making the developer experience simpler, easier, more delightful, and getting us closer to that exhilerating creative essence of programming, the stuff made us all want to learn this amazing subject in the first place! Or if we can't have that, at the very least, let's have programming be _reasonable_ and not insane or arcane. "Well, it was done this way in the 70s and no one's really bothered to really revisit it" is not a good reason for continuing to do something that makes programming worse.

This is the philosphy behind Unison and we take it wherever it leads us! Who's with us??

> 🧐 Okay, but boiling the ocean can take a long time. It's sensible to make decisions about when and where to innovate and not try to innovate All The Things right now. But let's be honest that this is what we're doing... and not forget to keep making things better later!

Okay, but if there is one big _technical_ idea behind Unison, explored in pursuit of our overall goals, it's this: __Unison definitions are identified by content.__ Each definition is some syntax tree, hashed in a way that incorporates the hashes of all the dependencies of that definition and which is independent of the names of these definitions. A Unison hash uniquely identifies a Unison definition. This is the basis for some serious improvements to the programmer experience: it eliminates builds and eliminates dependency conflicts, allows for easy dynamic deployment of code, typed durable storage, and lots more.

But this one technical idea is also a bit weird. The implications of it are weird! Consider this: if definitions are identified by their content, there's no such thing as changing a definition, there's only introducing new definitions. What can change is how we map definitions to human-friendly names. e.g. `x -> x + 1` (a definition) vs `Int.increment` (a name we associate with it for the purposes of writing and reading other code that references it). An analogy: Unison definitions are like stars in the sky. We can discover new stars and create new star maps that pick different names for the stars, but the stars exist independently of what we choose to call them.

But the longer you spend with this weird idea, the more the niceness of it starts to take hold of you. You start seeing it everywhere: "wow, this would be so much easier with the Unison approach". And you start wanting to see the implications of it worked out in detail...

It does raise lots of questions, too: for one, how do you actually edit and refactor code deep down in the dependency graph of your code in this new world without it being unbelievably tedious? Is the codebase still just a mutable bag of text files, or do we need something else?

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

> 🐘 Remember that `pull git@github.com:unisonweb/unisonbase.git` we used in the [quickstart guide][quickstart]? That used git behind the scenes to sync new definitions from the remote Unison codebase to the local codebase.

Because of the append-only nature of the codebase format, we can cache all sorts of interesting information about definitions in the codebase and _never have to worry about cache invalidation_. For instance, Unison is a statically typed language and we know the type of all definitions added to the codebase (the codebase is always in a well-typed state). So one thing that's useful and easy to maintain is an index that lets us query for definitions in the codebase by their type. Try out the following commands:

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
    use base.List +:
    base.List.foldl (acc a -> a +: acc) [] as
```

Here, we did a type-based search for functions of type `[a] -> [a]`, got a list of results, and then used the `view` command to look at the nicely formatted source code of one of these results. We'll introduce more Unison syntax later, but a couple things:

* `base.List.reverse : [a] -> [a]` is the syntax for giving a type signature to a definition. We sometimes pronounce the `:` symbol as "has type", as in "reverse has the type `[a] -> [a]`".
* `[Nat]` is the syntax for the type which is lists of natural numbers (terms like `[0,1,2]` and `[3,4,5]`, and `[]` will have this type), and more generally `[Foo]` is the type of lists whose elements have type `Foo`.
* Any lowercase variable in a type signature is assumed to be _universally quantified_, so `[a] -> [a]` really means and could be written `forall a . [a] -> [a]`, which is the type of functions that take a list whose elements are any type, and return a list of elements of that same type.

### Names are stored separately from definitions so renaming is fast and 100% accurate

The Unison codebase, in its definititon for `reverse`, doesn't store the names for the definitions it depends on (like the `foldl` function). All definitions in Unison are given a unique, content-based hash, and references to dependencies use this hash. As a result, changing the name(s) associated with a definition is easy as pie.

Let's try this out. `reverse` is defined using `List.foldl`. Geez, "foldl", what the heck is that?? Down with pointless abbreviations! Let's rename that to `List.foldLeft`. Try out the following command (you can use tab completion here to help if you like):

```
.> move.term base.List.foldl base.List.foldLeft

  Done.

.> view base.List.reverse

  base.List.reverse : [a] -> [a]
  base.List.reverse as =
    use base.List +:
    base.List.foldLeft (acc a -> a +: acc) [] as
```

What's happening here? Notice that `view` shows the `foldLeft` name now, so the rename has taken effect. Nice!

To make this happen Unison just changed the name associated with the hash of `foldl` _in one place_. The `view` command just looks up the names for the hashes on the fly, right when it's printing out the code.

This is important: Unison __isn't__ doing a bunch of text munging on your behalf, updating possibly thousands of files (sounds hard!), generating a huge textual diff, and also breaking a bunch of downstream library users of yours that are still expecting that definition to be called the old name. That would be crazy, right?

So rename and move things around as much as you want. Don't worry about picking the perfect name at first. Give the same definition multiple names if you want, it's all good!

> ☝️ Using `alias.term` instead of `move.term` introduces a new name for a definition without removing the old name(s).

> 🤓 If you're curious to learn about the guts of the Unison codebase format, you can check out the [v1 codebase format specification][repoformat].

Okie dokie, let's go ahead and try out this `reverse` function and learn more about Unison's interactive way of writing and editing code:

### Unison scratch files are like spreadsheets and replace the usual read-eval-print-loop (REPL)

The codebase manager lets you make changes to your codebase and explore the definitions it contains, but it also listens for changes to any file ending in `.u` in the current directory (including any subdirectories). When any such file is saved, it parses and typechecks that file and runs any _watch expressions_, which are lines starting with `>`. Let's try this out.

Keep your `unison` terminal running and open up a file, `scratch.u` (or `frobnicate.u`, or whatever you like) in your preferred editor, and add a few watch expressions:

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

When you save the file, the codebase manager notices and responds:

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

Great! Unison evaluates all the expressions at once and prints out the results. The numbers on the left are the line numbers from the `scratch.u` file.

Unison uses this basic workflow instead of having a separate read-eval-print-loop (REPL): the code you are editing can be run interactively, right in the same spot as you are doing the editing, with a full text editor at your disposal and without needing to switch to a separate tool.

This is nice, but do we really want to reevaluate all these expressions on every file save? What if they're expensive? Luckily, Unison keeps a cache of results for expressions it evaluates, keyed by the hash of the expression (you can clear this cache at any time without ill-effects). If it has a result for a hash, it returns that instead of evaluating the expresion again. So you can think of and use your Unison `.u` scratch files a bit like spreadsheets, which only recompute the minimal amount when dependencies change!

> 🤓 There's one more ingredient that makes this work  effectively, and that's functional programming. When an expression has no side effects, its result is deterministic and you can cache it as long as you've got a good key to use for the cache, like the Unison content-based hash. Unison's type system won't let you do I/O inside one of these watch expressions or anything else that would make the result change from one evaluation to the next.

> 😤 How many times have you started doing some exploration in a REPL only to become irritated as you realize "geez, I shoulda just put this in a file". No more!

### Taking detours while working on a scratch file

Sometimes while in the middle of writing some code, your scratch file is in a broken state of things being half implemented, and you realize you want to switch contexts and try something else out or maybe tinker with a function you're thinking of using. Imagine you're write a big complicated function, called `transmogrify`. It's half done, with a bunch of syntax and/or type errors and things to fill in, but you realize you could use a function for generating the list `[0,1,2..n]`. First you might try out a type-based search:

```
.> find : [base.Nat]

  ☝️

  I couldn't find exact type matches, resorting to fuzzy
  matching...


  1. base.Bytes.fromList : [base.Nat] -> base.Bytes
  2. base.Bytes.toList : base.Bytes -> [base.Nat]
  3. base.List.range : base.Nat -> base.Nat -> [base.Nat]
```

> 🧪 It would be nice if you could specify some "default imports" to the codebase manager to avoid seeing these fully qualified names everywhere. That's not implemented yet unfortunately.

> 🏗 We are working on making documentation also available from within the `find` and `view` commands.

The search does fuzzy type matching if it can't find an exact match. `List.range` sounds promising, it takes two numbers and returns a `[Nat]`. Let's try it out. First, put a line at the top of your file with just three dashes in it:

**scratch.u**
```
---
use base.List

transmogrify f abracadabra x = ...
```

If you save that, Unison will say that it loaded the file but didn’t find anything. The `---` line is called "the fold", and it can be a convenient tool for working with Unison scratch files. Everything below the fold is ignored by Unison, but all the stuff you typed is still there in case you need it. We can test out the `range` function by putting a watch expression above the fold:


**scratch.u**
```Haskell
use base.List

> range 0 10
---
...
```

Function arguments are separated by spaces (`range 0 10` calls the `range` function, giving it `0` and `10`), and function application binds tighter than any operator, so `f x y + g p q` parses as `(f x y) + (g p q)`. You can always use parentheses to control grouping more explicitly. Unison responds

**Unison**
```
  ✅

  ~/unisoncode/scratch.u changed.

  Now evaluating any watch expressions (lines starting with
  `>`)... Ctrl+C cancels.

    3 | > range 0 10
          ⧩
          [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```

When you have to go on a detour, you can put the fold wherever is convenient (often right after your existing `use` statements saves you from needing to reestablish a similar set of imports). When you're done, you can delete the stuff above the fold or send it to the end of the file, then get back to what you were doing below the fold.

## Your first Unison code

It’s time to write some code. Put the following in your `scratch.u` file, above the fold:

**scratch.u**
```
twice x = x * 2
```

This defines a function called `twice`. It takes an argument called `x` and it returns `x` multiplied by two.

When you save the file, Unison replies:

**unison**
```
  ✅ I found and typechecked these definitions in
  ~/unisoncode/scratch.u:

       Legend:
    +  a new definition with a new name

    +  twice : Nat -> Nat
```

It typechecked `twice` and inferred that it takes a natural number and returns a natural number, so it has the type `Nat -> Nat`.

We can try calling our function right in the `scratch.u` file:

**scratch.u**
```
twice x = x * 2

> twice 47
```

And Unison replies:

**unison**
```
    3 | > twice 47
          ⧩
          94
```

We can add a test for this function.

**scratch.u**
```
twice x = x * 2

use Test

test> check (twice 4 == 8)
```

The `test>` prefix tells Unison that this is a test.

The `use Test` clause tells Unison that we’re going to use definitions from the `Test` namespace. This lets us say `check` instead of the function’s full name which is `Test.check`.

Save the file, and Unison comes back with:

**unison**
```
 ✅ Passed -  : Proved.
```

The `Test.check` function has type `Boolean -> [Test.Result]`. It takes a Boolean expression and gives back a list of test results. In this case the only result is that Unison was able to _prove_ that `twice 4` is in fact equal to `8`.

Let’s test this a bit more thoroughly.

**scratch.u**
```
twice x = x * 2

use Test
use Domain pairs nats

test> forAll 100 (pairs nats)
  (p -> case p of (a,b) -> twice a + twice b == twice (a + b))
```

This will test our function with a bunch of different inputs. `forAll 100 (pairs nats)` will generate 100 pairs of natural numbers to test with.

The last argument to `forAll` is a function literal. The function takes an argument called `p`, which will be a pair of `Nat`s. It takes the pair apart into its constituents `a` and `b` by pattern matching on it, and checks that `twice` distributes over addition.

The indentation on the second line of the test is significant and tells Unison that this line is part of the test on the previous line.

Save this, and we get:

**unison**
```
✅ Passed -  : Passed 100 tests.
```

All 100 of our tests passed!

The `twice` function is not yet part of the codebase. So far it only exists in our scratch file. Let’s add it now. Switch to the Unison console and type `add`. You should get:

```
+  twice : Nat -> Nat
```

You’ve just added a new function to your Unison codebase.

## Modifying an Existing Definition
Let’s keep going. Instead of starting a function from scratch, often you just want to slightly modify something that already exists.

For example, there is a function called `List.halve` that splits a list in half.

Here’s what it does:

**unison**
```
    1 | > halve [1,2,3,4,5,6]
          ⧩
          ([1, 2, 3], [4, 5, 6])

    2 | > halve ["a","b","c"]
          ⧩
          (["a"], ["b", "c"])
```

We can ask Unison for the definition:

**unison**
```
.> view List.halve

List.halve : [a] -> ([a], [a])
List.halve s =
  n = List.size s / 2
  (List.take n s, List.drop n s)
```

The first line, `List.halve : [a] -> ([a], [a])`, gives a type for the function. `List.halve` takes a value of type `[a]`, which is a list of values of type `a`, where `a` stands for any type whatsoever. And the returned value is of type `([a], [a])`, a pair of lists of the same type as the input.

The indentation here is significant, as the two lines of the function’s body form a _block_. A block can have any number of local definitions, and here `List.halve` declares a local variable `n` to be half the size of the list `s`. The value of the whole block is the value of its final expression, in this case a pair of lists. One list is constructed by taking `n` elements from the front of `s`, and the other by dropping `n` elements from the front of `s`.

Let’s say we want to make a new function `third`, which instead of chopping a list in half, chops one third of the list off the front. We can just modify the existing definition of `halve`.

Type `edit List.halve` at the Unison prompt. This will copy the definition of `List.halve` into your `scratch.u` file and put everything that was there previously below the fold.

Now, we could just modify it by replacing `2` with `3` on the 3rd line, but what if we want to write `quarter` later? Let’s abstract out the number and make a new, more general version of `List.halve`:

**scratch.u**
```
List.divide : Nat -> [a] -> ([a], [a])
List.divide d s =
  n = List.size s / d
  (List.take n s, List.drop n s)

List.halve = divide 2
List.third = divide 3
```

Save that and we get:

**unison**
```
    +  List.divide : Nat -> [a] -> ([a], [a])
    +  List.third  : [a] -> ([a], [a])
    Δ  List.halve  : [a] -> ([a], [a])
```

If you type `add` into Unison, it will add `List.divide` and `List.third` to the codebase, but it will refuse to add `List.halve` since a different definition already exists with that name.

We could try to delete the existing definition:

**unison**
```
.> delete.term List.halve

  ⚠️

  I couldn't delete

    1. List.halve : [a] -> ([a], [a])

  because it's still being used by these definitions:

    1. List.foldb : (a ->{𝕖} b) -> (b ->{𝕖} b ->{𝕖} b) -> b -> [a] ->{𝕖} b
```

So `List.halve` has some other things that depend on it, so we can’t delete it. But Unison can replace the old definition with our new one and automatically propagate the change to all transitive dependents of the old function.

First we use the `update` command to create a patch.

**unison**
```
.> update mypatch

     Legend:
  Δ  a new definition using an existing name (ok to `update`)

  Δ  List.halve : [a] -> ([a], [a])
```

Then we apply the patch using the `patch` command.

**unison**
```
.> patch mypatch

  ✅

  No conflicts or edits in progress.
```

We have now updated both `List.halve` and its dependent `List.foldb`.

## Publishing Code to the Internet
Let’s publish our new and improved functions for others to consume.

First, we should put our functions into their own namespace so we can package them up neatly. Fortunately adding new names for things in Unison is pretty trivial:

**unison**
```
.> alias.term List.halve Awesome.List.halve

  Done.

.> alias.term List.third Awesome.List.third

  Done.

.> alias.term List.divide Awesome.List.divide

  Done.
```

Then we can change to the `Awesome` namespace:

**unison**
```
.> cd Awesome
.Awesome> ls

  1. List.halve : [a] -> ([a], [a])
  2. List.third : [a] -> ([a], [a])
  3. List.divide : Nat -> [a] -> ([a], [a])
```

We can then push the contents of the  `Awesome` namespace to a Git repository. For example, to push to GitHub:

1. Make sure you have `git` installed and available on your path. See [Git - Installing Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) for how to do that.
2. Log in to your GitHub account and create a new repository.
3. Type `push git@github.com:username/reponame.git`, where `username` is your GitHub user name and `reponame` is the name of the repo you just created. The `push` command accepts any Git URL.

Unison calls out to `git` which generates a bunch of output. At the end, if everything is successful, you should see Unison respond with `Done.`

Your code is now live on the internet! Others can get your definitions of `halve`, `third`, and `divide` by issusing e.g. `pull git@github.com:username/reponame.git`.

## What next?
TODO: Where do we direct readers next after this?


### The Unison namespace, paths, and imports

Meta: this section seems boring and not needed...

The _Unison namespace_ is the mapping from names to definitions. Names in Unison look like this: `math.sqrt`, `.base.Int`, `base.Nat`, `base.Nat.*`, `++`, or `frobnicate`. That is: an optional `.`, followed by one or more segments separated by a `.`, with the last segment allowed to be an operator name like `*` or `++`.

We often think of these names as forming a tree, much like a directory of files, and names are like file paths in this tree. _Absolute_ names (like `.base.Int`) start with a `.` and are paths from the root of this tree and _relative_ names (like `math.sqrt`) are paths starting from the "current path", which you can set using the `path` command:

```
.> path base
.base>
```

Notice the prompt changes to `.base>`, indicating your current path is now `.base`. When editing scratch files, any relative names not locally bound in your file will be resolved by prefixing them with the current absolute path of `.base`.


You can always refer to definitions using some absolute name (try evaluating `.base.List.reverse [1,2,3]`) but the `use` statement lets us refer to things using a shorter, relative name. So we


Nothing too exciting here.

