![build status](https://travis-ci.com/cfsamson/books-futures-explained.svg?branch=master)

# Futures Explained in 200 Lines of Rust

The rendered version is found at: [https://cfsamson.github.io/books-futures-explained/](https://cfsamson.github.io/books-futures-explained/)

You can find the main example in the repository [examples-futures](https://github.com/cfsamson/examples-futures).

This book aims to explain `Futures` in Rust using an example driven approach,
exploring why they're designed the way they are, and how they work. We'll also
take a look at some of the alternatives we have when dealing with concurrency
in programming.

Going into the level of detail I do in this book is not needed to use futures
or async/await in Rust. It's for the curious out there that want to know _how_
it all works.

## What this book covers

This book will try to explain everything you might wonder about up until the
topic of different types of executors and runtimes. We'll just implement a very
simple runtime in this book introducing some concepts but it's enough to get
started.

[Stjepan Glavina](https://github.com/stjepang) has made an excellent series of
articles about async runtimes and executors, and if the rumors are right there
is more to come from him in the near future.

The way you should go about it is to read this book first, then continue
reading the [articles from stejpang](https://web.archive.org/web/20200610130514/https://stjepang.github.io/) to learn more
about runtimes and how they work, especially:

1. [Build your own block_on()](https://web.archive.org/web/20200511234503/https://stjepang.github.io/2020/01/25/build-your-own-block-on.html)
2. [Build your own executor](https://web.archive.org/web/20200207092849/https://stjepang.github.io/2020/01/31/build-your-own-executor.html)

> Sorry for the archive links. The original site doesn't exist anymore.

## Contributing

All kinds of contributions are welcome. Spelling, wording or clarifications are
very welcome as well as adding or suggesting changes to the content. I'd appreciate
if you contribute through a PR.

The images in chapter 3 is created using Power Point. The power point itself is located in the
"resources" folder.

Feedback, questions or discussion is welcome in the issue tracker.

## Changelog

**2020-04-06:** Final draft finished

**2020-04-10:** Rather substantial rewrite of the `Reactor` to better the
readability and make it easier to reason about. In addition I fixed a mistake
in the `Poll` method and a possible race condition. See #2 for more details.

**2020-04-13:** Added a "bonus section" to the [Implementing Futures chapter](https://cfsamson.github.io/books-futures-explained/6_future_example.html) where we avoid using `thread::park` and instead show how we
can use a `Condvar` and a `Mutex` to create a proper `Parker`. Updated the [Finished Example](https://cfsamson.github.io/books-futures-explained/8_finished_example.html) to reflect these changes. Unfortunately, this led us
a few lines over my initial promise of keeping the example below 200 LOC but the I think the inclusion
is worth it.

**2020-12-21:** Rewrote the "Runtimes" paragraph of chapter 2 adding a useful model to understand
how runtimes work and included a lot of the suggested text from @ckaran in #25. Added a new chapter
"3 A mental model of how Futures work" which tries to visualize and give a good mental model to
build upon.

## License

This book is MIT licensed.

[rendered]: https://cfsamson.github.io/books-futures-explained/
