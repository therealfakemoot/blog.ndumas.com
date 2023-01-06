+++
draft = false
title = "Automating Caddy on my DigitalOcean Droplet"
date = "2023-01-05"
author = "Nick Dumas"
authorTwitter = "" #do not include @
cover = ""
tags = ["webdev", "devops"]
keywords = ["", ""]
description = "Automation ambitions fall flat"
showFullContent = false
+++

# Defining units of work
I've got a few different websites that I want to run: this blog, my portfolio, and my about page which acts as a hub for my other sites, my Bandcamp, and whatever else I end up wanting to show off.

To keep things maintainable and reproducible, I decided to stop attempting to create a monolithic all-in-one configuration file. This made it way harder to keep changes atomic; multiple iterations on my blog started impacting my prank websites, and it became harder and harder to return my sites to a working state.

# Proof of concept

The first test case was my blog because I knew the Hugo build was fine, I just needed to get Caddy serving again.

A static site is pretty straightforward with Caddy:

```
blog.ndumas.com {
    encode gzip
    fileserver
    root * /var/www/blog.ndumas.com
}
```

And telling Caddy to load it:

```bash
curl "http://localhost:2019/load" \
	-H "Content-Type: text/caddyfile" \
	--data-binary @blog.ndumas.com.caddy
```

This all works perfectly, Caddy's more than happy to load this but it does warn that the file hasn't be formatted with `caddy fmt`:

```
[{"file":"Caddyfile","line":2,"message":"input is not formatted with 'caddy fmt'"}]
```

# The loop

Here's where things went sideways. Now that I have two unit files, I'm ready to work on the tooling that will dynamically load my config. For now, I'm just chunking it all out in `bash`. I've got no particular fondness for `bash`, but it's always a bit of a matter of pride to see whether or not I *can*.

```bash
# load-caddyfile
#! /bin/bash

function loadConf() {
    curl localhost:2019/load \
        -X POST \
        -H "Content-Type: text/caddyfile" \
        --data-binary @"$1"
}

loadConf "$1"
```

```bash
# load-caddyfiles
#! /bin/bash

source load-caddyfile

# sudo caddy stop
# sudo caddy start
for f in "$1/*.caddy"; do
    echo -e "Loading $(basename $f)"
    loadConf "$f"
    echo
done
```

After implementing the loop my barelylegaltrout.biz site started throwing a 525 while blog.ndumas.com continued working perfectly. This was a real head scratcher, and I had to let the problem sit for a day before I came back to it.

After some boring troubleshooting legwork, I realized I misunderstood how the `/load` endpoint works. This endpoint completely replaces the current config with the provided payload. In order to do partial updates, I'd need to use the `PATCH` calls, and look who's back?

# can't escape JSON

The `PATCH` API *does* let you do partial updates, but it requires your payloads be JSON which does make sense. Because my current set of requirements explicitly excludes any JSON ( for now ), I'm going to have to ditch my dreams of modular code.



Not all my code has gone to waste, though. Now that I know `POST`ing to to `/load` overwrites the whole configuration, I don't need to worry about stopping/restarting the caddy process to get a clean slate. `load-caddyfile` will let me keep iterating as I make changes.

# Proxies

In addition to the static sites I'm running a few applications to make life a little easier. I'll showcase my Gitea/Gitlab and Asciinema configs. At the moment, my setup for these are really janky, I've got a `tmux` session on my droplet where I've manually invoked `docker-compse up`. I'll leave cleaning that up and making systemd units or something proper out of them for a future project.

Reverse proxying with Caddy is blessedly simple:
```
cast.ndumas.com {
    encode gzip
    reverse_proxy localhost:10083
}
```

```
code.ndumas.com {
    encode gzip
    reverse_proxy localhost:3069
}
```

With that, my gitea/gitlab is up and running along with my Asciinema instance is as well:

[![asciicast](https://cast.ndumas.com/a/28.svg)](https://cast.ndumas.com/a/28)

# Back to Square One
After finally making an honest attempt to learn how to work with Caddy 2 and its configurations and admin API, I want to take a swing at making a systemd unit file for Caddy to make this a proper setup.

# Finally

Here's what's currently up and running:
- [My blog](blog.ndumas.com)
- [Asciinema](blog.ndumas.com)
- [Gitea](blog.ndumas.com)

I've had loads of toy projects over the years ( stay tuned for butts.ndumas.com ) which may come back, but for now I'm hoping these are going to help me focus on creative, stimulating projects in the future.

The punchline is that I still haven't really automated Caddy; good thing you can count on `tmux`
[![asciicast](https://cast.ndumas.com/a/aWYCFj69CjOg94kgjbGg4n2Uk.svg)](https://cast.ndumas.com/a/aWYCFj69CjOg94kgjbGg4n2Uk)

The final code can be found [here](https://code.ndumas.com/ndumas/caddyfile). Nothing fancy, but troublesome enough that it's worth remembering the problems I had.
