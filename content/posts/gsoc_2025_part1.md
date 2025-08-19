---
title: "Google Summer of Code 2025: Part One"
tags: ["google summer of code", "gsoc"]
---

I was selected as a contributor for GSoC 2025 under the project "Improving Core Clang-Doc Functionality" for LLVM.
My mentors for the project are Paul Kirth and Petr Hosek.

They let me know there was *a lot* to improve, and they were right!

## Introducing Clang-Doc

Clang-Doc is a tool in clang-tools-extra that generates documentation from Clang's AST.
Prior to this summer, it emitted documentation in Markdown, HTML, and YAML.
The project started in 2018 but major development eventually slowed.
Recently, there have been efforts to get it back on track.

Here's a quick overview on Clang-Doc's architecture, which follows a map-reduce pattern:

1. Visit source declarations via Clang's ASTVisitor.
2. Serialize relevant source information into an Info (Clang-Doc's main data entity).
3. Once all source declarations are serialized, write them into bitcode, reduce, and read the reduced Infos.
4. Serialize Infos into the desired format.

## GSoC

The [GSoC project idea](https://discourse.llvm.org/t/improve-documentation-parsing-in-clang/84513) had a simple premise: improve core functionality.
For the user-facing side, that meant documentation quality.
From the contributor's side, that meant making high quality documentation easy to emit.
Many of the obstacles impeding high-quality documentation actually came from challenges of the latter.

### The Issues

Clang-Doc emits documentation in a couple formats, but it isn't the highest quality.
The project idea proposed three main areas of focus to improve documentation quality.

1. C++ support
2. Doxygen comments
3. Markdown support

First, not all C++ constructs were supported, like friends or concepts.
Not supporting core C++ constructs in C++ documentation isn't great.

Second, the actual HTML documentation produced wasn't great, either.
It didn't look the prettiest and it wasn't organized the best, but getting it to look pretty is actually secondary, currently.
Making sure all the right data is available to display is the much more difficult problem, and it didn't help that our HTML generator wasn't easy to extend.
Finally, having Markdown available to developers for documentation would be so useful.
Markdown provides expression in an area that can be devoid of it.

Clang-Doc's architecture wasn't the easiest to extend.
If a new C++ construct needed to be supported, it would be visited and serialized, but then support would have to be added to each backend individually.
Thus, if you wanted to output functions in YAML, you'd have to go out of your way to implement the Markdown logic separately.
This placed a very high maintenance cost for extending basic functionality, even if you just wanted to add a missing specifier, like `explicit`.

This led to another problem: testing.
Testing was in an awkward spot because there wasn't a clear source of truth, i.e. which format generator should be used to verify the quality of the documentation.
Instead, feature parity was far apart; some backends were tested for certain attributes that others didn't have.

### The Good: Mustache

Clang-Doc still had issues, but last year's GSoC brought in great improvements that became the basis of my summer.
First, last year's GSoC contributor landed a large performance improvement.
I might not have been able to test Clang-Doc on Clang itself without it.

Second, and most important to my work, was introducing [Mustache templates](https://mustache.github.io/) to LLVM.
Mustache templates will allow Clang-Doc to shift away from manually generating HTML tags and eliminate high maintenance burdens.

## The first four weeks

### A small change of plans

While familiarizing myself with the codebase during the Community Bonding Period, I started to think about how much the project (and I) would benefit from a JSON backend.
The new HTML backend already created in-memory JSON objects to feed its templates, but it immediately discarded them once the HTML was created.
[This had already been suggested as a future improvement](https://github.com/llvm/llvm-project/issues/140094), so I decided to change my plans and implement this backend.

A JSON backend presented two immediate benefits:

1. We could use it as a feeder to the HTML Mustache templates, which was already the behavior, and use it as a feeder to other formats. Instead of manually generating Markdown, we could use templates fed by JSON.
2. As the main feeder format, the JSON output could be validated as the "main source of truth" to ensure our documentation quality is good.

I ended up ripping out the existing JSON code inside the HTML Mustache backend
and creating a separate JSON generator.
I created basic tests to check our exiting functionality for classes, functions, etc.
I was able to land this within about a week thanks to a lot of existing functionality in the Mustache backend.

After this JSON backend landed, I focused on my original goals for these 4
weeks which was strengthening our C++ support. That meant visiting Decls that
we didn't before and making sure to serialize all of their relevant information.
I began with concepts.

### Documentation Serialization Workflow

Adding support for documenting declarations follows the following workflow:

1. Adding a visit method to our recursive ASTVisitor.
2. Adding a serialize function that extracts the information we need.
3. Adding or modifying the bitcode write/read functions to ensure a new Info or Info member is serialized/deserialized.
4. Serializing it from one of our backends, in this case JSON.

I had suspected concepts would be the most time consuming feature to implement, so I decided to work on it first.
"Start with the hardest" usually works out best for me.
I landed the feature within about a week followed by global variables and friends.
I also introduced name mangling for our documentation filenames to avoid a double

## Another Change of Plans and The next 4 weeks

Implementing a new backend brought up another important decision that would affect my original timeline.
The newly-implemented JSON backend only employed one of the previously listed benefits, which was a centralized testing format.
I decided, along with my mentors, that it would probably be best to integrate the JSON and HTML Mustache backends to have the full benefits.

The integration took about a week.
Further development without the integration might have caused more divergence that would've had to be resolved later.
It also helped immediately in the next phase of the project that I had planned: comments.
