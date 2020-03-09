---
title: "Genesis Flags"
date: 2018-04-08T03:44:21Z
tags: ["genesis", "golang"]
draft: false
---

# Genesis

Genesis is a project I’ve spent a great deal of time thinking about and working on for a while with little progress. I’m recycling my old Github blog [post](/blog/genesis-roadmap/) because it still highlights the overall design plan. I’ve since altered the project to use Golang instead of CPython. The change is inspired by a desire/need for improved performance, in my view Golang is the perfect tool to accomplish this goal and is the natural next step in my progression as a developer.

# Config files, CLI flags, and repeatability

With the decision to switch to Golang some necessary design choices had to be made. Due to the interactive and 'multi-phase' design of Genesis, it naturally lends itself to a single binary with an abundance of subcommands, such as `genesis render`, `genesis generate terrain` and so on.

After some research, an extremely appealing option for building the command-line interface came up: spf13's [cobra](https://github.com/spf13/cobra). This library is used by a lot of pretty big projects, including Hugo ( used to build the site you're reading right now ).

Due to the complex nature involved in each step of the world generation process, and considering one of the design goals is *repeatability*, I required a powerful yet flexible and reliable option for consuming and referencing configuration data. A user should be able to use interactive mode to iteratively discover parameters that produce results they desire and be able to save those values. Once again, spf13 comes to the rescue with [viper](https://github.com/spf13/viper). `viper` allows you to pull config values from quite a few different sources ranging from local files to environment variables to remote stores such as `etcd`.

The most complex requirement is a composition of the previous two; once a user has found a set of parameters that approximate what they’re looking for, they need to be able to interactively ( via command-line or other user interfaces yet to be designed and developed ) modify or override parameters to allow a fine-tuning of each phase of the generation process. Fortunately, the author of these libraries had the foresight to understand the need for these libraries.

## BindPFlags
This composition is then exposed via the `BindPFlags` [method](https://github.com/spf13/cobra#bind-flags-with-config). Given the correct arrangement of `cobra` flags, `viper` can now source 'config' values from the aforementioned sources _and_ command-line flags, with flags taking priority over all values except explicit `Set()` calls written directly into the Golang source code.

Thus, I had my solution. `viper` will read any configuration files that are present, and when prompted to present the value for a parameter (a pretend example would be something like `mountain-tallness`), it would check config files, environment variables, and then command-line flags providing the value given _last_ in the sequence of options.

Unfortunately, I was stymied by a number of different issues, not least of which was somewhat unspecified documentation in the README for `viper`. I opened a [Github issue](https://github.com/spf13/viper/issues/375) on this in August of 2017 and for a variety of personal reasons lost track of this issue and failed to check for updates. Fortunately, [Tyler Butters](https://github.com/tbutts) responded to it relatively quickly and even though I didn't come back to the issue until April of 2018, I responded to further questions on his [pull request](https://github.com/spf13/viper/pull/396) almost instantly.

I'm going to break down my misunderstandings and what might be considered shortcomings in the libraries and documentation before wrapping up with my solutions at the end of the post.

My first misunderstanding was not quite realizing that once `viper` has consumed flags from a given command, those values are then within the `viper` data store available to all commands and other components of the application. In short, `PersistentFlags` are not necessary once `viper` has been bound. This being true is a huge boon to the design of my parameters and commands; so long as my parameter names remain unique across the project, I can bind once in each command’s `init()` and never have to touch any `cobra` value APIs using it for nothing more than dealing with posix flags etc etc. The rest of the problems I had require a little more elaboration.

### Naming Confusion
The next issue, I would argue, is a design...oversight in `viper`. `viper`’s `BindPFlags` is troublingly named; in the context of `cobra`, `PFlags` can be misconstrued as `PersistentFlags` which are values that propagate downward from a given command to all its children. This could be useful for setting parameters such as an output directory, a desired file format for renders/output files and so on. `PersistentFlag` values would allow you to avoid repeating yourself when creating deeply nested command hierarchies.

What `BindPFlags` _actually_ means is "bind" to [PFlags](https://github.com/ogier/pflag), a juiced up, POSIX compliant replacement for the Golang standard library's `flag` toolkit. Realizing this took me quite a while. I can’t be _too_ upset though because `BindPFlags` accepts a [*pflag.Flagset](https://godoc.org/github.com/ogier/pflag#FlagSet), so it might be assumed that this would be obvious. Either way, it really disrupted my understanding of the process and left me believing that `BindPFlags` was able and willing to look for `PersistentFlag` values.

In [this commit](https://github.com/therealfakemoot/genesis/blob/da7e9c39e8e443df7d2de23ab1172ce5b3a100ff/cmd/root.go#L49-L63) you can see where I set up my flags; originally these were `PersistentFlags` because I wanted these values to propagate downwards through subcommands. Thanks to the use of `viper` as the application's source-of-truth, `PersistentFlags` aren't strictly necessary.

### Order of Operations
The last issue is more firmly in the realm of 'my own fault'; `cobra` offers a range of initialization and pre/post command hooks that allow you to perform setup/teardown of resources and configurations during the lifecycle of a command being executed.

My failing here is rather specific. `cobra` by default recommends using the `init()` function of each command file to perform your [flag setup](https://github.com/therealfakemoot/genesis/blob/da7e9c39e8e443df7d2de23ab1172ce5b3a100ff/cmd/root.go#L49-L63). On line 62, you can see my invocation of `BindPFlags`. The code I inserted to test whether `viper` was successfully pulling these values was also included in the same `init()` method. After some discussion with Tyler B, I had to re-read every single line of code and eventually realize that when `init()` is called `cobra` hasn't actually parsed any command line values!

In addition to the change from `PersistentFlag` to `Flag` values, I moved my debug code _inside_ of the `cobra` [command hooks](https://github.com/therealfakemoot/genesis/blob/da7e9c39e8e443df7d2de23ab1172ce5b3a100ff/cmd/root.go#L21-L25) and found that configuration file values were being read correctly (as they always had been) *and* when an identically named command-line flag was passed, `viper` presented the overriding value correctly.

# Summary
This series of misunderstanding and error in logic roadblocked my work on `genesis` for far longer than I'm proud to admit; efficient, effective, and sane configuration/parameterization is a key non-neogitable feature of this project. Any attempts to move forward with hacked-in or 'magic number' style parameters would be brittle and have to be dismantled (presumably painfully) at some point in the future. Thanks to Tyler, I was able to break through my improper grasp of the tools I was using and reach a point where I can approach implementing the more 'tangible' portions of the project such as generating terrain maps, accomplishing everything from rendering them to even starting to reason out something like a graphical interface.
