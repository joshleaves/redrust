---
title: "Trimming down a rust binary in half"
date: "2024-10-27"
extra:
  author: "Arnaud 'red' Rouyer"
# taxonomies:
#   languages: ["rust"]
generate_feeds: false
---

Lately, I've stumbled on a [blog post about Rust binary sizes](https://kobzol.github.io/rust/cargo/2024/01/23/making-rust-binaries-smaller-by-default.html). I haven't done much compilation[^compilation] since I last touched C in school, but I was intrigued by the subject and decided to look at how binary size reduction could impact my own Rust project: [advent-rs](https://github.com/joshleaves/advent-rs), a simple binary taking year/day/part/filename parameters and solving exercises from the [AdventOfCode](https://adventofcode.com/) online contest.

## Starting point
There's already some stuff I know and set up properly, so I already got the correct defaults for my `Cargo.toml`.
```rust
[profile.release]
opt-level = 3
```
Let's look at the resulting file through `size`[^size] and `ls`:
```bash
$ size target/release/advent-rs                                                                                                                                                                                 __TEXT	__DATA	__OBJC	others	dec	hex
1409024	16384	0	4295393280	4296818688	1001c4000

$ ls -lah  target/release/advent-rs
 -rwxr-xr-x@ 1 red  staff   1.8M Oct 27 08:49 target/release/advent-rs
```
So we are standing at 1.8M, for a binary that's solving around 150 exercices. That doesn't sound so bad, but again, I haven't compiled code in a long time, so I have no idea where I stand. Let's check the first binary I can think of.

```bash
$ ls -lah /bin/ls
-rwxr-xr-x  1 root  wheel   151K Sep  5 11:17 /bin/ls
```
Well, I definitely need to do some trimming.

## Optimizing compilation
First things first, let's check compilation.

### Start with stripping
As I remember from my C days, and as the blog clearly states: not stripping symbols from a release is the root of all evil.

```rust
[profile.release]
opt-level =  3
strip = true
```
Build, check file.
```bash
$ /bin/ls -lah  target/release/advent-rs
-rwxr-xr-x@ 1 red  staff   1.5M Oct 27 09:03 target/release/advent-rs
```
Already 300kb trimmed off!

### Halt and catch fire
A **backtrace** is a very useful to have when debugging, all interpreted languages come with them, and for a while, I wasn't even surprised to see them pop up in Rust, in spite of C never printing one, only going as far as telling me "Segmentation fault" when my carefully-crafted binary would have the politeness to tell me on its own that I was doing something wrong.

Well, Rust should not have them.

Wait, let me explain myself here: backtraces are not a normal feature of a compiled language, since the backtraces you are used to actually come courtesy of the interpreter. So in a compiled language with no interpreter, where do they come from?

Yes, from your binary.

```rust
[profile.release]
opt-level =  3
strip = true
panic = "abort"
```
Build, check file.
```bash
$ /bin/ls -lah  target/release/advent-rs
-rwxr-xr-x@ 1 red  staff   1.2M Oct 27 09:05 target/release/advent-rs
```
Can you believe this shaved off another 300kb?

### Insane stuff I don't want
The [Cargo documentation](https://doc.rust-lang.org/cargo/reference/profiles.html) gives us more options we can experiment with for fun. They don't work for me but your mileage may  vary.
#### Optimize for size
After `3`, there are `opt-level` values that target smaller binaries. I can try with `s`or `z` and they will respectively shave off 100 and 200kb, BUT the tradeoff here is execution speed, which is not a value I want to allow myself to play with[^speed].

#### Don't check integer overflow
You can disable checking for integer overflow with `overflow-checks = false` from your binary. Obviously, this is NOT recommended when you're playing with user input, and in my case, it won't even register any change.

#### Link-time optimization
It seems that other than compiling, you can also optimise linking[^linking] with `lto = true`. I don't recommend it since it doubled my build time AND didn't give me a good size reduction...


# Cleaning up crates
The other thing that will take up space is...code. More specifically, code you wouldn't need. Let's check my dependencies:

```rust
[dependencies]
clap = { version = "4.5.1", features = ["derive"] }
itertools = "0.12.1"
md-5 = "0.10.6"
mutants = "0.0.3"

[dev-dependencies]
assert_cmd = "2.0.13"
predicates = "3.1.0"
criterion = { version = "0.5.1", features = ["html_reports"] }
```
Nothing too barbaric here, [clap](https://github.com/clap-rs/clap) is the *de-facto* crate to parse command-line arguments, [itertools](https://docs.rs/itertools/latest/itertools/) is required because a ton of exercises require iterating over data in strange ways, I require [md-5](https://docs.rs/md-5/latest/md5/) for four exercices, and [mutants](https://github.com/sourcefrog/cargo-mutants) is allowed here only to [skip some tests when running mutations](https://mutants.rs/attrs.html).

As for the development dependencies, I know I got nothing to fear from them.

# Removing a crate
You may feel like the above dependencies are **sane**, that I need all of them, that there is no way my project could function without any of them,... But what if it could?

## Past experience
When I started writing all of these parsers I required for handling input data, I often used the [regex crate](https://docs.rs/regex/latest/regex/). While it was very useful, I knew from my benchmarks that using old-school "split-on-this-char" approach was faster[^speed], so I slowly started phasing it out.

As of [Year 2017: Day 15](https://github.com/joshleaves/advent-rs/commit/50cb0325dcaa039049106057a2c64e635b3e8955#diff-2e9d962a08321605940b5a657135052fbcef87b5e360662bb527c96d9a615542L35), I removed the crate from my `Cargo.toml`.

## Future ideas
While I know that "rolling out your own crypto" is a bad idea, the crate `md-5`can clearly be replaced. Plus the fact that its input and outputs are of different types mean one exercise could actually be way faster[^fastermd5] if I rolled out my own!

# Okay, now where's the bloat?
I found out the appropriately-named tool [cargo-bloat](https://github.com/RazrFalcon/cargo-bloat) on GitHub.

Let's install and try it, it's just a command[^cargo-bloat] after all.

```
$ cargo bloat --release
    Finished `release` profile [optimized] target(s) in 0.01s
    Analyzing target/release/advent-rs

 File  .text     Size        Crate Name
 1.1%   1.6%  16.0KiB clap_builder clap_builder::parser::parser::Parser::get_matches_with
 1.1%   1.6%  15.8KiB    [Unknown] __mh_execute_header
 0.7%   1.1%  10.5KiB clap_builder clap_builder::builder::command::Command::_build_self
 0.6%   0.9%   8.9KiB          std std::backtrace_rs::symbolize::gimli::resolve
 0.6%   0.9%   8.7KiB          std std::backtrace_rs::symbolize::gimli::Context::new
 0.5%   0.7%   7.1KiB          std gimli::read::dwarf::Unit<R>::new
 0.4%   0.7%   6.5KiB clap_builder <clap_builder::error::format::RichFormatter as clap_builder::error::format::ErrorFormatter>::format_error
 0.4%   0.6%   6.1KiB clap_builder clap_builder::parser::validator::Validator::validate
 0.4%   0.6%   5.8KiB    advent_rs core::slice::sort::stable::quicksort::quicksort
 0.4%   0.6%   5.6KiB          std addr2line::ResUnit<R>::find_function_or_location::{{closure}}
 0.4%   0.5%   5.2KiB          std addr2line::Lines::parse
 0.4%   0.5%   5.2KiB clap_builder <alloc::vec::Vec<T,A> as core::clone::Clone>::clone
 0.3%   0.5%   5.0KiB clap_builder clap_builder::output::help_template::HelpTemplate::write_all_args
 0.3%   0.5%   5.0KiB clap_builder clap_builder::output::usage::Usage::write_arg_usage
 0.3%   0.5%   4.9KiB    advent_rs advent_rs::year_2016::day_10::solve
 0.3%   0.4%   4.3KiB clap_builder clap_builder::parser::parser::Parser::react
 0.3%   0.4%   4.2KiB clap_builder clap_builder::output::usage::Usage::write_usage_no_title
 0.3%   0.4%   4.2KiB    advent_rs advent_rs::year_2017::day_21::Image::mutate
 0.3%   0.4%   4.1KiB          std addr2line::function::Function<R>::parse_children
 0.3%   0.4%   3.9KiB clap_builder clap_builder::output::help_template::HelpTemplate::write_templated_help
58.8%  87.9% 872.3KiB              And 1925 smaller methods. Use -n N to show more.
66.9% 100.0% 992.5KiB              .text section size, the file size is 1.4MiB
```

<div class="content-img" style="width: 50%">
  <img src="/img/2024-10-27/fortune-teller.jpeg" alt="Never going to that fortune-teller ever again">
  <p class="img-alt">
    "Never going to that fortune-teller ever again"
  </p>
</div>


Something must be wrong, let's try another command.

```
$ cargo bloat --release --crates
    Finished `release` profile [optimized] target(s) in 0.01s
    Analyzing target/release/advent-rs

 File  .text     Size Crate
25.9%  38.8% 384.8KiB std
22.3%  33.4% 331.1KiB advent_rs
15.8%  23.5% 233.6KiB clap_builder
 1.2%   1.7%  17.1KiB itertools
 1.1%   1.6%  15.9KiB [Unknown]
 0.7%   1.0%  10.0KiB md5
 0.3%   0.5%   5.0KiB digest
 0.3%   0.5%   4.6KiB strsim
 0.2%   0.3%   3.3KiB clap_lex
 0.1%   0.2%   2.1KiB anstyle
 0.1%   0.2%   1.8KiB anstream
 0.0%   0.0%      16B colorchoice
66.9% 100.0% 992.5KiB .text section size, the file size is 1.4MiB
```

First off, I'm surprised that md5 is only taking around 10Kb, so maybe I won't need to replace it until I really want to shave a few milliseconds off that one exercise[^fastermd5]. But for the rest...

# What the fuck Clap?
It seems I'm [not the first person to complain about clap's binary size](https://github.com/clap-rs/clap/issues/1365). I'm even surprised it can even get to this size because its only use is to parse an array of strings.

I mean, seriously:
```rust
#[derive(Parser, Debug)]
#[command(version, about, long_about = None)]
struct Args {
  /// The year of the exercise, from 2015 to today
  #[arg(short, long, value_parser = clap::value_parser!(u16))]
  year:  u16,
  
  /// The day of the exercise, from 1 to 25
  #[arg(short, long, value_parser = clap::value_parser!(u8))]
  day:  u8,

  /// The part of the exercise, 1 or 2
  #[arg(short, long, default_value_t = 1, value_parser = clap::value_parser!(u8))]
  part:  u8,

  /// File name
  #[arg(help =  "Input file path (will read from STDIN if empty)", value_parser = clap::value_parser!(PathBuf))]
  input: Option<PathBuf>,
}
fn main() {
  let  args  = Args::parse();
}
```
And I'm not even using ALL the features!

# Picking a sane alternative to Clap
Am I in the mood to look up for a new crate, and completely rewrite the input part of my code? Not really, but I've got a blog to write!

Let's do a quick Google, and thankfully, among the first results is a [recap of various arg-parsing libraries](https://github.com/rosetta-rs/argparse-rosetta-rs):

* **null**
Obviously, I'm not in the mood to parse them by name.
* **[argh](https://github.com/google/argh)**
Looks very similar to Clap.
* **[bpaf](https://github.com/pacak/bpaf)** 
If I wanted a DSL, I'd be using Ruby.
* **[gumdrop](https://docs.rs/gumdrop/latest/gumdrop/)**
That looks like A LOT of code.
* **[lexop](https://github.com/blyxxyz/lexopt)**
I kinda like this one's approach!
* **[pico-args](https://github.com/RazrFalcon/pico-args)**
No examples, that's too simple for my taste.
* **[xflags](https://github.com/matklad/xflags)**
Looks a bit too complicated.

I can easily start with **Argh** then, it looks similar enough that I could maybe just drop it in and have it work.

# The beauty of Rust unit tests
As a developper, I've written a lot of untested code.

With [advent-rb](https://github.com/joshleaves/advent-rb), I used unit tests for each exercice, using the examples as inputs and result values in [RSpec tests](https://github.com/rspec), which was very fun.

With [advent-rs](https://github.com/joshleaves/advent-rs), and Rust in general, I discovered a new paradigm. While all the languages I ever used made it "easy" to forget to add another file with the correct name, asked me to remember a complicated syntax, made me think of ways to cheat on my code to access private members from another file,... Rust just made away with that: testing a function is done from the same file where it's defined, and I don't even have to care about visibility.

<div class="content-img" style="width: 75%;">
  <img src="/img/2024-10-27/allthere-720x405.jpg" alt="It's all there!">
  <p class="img-alt">
    "It's all there!"
  </p>
</div>

The only thing not handled out-of-the-box is testing a binary (well, that's not really a unit test), but since everything else was so easy, it felt okay to take some time to write a test specifically for that in `tests/cli.rs`.

In that case, rewriting is very easy:
* remove `clap` from dependencies.
* add `argh` to dependencies.
* rewrite my `#[derive()]`
* some syntax differences

Quite frankly, the [whole diff](https://github.com/joshleaves/advent-rs/commit/e821b287eb567627977087b49663717023d224ce) is nothing short of hilarious given how simple it is.

But the best part?

I can just run `cargo test` and my tests will tell me everything works the same!

# Cleaning up
Let's check up on that bloat one more time, shall we?
```
$ cargo bloat --release --crates
    Finished `release` profile [optimized] target(s) in 0.01s
    Analyzing target/release/advent-rs

 File  .text     Size Crate
32.8%  49.3% 360.3KiB std
30.3%  45.5% 332.6KiB advent_rs
 1.6%   2.4%  17.2KiB itertools
 1.0%   1.5%  11.2KiB [Unknown]
 0.9%   1.4%  10.0KiB md5
 0.6%   0.8%   6.2KiB argh
 0.5%   0.7%   5.0KiB digest
66.5% 100.0% 730.3KiB .text section size, the file size is 1.1MiB
```
As for the file size:
```
$ ls -lah  target/release/advent-rs
-rwxr-xr-x@ 1 red  staff   898K Oct 27 11:02 target/release/advent-rs
```
Woah, not bad, that shaved off 300kb more, and we reduced our total binary size by almost 1MB.

# Going just a bit further
Once again, I am standing on the shoulders of giants, and while I am quite happy with the trimming I did, I found way more experimentations I could perform on [johnthagen's min-sized-rust repository](https://github.com/johnthagen/min-sized-rust).

However, those solutions either require unstable features (we don't do `nightly` in this house), or stripping off Rust's standard library, which is a whole other can of worms...


* * *
[^compilation]: Blame NodeJS and Ruby!
[^size]: As its [man page](https://linux.die.net/man/1/size) will tell you, `size` prints the "sections" in a binary, helping you visualise the separation between code and data for instance. It's not very needed here, but it's a useful command to know about.
[^speed]: This project was also built to compete with [advent-rb](https://github.com/joshleaves/advent-rb), the same project written in Ruby, to marvel at the speed of execution I was missing from compiled languages.
[^linking]: Linking is the second part of building a project: in layman's terms, it will bind all code parts together into a single executable.
[^fastermd5]: Part two of [Year 2016: Day 14](https://adventofcode.com/2016/day/14) requires for a string to be hashed on itself 2016 times. Since the input must be a string, and the output is a buffer of hexadecimal data, my code needs to re-translate that hex buffer to a string 2016 times...
[^cargo-bloat]: The `cargo-bloat` command itself will actually bloat your code. I haven't checked out how, but I'm 99% sure it's keeping in some symbols in so it can return the name of the fonction taking space. In my case, it's adding around 200kb.
