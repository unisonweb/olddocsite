# 3 minute quickstart guide

This short guide will have you downloading Unison, running your first program (a toy distributed mergesort implementation), and publishing your first definition. The focus here is on getting you up and running as quickly as possible. 🏎 More in-depth guides follow this one.

If you have any trouble with the process or ideas about how to improve this document, [come talk to us in Slack][slack]! Also this document is [on GitHub][on-github].


[slack]: https://unisonlanguage.slack.com
[mac-dl]: https://www.dropbox.com/s/6vl246m4faaps5k/unison?dl=0
[linux-dl]: todo
[windows-dl]: todo
[on-github]: todo
[guide]: gettingstarted.html

### Step 1: Download Unison

Download the `unison` executable for [Mac][mac-dl], [Linux][linux-dl], or [Windows][windows-dl] and then (optional) add it to your path.

### Step 2: Create your Unison codebase

Create a new directory, `unisoncode` (or any name you choose), then run the `unison` binary from within that directory. You'll see a note about "No codebase exists here so I'm initializing one..." and a welcome screen.

<script id="asciicast-IYWfFwIgyl9Gilk3ZExvLfOjg" src="https://asciinema.org/a/IYWfFwIgyl9Gilk3ZExvLfOjg.js" data-speed="2" data-cols="65" async></script>

### Step 3: Fetch and run a distributed mergesort example

At the Unison `.>` prompt, do:

```
.> pull git@github.com:unisonweb/unisonbase.git
``` 

to fetch a base library with the example you'll be running. You'll see some output from `git` in the background, and once that's done you can do `edit quickstart.dsort` to add the `dsort` distributed mergesort function to the top of a newly created _scratch file_, `scratch.u`:

<script id="asciicast-o9lfrfetnmUT4ArqdDFMXZkr9" src="https://asciinema.org/a/o9lfrfetnmUT4ArqdDFMXZkr9.js" data-speed="2" data-rows="30" data-cols="65" async></script>

Open that file and add the following _watch expression_ (a line starting with `>`) to the top, then save the file:

```
> runLocal '(quickstart.dsort (<) [8,2,3,1,4,5,6,7])
```

<script id="asciicast-aTn8qIa3DHaxhspsZJmXodfO7" src="https://asciinema.org/a/aTn8qIa3DHaxhspsZJmXodfO7.js" data-speed="2" async></script>

You should see your watch expression evaluate to a sorted list.  _Disclaimer:_ This example is just a toy that simulates execution locally and does no error handling, but it shows the general idea of being able to test Unison distributed programs locally (perhaps with simulated latency and failures injected) and then run the same programs unchanged atop an actual elastic source of distributed compute!

### Step 4: Publishing your first definition

Let's try publishing some code. First, fork the Unison base library, using the button below. This creates a valid minimal Unison codebase repo that you can push to:

<iframe src="https://ghbtns.com/github-btn.html?user=unisonweb&repo=unisonbase&type=fork&count=true&size=large" frameborder="0" scrolling="0" width="158px" height="30px"></iframe>

Then add the following to the top of your `scratch.u` file and save:

```haskell
firstlibrary.frobnicate x = x * 2

--- include this too, it's called "the fold"
```

Include that `---` at the end to ignore the `dsort` definition below (or you can delete the `dsort` definition - you can always get it back via `edit`). Then from the `unison` prompt:

```
.> add
.> push git@github.com:<your-githubusername>/unisonbase.git
```

Code is published just by being pushed to GitHub; there's no extra publication step. Others can use any namespace of definitions you've published just using `pull`. We'll cover how more about how library publishing and upgrades work in detail in a later guide, but the process of installing a new namespace of definitions looks like:

```
.> pull git@github.com:<github-username>/unisonbase.git install
.> move.path install.firstlibrary awesomelibrary
.> delete.path install
```

And then `awesomelibrary.frobnicate` will be resolved to the `frobnicate` definition you published.

### What next?

* Come [say hello in Slack][slack], tell us what you thought about this guide, and ask questions. 👋
* A [more leisurely guide][guide] to the Unison language and the `unison` command line tool. (25 minutes)
