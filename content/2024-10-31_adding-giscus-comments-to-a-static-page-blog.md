---
title: "Adding Giscus comments to a static page blog"
date: "2024-10-31"
extra:
  author: "Arnaud 'red' Rouyer"
# taxonomies:
#   languages: ["javascript"]
# generate_feeds: false
---

For my [personal blog](https://blog.dreamleaves.org/), which I started a few months ago already, I don't really need comments. It's more of a "keep-a-useful-calendar-of-medias-I-consume" list, I write it in French, and it's not meant to open discussion, so I wasn't looking at comments. At all.

However, for this tech-related blog, [posting the first article on Reddit](https://www.reddit.com/r/rust/comments/1gdd0md/trimming_down_a_rust_binary_in_half/) gave way to very interesting discussions and commentaries. So I figured: "Heh, why not?".

# The state of Internet comments in 2024

My very first personal blog, back in 2004, was written in PHP and hosted on a Free server[^free]. The (anonymous) comments were saved in own my database, and when I went back to check on it twenty years later, the place was littered with spam, and what I'm pretty sure were various attempts at HTML injections.

Clearly, nobody wants that. And nobody wants to spend time away from writing in order to play warden and build an user-authentification service just for a few comments[^fewcomments]. Plus, this blog's HTML is only rendered once I deploy it, so there' no way I can add comments anywhere, I guess.

And then there is [Disqus](https://en.wikipedia.org/wiki/Disqus).

Disqus looks nice. It's a testament to its beauty in design that every solution tries to imitate Disqus today. I remember over time seeing a LOT of competitors[^moot], but only Disqus survived the test of time, and grew old enough to drink some alcohol in France.

However, Disqus is also a proprietary solution, I'm not entirely sure of where they stand on [GDPR](https://en.wikipedia.org/wiki/General_Data_Protection_Regulation), I've seen enough Disqus conversations get derailed by trolls,... This is not the right choice for a tech blog.

# Enter Giscus

On the paper, [Giscus](https://giscus.app/) fills what I'm missing:

- Open-source.
- Commenting requires a GitHub account.
- Comments are hosted on GitHub.

Therefore, only developers will comment. Maybe it's gonna end up being a fate worse than trolls.

## Setting up Giscus

Setting up Giscus is actually a breeze and their homepage is a very nice setup wizard. My only issues were:

- Which repository is best suited for Giscus?

Giscus comment sections are tied to a repository, which left me puzzled, as I wasn't sure WHICH repository I should tie them to: [this blog's repository](https://github.com/joshleaves/redrust/)? Or another and completely unrelated one?

The answer is...it's up to you, but it must be public. So if your blog is in a private repository, you'll need a specific repo for the comments. I chose to have everything together to have the full "static-page-blog-on-github" experience, but it's nice to have a choice.

- What are those pathname/url/<title> options?

Just pick one and forget about it. Having this option feels like too much hassle for what it is, and if you ever change an article's title/URL, you'll be screwed anyway. I'd have like an option to map the conversation to something like `<meta property="giscus:uuid" value={{ uuid }}>`, and maybe this will be available in the future, BUT this would break the nice display of the discussions on Github.

For my part, I started with `pathname` but just having an URL as the title of the GitHub Discussion was irking me, so I went the `og:title` route and customised it a little to add the date.

- Which theme should I use on my blog?

I'm used to Dark Mode on Github, but this blog is very LIGHT. Starting with a dark theme was a wrong (and very ugly) move, and trying one of the "no-border" themes rendered the comment section almost invisible.

The preview on the setup wizard isn't very effective in guessing how it's gonna fit on your blog, so apart from manually trying each theme, just decide on `light_protanopia` or `dark_protanopia` and roll on with it.

## Adding Giscus

You'll get a nice `<script>` tag to copy-paste into your template. The whole process is [so simple, it's almost ridiculous](https://github.com/joshleaves/redrust/commit/9f7e5257d1887207baaa70d71334851811ddb4dc).

# And then?

An input will be available when your visitors reach the end of your posts, and they'll be able to post comments.

Whenever a comment is posted:

- you'll get notification in GitHub (nice!).
- you'll be able to follow (and moderate?) the discussion in your repository's [Discussions](https://github.com/joshleaves/redrust/discussions) tab.

Happy commenting.

* * *
[^free]: "Free" as in both ["free of charge"](https://en.wiktionary.org/wiki/free_of_charge), and ["Free the French ISP"](https://en.wikipedia.org/wiki/Free_(ISP)).
[^fewcomments]: But I'm keeping hope I will become famous in the future and get tons of comments!
[^moot]: I remember being quite fond of [moot](https://news.ycombinator.com/item?id=6818416) back then, [just for its name](https://en.wikipedia.org/wiki/Christopher_Poole), but the product is dead as nails today.
