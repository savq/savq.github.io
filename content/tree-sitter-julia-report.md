+++
title = "Improving tree-sitter-julia"
authors = ["Sergio A. Vargas"]
date = 2022-12-27
+++

> This report was originally presented as part of an independent contractor agreement with NumFOCUS/Julia Language.

This report presents the work done to improve the [tree-sitter-julia](https://github.com/tree-sitter/tree-sitter-julia) parser.
The parser is generated from a DSL for context-free grammars,
so the terms _parser_ and _grammar_ are used interchangeably.

The main objective was to improve the design and implementation of the tree-sitter parser for Julia,
with the goal of parsing 99% of the lines of code from a [set of popular packages](https://github.com/returntocorp/ocaml-tree-sitter-semgrep/blob/main/lang/julia/projects.txt),
measured by the `./scripts/lang-stat` and `./lang/test-lang` shell scripts in the [returntocorp/ocaml-tree-sitter-semgrep](https://github.com/returntocorp/ocaml-tree-sitter-semgrep) repository.

The improvements to the project were a success, and the grammar now parses 99.24% out 350,787 lines of Julia code correctly.


## Former state of the grammar

The grammar was originally developed by Max Brunsfeld, the creator of tree-sitter.
This original grammar was similar to the grammars of languages with better tree-sitter support, like Ruby and JavaScript.
While it worked well for simple Julia code, the grammar lacked complete support for Julia specific features like:

- Multidimensional arrays.
- quotations, interpolations and other meta-programming syntax.
- Unicode identifier support.


## Improvements to the grammar

The improvements were made in a dozen pull requests to the
[tree-sitter-julia](https://github.com/tree-sitter/tree-sitter-julia/pulls?q=is%3Apr+is%3Aclosed) repository.
These can be grouped into two stages of development.

### First stage of development

The first stage consisted of broad changes to the grammar,
focusing on aspects of the grammar that are well understood.
I made few large changes, following a test driven development workflow.
Each PR in this stage roughly corresponds to a group of syntactic forms:

- Modules and type definitions: [#54](https://github.com/tree-sitter/tree-sitter-julia/pull/54)
- Function and macro definitions: [#54](https://github.com/tree-sitter/tree-sitter-julia/pull/54)
- Statements (if statements, for statements): [#64](https://github.com/tree-sitter/tree-sitter-julia/pull/64)
- Primary expressions (function calls, array indexing, etc): [#72](https://github.com/tree-sitter/tree-sitter-julia/pull/72)
- Assignments: [#78](https://github.com/tree-sitter/tree-sitter-julia/pull/78)
- Quotables (matrices, tuples, etc): [#83](https://github.com/tree-sitter/tree-sitter-julia/pull/83)


### Nomenclature

Throughout this first stage of development,
I also made changes to the nomenclature to closely match the Julia intermediate representation.
Julia users that write macros will probably be familiar with these naming conventions.

The Julia documentation and internals give ambiguous naming conventions for many syntactic forms.
In these cases I left the existing names unaltered.
For example, while Julia is an expression-based language, many syntactic forms are still referred to as “statements”.

In other cases, there's no explicit equivalent in the reference parser to a derivation in the tree-sitter parser.
Here, I made up a names hoping they’re sufficiently clear.
For example, the `_quotable` rule was defined for expressions that can be quoted without parentheses,
but this term is never used in the Julia documentation.


### Second stage of development

The second stage focused on more specific changes, and adding features that aren't explicitly listed Julia documentation.
I listed possible changes based on existing parse failures, and worked on fixing each entry in the list.
As the grammar improved and other errors became more apparent, I updated the list and kept iterating.
During this second stage the main set of rules I worked on were:

- Arrays, tuples and function parameters: [#83](https://github.com/tree-sitter/tree-sitter-julia/pull/83)
- Quotations, interpolations and macros: [#85](https://github.com/tree-sitter/tree-sitter-julia/pull/85)
- Unicode support: [#83](https://github.com/tree-sitter/tree-sitter-julia/pull/83)
- Prefixed strings: [#76](https://github.com/tree-sitter/tree-sitter-julia/pull/76)
- Type annotations and type parameters:
  [#77](https://github.com/tree-sitter/tree-sitter-julia/pull/77),
  [#87](https://github.com/tree-sitter/tree-sitter-julia/pull/87).


The first stage focused on trying to simplify the grammar and improve code reuse,
but some of those changes had to be undone in the second stage for multiple reasons.
For example, many derivations can be parsed by Julia, but have no associated semantics,
so in practice they're invalid code.
However, these "invalid" expressions can be used inside macros to define custom DSLs.
Popular packages such as [JuMP.jl](https://github.com/jump-dev/JuMP.jl) and [Gen.jl](https://github.com/probcomp/Gen.jl)
use "invalid" code in their DSLs.


### Collaboration with the community

During development, feedback from the community was very valuable.
I asked Neovim users in the Julia Zulip chat to file issues they found,
and Emacs users also approached the project after seeing the uptick in contributions.
I discussed issues with the maintainers of [nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter)
and contributors to the [Helix text editor](https://github.com/helix-editor/helix).
I also reviewed and gave feedback on the PRs that were opened while I was working on the grammar.


## Difficulties

Some of the notable difficulties I had during development can be grouped into the following categories:

#### Julia has no formal grammar

There’s no formal lexical or syntactical grammar for the language,
so Julia syntax is subjected to the bugs in the reference Femtolisp implementation.
JuliaSyntax.jl, a project aimed at replacing the Femtolisp parser, has a great [overview of these bugs](https://github.com/JuliaLang/JuliaSyntax.jl#flisp-parser-bugs).


#### Julia syntax has many context-sensitive rules

Context-sensitive rules can be easy to implement in an imperative, recursive-descent parser, but are very difficult to specify in a formal grammar.
For example, the rules relating to whitespace-sensitivity and juxtapositions cannot be defined in the tree-sitter grammar DSL.

Parsing context-sensitive rules correctly requires calling an external scanner written in C.
The scanner was improved to handle some of these rules, but others were left partially specified.


#### Loose rules vs tighter rules with more meaning

Julia doesn't distinguish some syntactic forms that have different semantic purposes.
In particular, it doesn't distinguish between "patterns" and expressions.

For example, function parameters are parsed as tuples.
The following quotation is syntactically valid, but cannot be evaluated because integer literals are not valid parameters.

```julia
quote
  function f(1, 2, 3) # Nonsense, but it parses fine.
  end
end
```

There are many similar cases all over the grammar. That could be a whole post on its own.

On one hand, this liberal use of syntax allows for more DSLs.
One could imagine a macro for pattern matching that did allow literals as parameters.

On the other hand, having more distinctions in the grammar is useful for static analysis.
Distinguishing parameters can inform about the scope of variables and can be used by automated refactoring tools.
The tree-sitter grammar does have these distinctions because the benefits for text editors and higher-level libraries
seem to be worth the cost (i.e. it won't parse the snippet above).


## Future work

The tree-sitter parser still doesn't parse 100% of lines of code.
However, the few syntactic forms that aren't parsed correctly all have alternatives.
At this point, most additions to the grammar will have diminishing returns.

The workarounds required to parse context-sensitive rules resulted in a loss of performance, and increased binary size of the compiled parser.
Future development should focus on offloading more rules to the external scanner,
or contributing to tree-sitter to facilitate defining these rules in the grammar DSL.

The tree-sitter grammar could also be used as a starting point to specify a formal grammar for Julia.

<!-- pandoc --variable urlcolor=teal --lua-filter ./smallcaps.lua -o draft.pdf report.md -->

