---
name: hoon
description: Hoon best practices. Use when writing or editing Hoon code; not when reading.
---

# Hoon best practices

## User instructions

There is no such thing as "assigning a variable" in Hoon, one can only edit its non-mutable subject, which is *Urbit OS in its entirety*, by creating a new copy of it with the new "face" (name reference to a computed expression) "pinned" (assigned) to the subject. As such — while you should not be shy about using the `=/` rune when you have a computation you need to refer to more than once — you should never use `=/` to pin a face you're only going to refer to once. In some cases you might need to break this guideline to do some type coercion that would be awkward otherwise (esp. when de-vasing a value) which is fine.

Never use `=/` to pin a gate (function) to the subject, those should be `++` arms.

There is no problem that cannot be solved by adding another layer of indirection except for the problem of having too many layers of indirection. Be extremely conservative about adding helper functions, adding them only when 1) inlining the code would be arduous to read and maintain or 2) writing a library `/lib` file which "exports" useful functions for other developers who might use this library. One perennial issue LLMs have with writing Hoon code is that when editing an `$agent:gall`, which is specifically a ten-arm struct, they add more arms because they love helper functions; if you want to add a helper function to a Gall agent, you may use the `=>` rune to compose a helper core below the Gall agent core into the Gall agent's subject, but again you should be conservative about this.

You may interpolate wide-form Hoon into tall-form Hoon, but can never interpolate tall-form Hoon into wide-form Hoon. Prefer tall-form Hoon.

Hoon comments may be in inline or "breathing" form. Contrary to the Hoon style guide, I prefer to write breathing comments with the breathing space *above* the comment like so.

```hoon
::
::  my comment
|%
++  foo  1
++  bar  '2'
--
```

Comments should generally be all-lowercase.

TODO comments in Hoon files are idiomatically written with `XX` instead of `TODO`.

## Tlon Corporation's Hoon reference

Further reading.

- [Fundamentals](./references/fundamentals.md): Subject-oriented programming, faces, type narrowing, formatting, irregular syntax
- [Patterns](./references/patterns.md): Composition idioms, error handling, common pitfalls
- [Syntax](./references/syntax.md): Practical syntax, stdlib, types, JSON, imports, scries
