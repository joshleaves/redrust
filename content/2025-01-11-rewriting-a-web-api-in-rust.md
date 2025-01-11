---
title: "Rewriting a Web API in Rust"
date: "2025-01-11"
extra:
  author: "Arnaud 'red' Rouyer"
# taxonomies:
#   languages: ["rust"]
# generate_feeds: false
---

In another life, I used to work for a company in Japan.

For that company, I wrote both an iOS app, and a Rails 5 API to handle the data. That time was so long ago that, while the app still exists and works, it's stuck on the Heroku-18[^heroku-18] stack, and given how critical it is to many businesses, I'm not sure I can just upgrade the stack haphazardly.

Many times, I've thought about how I could really evolve a critical part of the app: the synchronization API. It handles no incoming data, and its sole work is:

1. Read the user cookie, deduce the user's access scope.
2. Some SQL to get the data.
3. Return the data.

Which sounds actually quite simple, none of these is an area where Rails would particularly shine with ActiveRecord models, validations,... And the part of the API that requires these Rails-specific elements is a part that I don't want to touch because it already works wonderfully good, and could accept a downtime for a day or two if I were to undergo a stack upgrade.

# Indiana Jones school of switching stuff

Given how good the client app works, and how big a can of worms was the writing of the data synchronization process, using a new exchange protocol is obviously out of the question: the first requirement of the new app will be to function EXACTLY as the previous one, so I  could just switch the `ROOT_URL` to point  to a new server and have client apps fetch data from this new one instead of the dusty Rails app.

# But before switching, have a cookie

This issue has been on my mind so long that I took the opportunity of a previous job mixing Rails and NodeJS server APIs to implement a [Rails cookie decrypting library for NodeJS](https://github.com/rails-cookies-everywhere/rails-cookies-nodejs). The research and testing I made on it helped in writing the [Rust equivalent](https://github.com/rails-cookies-everywhere/rails-cookies-rust)[^rails-cookies].

# Are we web yet?[^web-yet]

The Web-with-Rust ecosystem actually grows VERY fast. Unfortunately, it also dies fast. As I was looking around for comparisons on web frameworks, I found out that [a lot of them where "outdated"](https://github.com/flosse/rust-web-framework-comparison?tab=readme-ov-file#outdated-server-frameworks), with some that looked promising having been completely abandoned.

As the Tide blog already noted way back in 2018:

>Most language ecosystems have ended up with a key interface for talking about web services in general: Ruby has Rack, Python has WSGI, Java has Servlets, and so on.   
>― [Rising Tide: building a modular web framework in the open](https://rustasync.github.io/team/2018/09/11/tide.html)

While it later mentions that [hyper](https://github.com/hyperium/hyper) is used as a base by a lot of other frameworks, and while a lot of patterns are used by almost all of them, unfortunately, a lot of frameworks got "their own way" of doing stuff that's already been settled down in other languages, and as a beginner, it can feel daunting to start writing an app in a framework that may not exist again next year.

Another library that reappears a lot is [tower](https://github.com/tower-rs/tower), which aims to be the modular middleware/handler format, just like `(req, res, next)` has become for NodeJS.

The only ubiquitous brick of this system is [tokio](https://github.com/tokio-rs/tokio), which is the _de facto_ library for everything async in Rust, and [diesel](https://github.com/diesel-rs/diesel), which isn't a web component, but an ORM/query builder.

# Why not both?

I've selected the following frameworks:

- [Actix Web](https://actix.rs/): Not built on hyper, but feels like the most mature of the lot, given its HUGE number of commits and contributors.   
- [Axum](https://github.com/tokio-rs/axum): Uses hyper and tower, and the repository belongs to the tokio org, so I guess it's their ward, which helps secure its future.   
- [ntex](https://github.com/ntex-rs/ntex): Feels very mature too, with very powerful extractors.   
- [Poem](https://github.com/poem-web/poem): Built on hyper and tower, and feels a bit more opinionated than Axum. It was the first where I found an example of a middleware that would extract cookies and store them in the request the way I found most proper.   
- [Rocket](https://github.com/rwf2/Rocket): The first one I actually tried, and which almost made me gave up. While I LOVE their [Request Guards](https://rocket.rs/guide/v0.5/requests/#request-guards), their middleware equivalent ("[fairings](https://rocket.rs/guide/v0.5/fairings/#fairings)") lacks some feature I consider essential (like...terminating a request?).   
- [Salvo](https://github.com/salvo-rs/salvo): Salvo gets very close to Express: everything is a [Handler](https://salvo.rs/book/concepts/handler.html), with access to Request, Response, and a "Depot" of intermediate values (like...`@current_user`), an approach I appreciate a lot.

And I've elected to not touch a few others:
- [trillium](https://github.com/trillium-rs/trillium): Not updated in months.   
- [Viz](https://github.com/viz-rs/viz): While it started slow, Viz has been releasing a lot in 2024. Unfortunately, a lot of its parts still feels too "raw" for it to be called a framework.   
- [wtx](https://github.com/c410-f3r/wtx): Likewise, too "young".   
- [zino](https://github.com/zino-rs/zino): Not sure about what it actually brings to the table, since it can be built over actix, axium, or ntex...   

# Next(Handler)

I'm gonna process these in order, tackling two routes for now, with the following requirements:

1. Listen to requests and define two routes for two different resource types.   
2. Process the cookie data: decrypt it, inject a `@current_user` in the request, and return 401 otherwise.
3. Query the database, return the data.
4. Write tests making sure the data is *iso* with the original API.

Each framework will build its own server binary, and I'll then try to externalize as much parts as I can in order to properly identify the most common parts with each framework.

```
$ tree src
src
├── main-actix.rs
├── main-rocket.rs
├── using_actix
│   ├── middlewares
│   │   └── cookie_decoder.rs
│   └── routes
│       ├── companies.rs
│       └── sites.rs
└── using_rocket
    └── routes
        ├── companies.rs
        └── sites.rs
```
Onward with Actix!


[^heroku-18]: A server stack introduced around 2018, and deprecated in 2023.
[^rails-cookies]: Since I became the maintainer of two Rails Cookie decrypting libraries, I made a [GitHub organization for that](https://github.com/rails-cookies-everywhere).
[^web-yet]: [Yes, and it's _blazingly fast_](https://www.arewewebyet.org/)
