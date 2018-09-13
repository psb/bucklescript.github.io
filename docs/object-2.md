---
title: Object 2
---

There is an alternative way of binding to JavaScript objects if the previous section's `bs.deriving abstract` does not suit your needs:

- You do not want to declare a type beforehand.
- You want your object to be "structural", i.e., your function wants to accept "any object with the field `age`, for example, not just a particular object whose type definition is declared above".

> A reminder of the distinction between a record and an object can found [here](https://reasonml.github.io/docs/en/record#record-types-are-found-by-field-name).

This section describes how BuckleScript uses OCaml's object facility to achieve this alternative way of binding and compiling to JavaScript objects.

## Pitfall

First, note that we **cannot** use the ordinary OCaml/Reason object type like this:

```ocaml
type person = <
  name: string;
  age: int;
  job: string
>
```

```reason
type person = {
  .
  name: string,
  age: int,
  job: string
};
```

You can still use this feature, but this OCaml/Reason object type does **not** compile to a clean JavaScript object! Unfortunately, this is because OCaml/Reason objects work too differently from JS objects.

## Actual Solution

BuckleScript wraps the regular OCaml/Reason object type with `Js.t` in order to control and track a **subset** of operations and types that we know would compile cleanly to JavaScript. This is how it looks:

```ocaml
type person = <
  name: string;
  age: int;
  job: string
> Js.t

external john : person = "john" [@@bs.val]
```

```reason
type person = Js.t({
  .
  name: string,
  age: int,
  job: string
});

[@bs.val] external john : person = "john";
```

**From now on**, we will call the BuckleScript interop object a "`Js.t` object" to disambiguate it from a normal object and a JS object.

Because object types are used often, Reason gives it some nice syntax sugar: `Js.t({. name: string})` will reformat to `{. "name": string}`.

### Accessors

#### Read

To access a field use `##`: `let johnName = john##name`.

#### Write

To modify a field, you need to first mark a field as mutable. By default, the `Js.t` object type is immutable.

```ocaml
type person = < age : int [@bs.set] > Js.t
external john: person = "john" [@@bs.val]

let _ = john##age #= 99
```

```reason
type person = {. [@bs.set] "age": int};
[@bs.val] external john : person = "john";

john##age #= 99;
```

**Note**: you cannot use dynamic/computed keys in this paradigm.

#### Call

To call a method of a field, mark the function signature as `[@bs.meth]`:

```ocaml
type person = < say : string -> string -> unit [@bs.meth] > Js.t
external john: person = "john" [@@bs.val]

let _ = john##say "hey" "jude"
```

```reason
type person = {. [@bs.meth] "say": (string, string) => unit};
[@bs.val] external john : person = "john";

john##say("hey", "jude");
```

**Why `[bs.meth]`**? Why not just call it directly? A JS object might carry around a reference to `this`, and, infamously, what `this` points to can change. OCaml/BuckleScript functions are curried by default; this means that if you intentionally curry `say`, by the time you fully apply it, the `this` context could be wrong:

```ocaml
(* wrong *)
let talkTo = john##say("hey")

let jude = talkTo "jude"
let paul = talkTo "paul"
```

```reason
/* wrong */
let talkTo = john##say("hey");

let jude = talkTo("jude");
let paul = talkTo("paul");
```

To ensure that folks don't accidentally curry a JavaScript method, we track every method call using `##` to make sure it is fully applied _immediately_. Under the hood, we effectively turn a function-looking call into a special `bs.meth` call (it only _looks_ like a function). Annotating the type definition of `say` with `bs.meth` completes this check.

### Creation

You can use the `[%bs.obj putAnOCamlRecordHere]` DSL to create a `Js.t` object:

```ocaml
let bucklescript = [%bs.obj {
  info = {author = "Bob"}
}]

let name = bucklescript##info##author
```

```reason
let bucklescript = [%bs.obj {
  info: {author: "Bob"}
}];

let name = bucklescript##info##author;
```

Because object values are also used often, Reason, again, gives it some nice syntax sugar. `[%bs.obj {foo: 1}]` will reformat to `{"foo": 1}`.

**Note**: there is no syntax sugar for creating an empty object in OCaml or Reason, i.e., this does not work: `[@bs.obj {}]`. Please use `Js.Obj.empty()` instead.

The created object will have an inferred type - no type declaration needed! The above example will be inferred as:

```ocaml
< info: < author: string > Js.t > Js.t
```

```reason
{. "info": {. "author": string}}
```

**Note**: since the value has its type inferred, **do not** accidentally do this:

```ocaml
type person = <age: int> Js.t
let jane = [%bs.obj {age = "hi"}]
```

```reason
type person = {. "age": int};
let jane = {"age": "hi"};
```

Do you see what went wrong here? We have declared a `person` type, but `jane` is inferred as its own type, so `person` is ignored and no error happens! To give `jane` an explicit type, simply annotate it: `let jane: person = ...`. This will create an error correctly.
