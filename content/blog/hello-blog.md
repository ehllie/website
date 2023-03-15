+++
title = "Hello Zola"
date = 2023-03-15

[taxonomies]
categories = ["Website"]
tags = ["website", "zola"]

[extra]
lang = "en"
toc = false
+++

# Welcome again!

This website initially debuted almost half a year ago, in a much barer state and nominally using [SvelteKit](https://kit.svelte.dev).
I'm now switching to [Zola](https://www.getzola.org) instead.

<!-- more -->

In hindsight, it should be quite obvious that a static site generator would a better fit for a personal blog than an SSR meta framework.
I did however hold onto the hope that it might convince me to learn how to do web frontends, so it took me some time before I admitted it was not going to happen.

Zola has felt very convenient so far. For my website I went with the [Serene](https://github.com/isunjn/serene) theme.
Configuration was done using a `config.toml` file in the root of the project, and then individual posts are made using markdown with toml metadata.
It's definitely something I can see myself using going forward.
Supposedly it comes with an automatic RSS feed generator as well, but I've honestly never used those.
However, to whom it may concern, it should be possible to subscribe to this blog from the blog tab.

That should be about it for this post.
However, I do have a couple thing on the agenda for what I want to write about next, so it hopefully will take me less than half a year to do that.
Stay tuned for a post about a toy language I've been designing, `unit`.
