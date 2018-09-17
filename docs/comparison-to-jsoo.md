---
title: Comparison to Js_of_ocaml
---

Js_of_ocaml is a popular compiler which compiles OCaml’s bytecode into JavaScript. It is the inspiration for this project, and has already been under development for several years and is ready for production. BuckleScript’s motivation, like js_of_ocaml, is to unify the ubiquity of the JavaScript platform and the truly sophisticated type system of OCaml.

However, there are a few areas that BuckleScript approaches differently:

- Js_of_ocaml takes low-level bytecode from the OCaml compiler, whereas BuckleScript takes the high-level raw lambda representation from the OCaml compiler.
- Js_of_ocaml focuses more on the existing OCaml ecosystem (OPAM), but BuckleScript’s major goal is to target NPM/Yarn and existing JS workflows.
- Js_of_ocaml and BuckleScript have a slightly different runtime encoding in several places. For example, BuckleScript encodes an OCaml Array as a JS Array, while js_of_ocaml requires its index 0 to be of value 0.

See also the [What & Why](what-why.md) page for more information on BuckleScript's emphasis.
