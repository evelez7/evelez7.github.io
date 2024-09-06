---
title: "Google Summer of Code 2023"
description: "My experience hacking on clang in 2023"
tags: ["google summer of code"]
draft: true
---

During the summer of 2023 I had the opportunity to work on clang for Google Summer of Code.
I worked on clang's ExtractAPI, which is a neat feature that extracts symbols from source.
It's currently used for documentation but it's a versatile tool that can be used for a lot.
During my project, I added C++ support.

If you want a quick summary, here's my [final submission](https://summerofcode.withgoogle.com/archive/2023/projects/uBg3dUrw)
which links to my final report.

# What is ExtractAPI?
ExtractAPI uses clang's syntax tree to gather up all the symbols in a source 
file, along with relevant context, into an intermediate format before spitting
a serialized copy.

For example, the following:
```cpp
class Foo {
  public:
    static int getBar();
};
```

*getBar* will be stored in an internal structure that recognizes it as a static method member of Foo.
Foo will be a top level record that houses all of its internal **public** members.
Foo might also be part of a namespace or it might be a public nested class.

# Why is ExtractAPI?
