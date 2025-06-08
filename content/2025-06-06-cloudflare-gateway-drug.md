---
title: "Cloudflare's Gateway Drug"
description: "Or how I learned to stop worrying and embrace the Edge."
date: 2025-06-06
extra:
  author: "Arnaud 'red' Rouyer"
  thumbnail: "img/2025-06-06/free-lunch.png"
generate_feeds: true
---

When you're not yet huge enough to run your own datacenter, or rent a 2U server colocation, hosting web applications can be a pain, and as DHH eloquently wrote it: [the merchants of complexity](https://world.hey.com/dhh/merchants-of-complexity-4851301b) made sure all developers found themselves faced with overpriced technologies they wouldn't actually need.

But it turns out there’s a much simpler (and often free!) path to getting projects online, if you know where to look.

# The myth of the million visitors

Three years ago, I was writing the API for a chat app where the higher-ups suddenly decided we required a Kubernetes cluster to handle the influx of users that would be coming our way on launch day. It took three months and a team of two freelancers. In all fairness, those freelancers built exactly what was asked of them: our very own K8S cluster, with production/staging support, CD on git deploy, and even auto-rent of short-burst Amazon EC2 instances to get them for a lower price. Seriously, what could go wrong?

The infrastructure was bulletproof. No traffic ever bothered to shoot at us.

# Before

As mainly a web developer, each project always comes with a pain: where am I gonna host it? I'm not a viral developer most of the time, my projects could actually be run and served off my own laptop[^badnetwork]. What then?

- **Spin up a t3.nano on AWS?** No, I don't want to set up a server every time I want to put something online?
- **One giant server for everything[^oneserver]?** That's too much work, too expensive upfront, and often, too many app environments to handle together. And on **AWS**, do I really want to live in fear of the EGRESS FEES!?
- **So-called "serverless" [^serverless] platforms like [Heroku](https://www.heroku.com/) or [Fly](https://fly.io/)?** They are actually nice, but some of them (looking at you [Vercel](https://vercel.com/)!) have a tendency to produce bills in the six-figure ranges[^horrorstories].
- **[Github Pages](https://pages.github.com/)?** It's free, comes with [GitHub actions](https://github.com/features/actions) for deployments, and...there's messing with `gh-pages`.

So **Heroku** ended up as my default: cheap, easy, and if my project ever uses too much resources, they'll gladly put it down. Kind of reassuring, actually.

But then I stumbled onto **Cloudflare**, and everything changed.

# Hosting my own blog

In May of 2024, I got back into the habit of reading books[^readingbooks], and a bad one got me to write about them[^verybadbook]. So I went for a static blog with [Jekyll](https://jekyllrb.com/)[^whyjekyll], and my first instinct was to look at **Github Pages**, but again, `gh-pages`: I didn't want to deal with "artifacts" or any of those strange branch-switching shenanigans.

At that point, I still thought **Cloudflare** was only about those annoying "Are you human?" pages on direct-download sites, CDN, or protecting sites from DDoS and all, which are not problems I've ever come into contact with. So I had no idea how I stumbled onto **Cloudflare Pages**. Surely the pricing for that kind of around-the-world service would be expensive, right?

> On both free and paid plans, requests to static assets are free and unlimited. A request is considered static when it does not invoke Functions.   
> ― [Cloudflare Pages Pricing](https://developers.cloudflare.com/pages/functions/pricing/)

In **Ruby on Rails** terms, "static assets" means stuff like the CSS and the JavaScript that the server would create once at build time, and serve as-is to visitors. But if I'm using **Jekyll**, then that would mean the HTML files generated are ALSO static assets, right? Is it saying **Cloudflare** would host my blog FOR FREE!?

Yes. Yes, it is.

# Hosting sites for other people (and feeling good about it)

Chances are, if you're reading this, you're the most tech-aware person of your family or friend group, and you're often asked: "Can you make me a website for my (small company/association/shop idea)?". It's okay, we've all been there, and I'm pretty sure we've all gone through the same thought process:

> Sure, I can make a website. Do they have a domain name or should I set it up? Do they have hosting? Should I set it up all by myself? I guess they won't need something as powerful as a full Rails engine, or a machine with 4TB of RAM. Should I go for something static then? But where? How will I pay? How do I make sure the domain/website is under their name even if I'm doing all the legwork? And what if they want to update some text, should I add an admin panel, but then it would require auth,... Do I need to set up anything LetsEncrypt again?   
> ― You, me, and everybody else, before going to [Wordpress.com](https://wordpress.com/).

So when my aunt came to me for help, I was relieved: her association already had the domain and hosting. You know the kind: *shared* hosting, with its own FTP server. As I'm writing this, I'm trying to remember: did the admin panel indicate they were running PHP 4 or 5? Because I know it was neither 7 nor 8[^php6] and it's making me shiver.

Thankfully, she just wanted me to rework an existing landing page, and it was static: no strange PHP scripts, no MySQL injections[^injection] to sanitize,... Since static page means static assets, I put everything on **GitHub**, connected the repository to Cloudflare, changed the DNS records in their domain's admin panel, and *voila!*, now if I (or a future developer) wanted to update the contents of their website, a simple `git push` would suffice, and the site would automatically deploy, and with a working SSL certificate!

The next person who needed my help for a "website" was a friend who wanted to host an image online, so a QR code would lead to it, as part of a birthday treasure hunt. I could have sent her to [Imgur](https://imgur.com/), or said no. But now, what was once the daunting task of hosting was so painless that I set it up while half-watching a TV show: I deployed the image under a subdomain of my main site, and it just worked.

That's when I realized how happy I was. Hosting had become simple: not a bill to stress over, or a sysadmin side quest. Now, I could just put a file online, instantly, for anyone, with zero drama about pricing, or the egress fees of an **AWS S3** bucket. And I had **Cloudflare** to thank for it.

# Code everywhere

A few months later, one of my brothers was the one to ask for my help. With crypto. Oh boy.

There was no trading involved, no *degens* to cater to,... But my brother had found a few wallets doing extraordinary calls, and he didn't trust copy bot services. So while he could manually check the wallets online, having code to automate the process would make his "work"[^tradingnotjob] easier.

I hadn't touched crypto in a while, and my latest crypto project[^valentinecoin] was a lot of fun to code for. So I went along with it, looked up the documentation, wrote a JS script, and...now what? My brother did have a computer, an old Windows laptop from who-knows-when, and I wasn't sure he could even run the script. Would you believe who came to the rescue again?

For another project, I had been looking at whether **Cloudflare** supported CRON triggers. Well, if it could run mail jobs at 2am, surely it could also run my bot's "check-balances-and-send-message" loops. But for what price, would that be free too?

[Yes. Yes, it is.](https://developers.cloudflare.com/workers/platform/pricing/)

The craziness of this pricing can't be overstated: for my brother's crypto fun, I had a CRON job running and sending him regular updates. I was even using a **D1 Database** to store the balances (which was also [free for that level of usage](https://developers.cloudflare.com/d1/platform/pricing/)), and best of all, I didn't have to worry about any kind of intrusion, which I would have with a regular server.

I don't know how much my brother made, as I don't really care about trading, but a few months later, he bought himself a MacBook Air, and later offered to buy me a PS5 Pro when I mentioned I was waiting for [Death Stranding 2](https://www.youtube.com/watch?v=wbLstJHlC4U) coming out in June.

Which is very nice from him, since, so far in this story, I still hadn't paid a cent to **Cloudflare**.

# Finally paying for the product

I finally paid for Cloudflare.

I still had a professional project running on **Heroku**, both **Rails** server and **PostgreSQL** database, and I wanted to move some parts of the server API to faster NodeJS servers. Unfortunately, the provided database was VERY conservative with the number of allowed client connections. So [Hyperdrive](https://blog.cloudflare.com/hyperdrive-making-regional-databases-feel-distributed/) felt like an awesome solution, both from a technological standpoint and as an answer to my needs. There was no free tier[^hyperdrivefreetier], but just subscribing to the "Workers Plan" enabled unlimited usage, plus it would even increase the (very generous) limits of all other products.

After a lot of tomfoolery regarding client connection limits and worldwide latency[^comingsoon], I ended up with a NodeJS API available worldwide, connecting to my database through **Hyperdrive**, with a performance measured in the hundreds of milliseconds, all in one night's work, and a commitment to a monthly payment of only **5 DOLLARS** per month. Take that, Kubernetes!

Honestly, it didn't even feel strange to finally "give in" and punch in my credit card digits: after so much free utility, it was a no-brainer. For the price of a fancy coffee[^starbucks], I got performance and global reach my old infra couldn’t touch!

# The future

Hey, if someone at **Cloudflare** is reading this, do you have any opening for a developer evangelist in France?

When I look at the **Node** APIs I had started writing to decouple some parts of my **Rails** monolith, they are hosted on either **Heroku**, or **Fly**, and these products are great, but getting one too many "{application_name} ran out of memory and crashed" emails from them was the nail in the coffin. Sure, I'm not leaving **Heroku** anytime soon, because [ActiveAdmin](https://activeadmin.info/) is still a wonderful backoffice solution and requires **Rails**, but for anything that only needs **JavaScript**, I can't imagine going anywhere other than the **Workers**.

Which is even funnier when I consider that I've barely touched all the available services: I set up [Email routing](https://developers.cloudflare.com/email-routing/) because I don't need a full-priced tool just to redirect domain emails to my Gmail, I'm still scratching the surface of [Workflows](https://developers.cloudflare.com/workflows/) (integrating that into my CRONs), I don't need [R2](https://developers.cloudflare.com/r2/) yet, and I haven't tried [Queues](https://developers.cloudflare.com/queues/) but they look interesting. And there are [a bunch more I haven't even looked at yet](https://developers.cloudflare.com/products/?product-group=Developer+platform)!

But hey, if you'd rather spend time on Twitter circlejerking about the size of the Kubernetes cluster serving five users, or complaining about the six-figure bill that crept up on you[^sixfigures], you do you.

Me? I'm building on Cloudflare.

* * *
[^badnetwork]: I could, if I had something better than 2MB/s on my home line.
[^oneserver]: Most notably, "indie" maker [@levelsio](https://x.com/levelsio) does that.
[^serverless]: Kudos to whoever coined this term that's so far away from reality!
[^horrorstories]: If you don't believe me, check out [Serverless Horrors](https://serverlesshorrors.com/)!
[^readingbooks]: The book to kickstart it was [Jake Adelstein](https://x.com/jakeadelstein)'s [Tokyo Vice](https://blog.dreamleaves.org/posts/tokyo-vice-book/).
[^verybadbook]: This book shall live in infamy: [Aux Origines de Castlevania Symphony of the Night](https://blog.dreamleaves.org/posts/aux-origines-de-castlevania-sotn/).
[^whyjekyll]: Why choose Jekyll? I'm a ruby boy at heart, and I liked the theme I had seen.
[^php6]: Can you believe [there never was a PHP 6 major version](https://ma.ttias.be/php6-missing-version-number/)?
[^injection]: A staple of all developers who ever touched PHP is learning about [SQL injections](https://en.wikipedia.org/wiki/SQL_injection), sometimes the hard way.
[^tradingnotjob]: I don't believe day-trading to be a real job, more like an addiction.
[^valentinecoin]: For Valentine's Day 2018, I launched [The Valentine Coin](https://www.thevalentinecoin.com/) with a friend.
[^hyperdrivefreetier]: Quite funnily, **Hyperdrive** now has a free tier, but it's too low for my requirements.
[^comingsoon]: Which will be covered in a further post.
[^starbucks]: And to think that ten years ago, I was buying myself one of those "fancy coffees" every day, if not two per day...
[^sixfigures]: Don't forget to tweet "VICTORY!" when the company discounts it to four in "a commercial gesture".
