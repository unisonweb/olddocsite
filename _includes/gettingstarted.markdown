# Getting Started with Unison
## Download and Install Unison
TBD

## Create Your Unison Codebase
Now that you have Unison installed, the first thing to do is to create a Unison codebase, which is where your Unison code is going to live.

Create an empty directory in a convenient location on your disk. Call this directory whatever you want. For example, you could call it `UnisonCodebase`:

**Command Line**
```bash
$ mkdir UnisonCodebase
$ cd UnisonCodebase
```

Now start up Unison by issuing the `unison`  command.

**unison**
```
$ unison
☝️  No codebase exists here so I'm initializing one in: .unison/v0

   _____     _
  |  |  |___|_|___ ___ ___
  |  |  |   | |_ -| . |   |
  |_____|_|_|_|___|___|_|_|

  Welcome to Unison!

  I'm currently watching for changes to .u files under
  ~/UnisonCodebase

  Type help to get help. 😎

.>
```


Unison is now waiting for your input in two different ways:
1. It’s listening for changes to files with the `.u` extension, anywhere under the `UnisonCodebase` directory (including any subdirectories).
2. It’s waiting for you to give a command at the prompt.

The first thing Unison did was create a codebase in `UnisonCodebase/.unison`. This already has a number of functions and data types in it to get you started.

## Evaluate Unison Expressions
While keeping Unison running in the terminal, open up a file called e.g. `UnisonCodebase/scratch.u`. Any file ending with `.u`  under the directory Unison says it’s watching will do.

Instead of typing Unison expressions at the prompt like in a traditional read-eval-print loop, you can ask Unison to evaluate expressions just by putting them in your `.u` file. If you begin a line with `>`, Unison will evaluate the rest of that line and print the result. For example, if you add this to `scratch.u` and save it:

**scratch.u**
```
> 4 + 6

> 4 - 5

> 5.0 / 2.0

> not true
```

Unison comes back with:

**unison**
```
  ✅ ~/UnisonCodebase/scratch.u changed.

  Now evaluating any watch expressions (lines starting with
  `>`)...

    1 | > 4 + 5
          ⧩
          9

    3 | > 4 - 5
          ⧩
          -1

    5 | > 5.0 / 2.0
          ⧩
          2.5

    7 | > not true
          ⧩
          false
```

Great! Unison evaluates all the expressions at once and prints out the results. The numbers on the left are the line numbers from the `scratch.u` file.

Now put a line at the top of your file with just three dashes in it.

```
---
> 4 + 6

> 4 - 5

> 5.0 / 2.0

> not true
```

If you save that, Unison will say that it loaded the file but didn’t find anything. The `---` line is called “the fold“, and it can be a convenient tool for working with Unison scratch files. Everything below the fold is ignored by Unison, but all the stuff you typed is still there in case you need it.

## Your First Unison Code
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
  ~/UnisonCodebase/scratch.u:

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
