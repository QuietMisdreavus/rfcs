- Feature Name: `rustdoc_json`
- Start Date: 2018-08-21
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC decribes the design of a potential JSON output for the tool `rustdoc`, to allow tools to
lean on its data collection and refinement but provide a different front-end.

# Motivation
[motivation]: #motivation

The current HTML output of `rustdoc` is often lauded as a key selling point of Rust. Using this
ubiquitous tool, you can easily find nearly anything you need to know about a crate. However,
despite its versatility, the use of this specific output has its drawbacks:

- Viewing this output requires a web browser, with (for some features of the output) a JavaScript
  interpreter.
- The HTML output of `rustdoc` is explicitly not stabilized, to allow `rustdoc` developers the
  option to tweak the display of information, add new information, etc. However, this also means
  that converting this HTML into a different output is infeasbile.
- As the HTML is the only available output of `rustdoc`, its integration into centralized
  documentation browsers such as [Kythe] is difficult.

In addition, `rustdoc` had JSON output in the past, but it failed to keep up with the changing
language and [was taken out][remove-json] in 2016. With `rustdoc` in a more stable position, it's
possible to re-introduce this feature and ensure its stability.

[remove-json]: https://github.com/rust-lang/rust/pull/32773

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

(*Upon successful implementation/stabilization, this documentation should live in The Rustdoc
Book.*)

In addition to generating the regular HTML, `rustdoc` can create JSON files based on your crate.
These can be used by other tools to take information about your crate and convert it into other
output formats, insert into centralized documentation systems, create language bindings, etc.

To get this output, pass the `--output-format json` flag to `rustdoc`:

```console
$ rustdoc lib.rs --output-format json
```

This will output a JSON file in the current directory (by default). For example, say you have the
following crate:

```rust
//! Here are some crate-level docs!

/// Here are some docs for `some_fn`!
pub fn some_fn() {}

/// Here are some docs for `SomeStruct`!
pub struct SomeStruct;
```

After running the above command, you should get a `lib.json` file like the following (indented for
clarity):

```json
{
    "name": "lib",
    "src": "lib.rs",
    "module": {
        "id": [0, 0],
        "docs": "Here are some crate-level docs!",
        "attrs": [],
        "type": "mod",
        "inner": {
            "items": [
                {
                    "id": [0, 1],
                    "name": "some_fn",
                    "docs": "Here are some docs for `some_fn`!",
                    "attrs": [],
                    "type": "fn",
                    "inner": {
                        "decl": {
                            "inputs": []
                        },
                        "generics": {
                            "params": [],
                            "where_predicates": []
                        },
                        "unsafe": false,
                        "const": false,
                        "async": false,
                        "abi": "Rust"
                    }
                },
                {
                    "id": [0, 2],
                    "name": "SomeStruct",
                    "docs": "Here are some docs for `SomeStruct`!",
                    "attrs": [],
                    "type": "struct",
                    "inner": {
                        "struct_type": "unit",
                        "generics": {
                            "params": [],
                            "where_predicates": []
                        },
                        "fields": [],
                        "fields_stripped": false
                    }
                }
            ]
        }
    }
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

(*Upon successful implementation/stabilization, this documentation should live in The Rustdoc
Book.*)

When you request JSON output from `rustdoc`, you're getting a version of the Rust abstract syntax
tree (AST), so you could see anything that you could export from a valid Rust crate. The following
types can appear in the output:

(*This documentation is deliberately left incomplete; filling it out will happen during the design
process.*)

## Crate

A Crate is the root of the AST, and the structure given for the whole output file. This contains
information about the crate as a whole, as well as information about the items in the root module.

Name      | Type    | Description
----------|---------|------------------------------------------------------------------------------
`name`    | String  | The name of the crate. If `--crate-name` is not given, based on the filename.
`version` | String  | *Optional.* The version string given to `--crate-version`, if any.
`src`     | String  | The filename of the root of the crate.
`module`  | Item    | The Item corresponding to the root module.

## Item

An Item represents anything that can hold documentation - modules, structs, enums, functions,
traits, type aliases, and more. The Item data type holds the fields that can apply to anything, and
leaves kind-specific details to the `inner` field.

Name      | Type    | Description
----------|---------|------------------------------------------------------------------------------
`id`      | Array   | A pair of numbers that together create a unique ID for this Item.
`name`    | String  | The name of the item, if present. Some items, like impl blocks, do not have names.
`docs`    | String  | The extracted documentation text from the item.
`attrs`   | Array   | The attributes (other than doc comments) on the item, rendered as strings.
`cfg`     | String  | *Optional.* Conditional-compilation information given by `#[doc(cfg)]`, if present.
`type`    | String  | The kind of Item this is. Determines what fields are in `inner`.
`inner`   | Object  | The type-specific fields describing this Item. Check the `type` field to determine what's available.

### `type == "mod"`

When `type` is `"mod"`, the Item refers to a module.

Name     | Type   | Description
---------|--------|--------------------------------------------------------------------------------
`items`  | Array  | The list of Items contained within this module.

### `type == "fn"`

When `type` is `"fn"`, the Item refers to a function.

Name       | Type     | Description
-----------|----------|----------------------------------------------------------------------------
`decl`     | Decl     | Information about the function signature, or declaration.
`generics` | Generics | Information about the function's type parameters and `where` clauses.
`unsafe`   | Boolean  | Whether the function is marked `unsafe`.
`const`    | Boolean  | Whether the function is marked `const`.
`async`    | Boolean  | Whether the function is marked `async`.
`abi`      | String   | The ABI string on the function. Non-`extern` functions have a `"Rust"` ABI, whereas `extern` functions without an explicit ABI are `"C"`.

### `type == "struct"`

When `type` is `"struct"`, the Item refers to a struct definition.

Name          | Type     | Description
--------------|----------|-------------------------------------------------------------------------
`struct_type` | String   | Either "plain" for braced structs, "tuple" for tuple structs, or "unit" for unit structs.
`generics`    | Generics | Information about the struct's type parameters and `where` clauses.
`fields`      | Array    | The list of fields in the struct. Always an array of Item with `type == "structfield"`.
`fields_stripped` | Boolean | Whether any fields have been removed from the result, due to being private or hidden.

(*Complete documentation information is deferred to final design and implementation work.*)

# Drawbacks
[drawbacks]: #drawbacks

- By supporting JSON output for `rustdoc`, we should consider how much it should mirror the internal
  structures used in `rustdoc` and in the compiler. Depending on how much we want to stabilize, we
  could accidentally stabilize the internal structures of `rustdoc`.

- Even if we don't accidentally stabilize `rustdoc`'s internals, adding JSON output adds *another*
  thing that must be kept up to date with language changes, and another thing for compiler
  contributors to potentially break with their changes. Because the HTML output is only meant for
  display, it requires less vigilant updating when new language features are added.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- **Status quo.** Keep the HTML the way it is, and make users who want a machine-readable version of
  a crate either parse it themselves or use `save-analysis`. (See below for discussion about
  `save-analysis`.) In the absence of an accepted JSON output, the `--output-format` flag in rustdoc
  remains deprecated and unused.
- **Alternate data format (XML, Bincode, CapnProto, etc).** JSON was selected for its ubiquity in
  available parsers, but selecting a different data format may provide benefits for file size,
  compressibility, speed of conversion, etc.
- **Alternate data structure.** Massage the data so that it echoes something closer to user
  perception, rather than the internal `clean` AST that they're currently modeled after. Such a
  refinement can be provided in a future RFC, as a potential alternate data format to output, if
  necessary.

# Prior art
[prior-art]: #prior-art

A handful of other languages and systems have documentation tools that output an intermediate
representation separate from the human-readable outputs:

- [PureScript] uses an intermediate JSON representation when publishing package information to their
  [Pursuit] directory. It's primarily used to generate documentation, but can also be used to
  generate `etags` files.
- [Doxygen] has an option to generate an XML file with the code's information.
- [Haskell]'s documentation tool, [Haddock], can generate an intermediate representation used by the
  type search engine [Hoogle] to integrate documentation of several packages.
- [Kythe] is a "(mostly) language-agnostic" system for integrating documentation across several
  langauges. It features its own schema that code information can be translated into, that services
  can use to aggregate information about projects that span multiple languages.
- [GObject Introspection] has an intermediate XML representation called GIR that's used to create
  langauge bindings for GObject-based C libraries. While (at the time of this writing) it's not
  currently used to create documentation, it is a stated goal to use this information to document
  these libraries.

[PureScript]: http://www.purescript.org/
[Pursuit]: https://pursuit.purescript.org/
[Doxygen]: https://www.stack.nl/~dimitri/doxygen/
[Haskell]: https://www.haskell.org/
[Haddock]: https://www.haskell.org/haddock/
[Hoogle]: https://www.haskell.org/hoogle/
[Kythe]: http://kythe.io/
[GObject Introspection]: https://gi.readthedocs.io/en/latest/

Moreover, Rust itself has another external representation already: `save-analysis`. This information
is collected and used by the Rust Language Server (RLS) to provide auto-complete and type
documentation information for editors and IDEs. However, the RLS and rustdoc have different design
goals, and require different information from the compiler, so the use of `save-analysis` data for
documentation occasionally suffers from this design tension. Allowing for `rustdoc` to create its
own intermediate representation lets these datasets specialize for their intended uses.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is it possible to fold this information into `save-analysis`? Being able to use this existing tool
  to output documentation information would save duplication of effort, since their goals are fairly
  similar.
- What is the stabilization story? As langauge features are added, this representation will need to
  be extended to accommodate it. As this will change the structure of the data, what does that mean
  for its consumers?
- How do we represent types, and allow people to properly collect type information from places like
  struct fields, function signatures, etc? `rustdoc`'s own `clean::Type` enum is large and recursive
  and represents a lot of primitives, in addition to ultimately deferring the lookup to a DefId.
- The `id` field is basically a copy of DefId from inside the compiler; is there a better way to
  represent it? How necessary is it to have? Will using numbers run into problems with JSON parsers
  that treat all numbers as floats?
- Where should we store impls?
  - In `rustdoc`, trait impls are pooled in the crate root (or placed in the module they're declared
    in), but before rendering, the information is copied into two maps: one mapping traits to their
    implementors, and one mapping types to all their impls (inherent or trait).
  - The HIR copies all trait impls into a map connecting traits to their implementors, though
    they're also available in the location they're defined if you iterate over the HIR.
  - However, while trait impls are unburdened by scope rules for visibility, *inherent* impls are.
    Currently, if `--document-private-items` is passed, the methods defined in an impl are all
    pooled into a struct, and any `pub(restricted)` scopes link to their respective modules.
    However, private methods are just shown as private, without any information connecting them to
    where they're allowed.
  - This leads to wanting to pool impls on their type (and copying them in to their trait for trait
    impls), and leaving the visibility fix for a later PR.
