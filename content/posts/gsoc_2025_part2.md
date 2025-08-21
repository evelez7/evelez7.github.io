---
title: "Google Summer of Code 2025: Part Two"
tags: ["google summer of code", "gsoc"]
---

In the second phase of my GSoC project, I focused on Clang-Doc's comment support.
Robust comment support is essential for documentation.
Clang's comment parser understands a variety of different comments, but either Clang-Doc didn't display them well or didn't recognize them.

## Grouped and Ordered Comments

Comments weren't ordered in Clang-Doc's HTML documentation.
They were just displayed in whatever order they were serialized in, which is the order that they're written in source.
This meant comments would be extremely difficult to read - you don't want to search for another parameter comment after reading the first one, even if they're expected to be written in order in source.

Funnily enough, Mustache made this a little more complicated.
The only logic operation that Mustache has to check if a field exists is an iteration like `{{#Fields}}`, but any header that denotes a comment section would be duplicated.

```html
{{#Fields}}
<h3>Field Header</h3>
  {{FieldInfo}}
{{/Fields}}
```

All of the logic to order them needs to be done in the serialization to JSON itself, so I overhauled our comment organization.
Previously, Clang-Doc's comments were organized exactly as in Clang's AST like the following:

- FullComment
  - BriefComment
    - ParagraphComment
      - TextComment
      - TextComment
  - BriefComment
    - ParagraphComment

Everything was unnecessarily nested under a FullComment, and TextComments were also unnecessarily nested.
Every non-verbatim comment's text was held in one ParagraphComment.
Since there was only one, we could reduce some boilerplate by directly mapping to the array of TextComments.

After the change, Clang-Doc's comments were structured like this:

- BriefComments
  - TextCommentArray
  - TextCommentArray
- ParagraphComments
  - TextCommentArray

Now, we can just iterate over every type of comment, which means iterating over every array.
This left our JSON documentation with a few more fields, since one is needed for every Doxygen command, but with easier identification of what comments exist in the documentation.

## Finding and Deferring Challenging Features

Something useful that I learned during this period was identifying and deferring technical features that aren't in project's immediate vision.
It's interesting because there are features that would be nice-to-have but don't have the same impact as others.
Especially in Clang-Doc's current state, core functionality changes have much higher impact than what might be seen in other projects.
I also learned, though, about trying to leave the situation in a place where it (hopefully) wont be forgotten.

### Doxygen Grouping

Something I identified in my project proposal was Doxygen grouping and member semantics.
Doxygen has a very useful [grouping](https://www.doxygen.nl/manual/grouping.html) feature that allows structures to be grouped under the same heading or on their own separate pages.
Clang uses this feature in a few places like in [llvm::sys::path](https://llvm.org/doxygen/namespacellvm_1_1sys_1_1path.html).
It would be a great feature, but while working on our comment structure, I realized it was much more complicated than I anticipated.

We ended up [opening up an issue](https://github.com/llvm/llvm-project/issues/151184#issuecomment-3133596874) for Clang to track this issue.
There would most likely have to be some major changes to Clang's comment parsing and Clang's own parsing. That's because a lot of the group opening tokens in Clang are free-floating, like so:

```cpp
/// @{

class Foo {};
```

That `@{` wont actually appear if you dump the Clang AST because of the formatting; only comments directly above a declaration are attached to a Decl in the AST.
My mentors wisely advised that this would be too much to even consider (and could probably be its own GSoC project).

### Cross-referencing

Something else I realized we would want to eventually support was cross-referencing entities.
In Doxygen you can use the `@copydoc` command to copy the documentation from one entity to another.
Doxygen also displays where an entity is referenced, like what classes invoke a particular function.

Clang-Doc has no support for this kind of behavior.
There would need to be some preprocessing step where any reference to another entity would need to be identified and then resolved somewhere else.
One of my mentors pointed out that it would be great to do during the reduction step where every Info is being visited anyways.

This actually wasn't something I had even considered in my proposal besides identifying that `@copydoc` wasn't supported by the comment parser.
It's a common feature of modern documentation, so hopefully someday soon Clang-Doc can acquire it.

## A Small Taste of the Fruits of My Labor

I previously mentioned that I spent a week aligning the JSON and HTML Mustache backends.
This proved to be absolutely necessary for a comment rework.
Because of the alignment, I only had to change the JSON backend to immediately see the changes in HTML.
Without the JSON-Mustache synergy, I wouldn't have been able to immediately visually test the changes.
Instead, I would've been contributing to more of the backend divergence we had identified earlier.
