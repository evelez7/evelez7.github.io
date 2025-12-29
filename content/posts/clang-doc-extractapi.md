---
title: "Are clang-doc and ExtractAPI Compatible?"
description: "Thinking about where ExtractAPI could help in clang-doc."
tags: ["clang", "clang-tools-extra", "clang-doc", "extractapi"]
---

At LLVM Dev Meeting 2025, [I was asked whether Clang ExtractAPI could be used in clang-doc's JSON generation](https://youtu.be/vuHF_ZeAPVI?si=MVEoQleXHBnE9FCy&t=1096).
My answer was, because of the way that clang-doc uses templating, ExtractAPI's JSON output wasn't compatible with clang-doc.
This actually isn't entirely true and I wanted to clarify that answer since these tools actually do a lot of similar things, but they differ heavily architecturally.
It also might be helpful to clarify since the LLVM project now has at least two tools that directly document source code into JSON.
Spoiler: They're compatible.
Sort of.

## How data representation differs

The differences in the tools start with their main data structures.
clang-doc uses what it calls an `Info` while ExtractAPI uses `DeclarationFragments`.
Here's what a simple clang-doc `Info` looks like (both of these records are simplified):

```cpp
struct Info {
  std::string Name;
  std::string Type;
  bool IsStatic;
};
```

`TypeInfo` is another record that holds information relating to the `Info`'s type, like whether it's user-defined or built-in.
Here's what ExtractAPI's `DeclarationFragments` looks like:

```cpp
struct Fragment {
  std::string Spelling;
};

struct DeclarationFragments {
  std::vector<Fragment> Fragments;
};
```

So, ExtractAPI essentially holds a `vector` of `std::string` that makes up the symbol's source spelling.
clang-doc does hold a string for the symbol's name, but it currently doesn't construct a full source spelling.
Instead, the templates handle qualifiers and specifiers, hence the `IsStatic` boolean:

```html
{{#IsStatic}}static {{/IsStatic}}{{Type}} {{Name}}
```

The template ends up being rendered as something like `static int StaticVal`.
Since clang-doc doesn't serialize the full source spelling, and since it uses the Mustache templating language (which is popularly referred to as "logic-less templates"), a lot of the JSON properties act as the logic for something to be serialized or not.
Hence, clang-doc templates can get pretty dense.
Here's something I just wrote to get function template declarations to show properly:

```html
<a class="sidebar-item" href="#{{USR}}">{{Name}}{{#Template}}{{#Specialization}}&lt;{{#Parameters}}{{Param}}{{^End}}, {{/End}}{{/Parameters}}&gt;{{/Specialization}}{{/Template}}</a>
```

Not terrible, not great.
So why not just construct a full spelling?
Well, in some cases, the flexibility is pretty useful.

```html
{{#Template}}
<pre><code>template &lt;{{#Parameters}}{{Param}}{{^End}}, {{/End}}{{/Parameters}}&gt;</code></pre>
{{/Template}}
<h1>{{TagType}} {{Name}}</h1>
```

This lets us keep the template declaration above the name, which helps keep the name justified to the left.
You can ask again, though, why not construct the full template spelling and emit it separately?
That keeps the flexibility while reducing template complexity.
That's a pretty good point, and definitely something we should consider for clang-doc, but that doesn't necessarily lead to integrating ExtractAPI.
At least, not all of it.

### How the JSON format differs

ExtractAPI and clang-doc actually serialize a lot of the same information, such as source location or type information.
An important distinction is that at the time of JSON generation, ExtractAPI doesn't have any readily-available information about its symbols.
It does of course have type info (is this a class or function?) but properties like `static` or `const` would need to be evaluated from the spelling, which is where it currently clashes with clang-doc.
ExtractAPI also currently only has a Symbol Graph serializer which can feed into Swift-DocC.
Clang-doc's JSON generation doesn't have a format; it just exists to feed the templates.

## Architectural differences

ExtractAPI is much more than just the JSON it generates, and in fact it's the architecture that really prevents full integration.
It runs via an `ASTConsumer` invoked by a frontend action that puts all of the headers that are supplied into a temporary file.
The AST from this temporary file is what ExtractAPI actually documents.
In contrast, clang-doc visits translation units in parallel via LibTooling.
Using ExtractAPI would mean completely replacing clang-doc's frontend functionalities which are much more than just AST visitation.
clang-doc handles directory and file creation for documentation placement and also loads assets.
clang-doc would have had to be created with every aspect of ExtractAPI in mind, and even then many features would not have been possible that give clang-doc its many great strengths.

ExtractAPI also has a very different purpose as evidenced by its name: it's really meant to document headers which provide the API.
Of course, you can feed it source files and it will document them, but since they're all put into a temporary file I'm not sure how it would perform.
I'm also not sure how duplicate symbols are resolved; of course, ExtractAPI doesn't really need to worry about that because that's not its use case.
On the other hand, clang-doc solves this via a map-reduce pattern powered by LibTooling.
Thus, clang-doc is really great at creating full codebase documentation where developers will be hacking on what clang-doc is documenting.

Overall, there are some pieces of ExtractAPI that clang-doc can leverage for much better functionality.
They are architecturally incompatible which prevents the reuse of JSON generation, but that's due to how great they are at tackling specific problems.

