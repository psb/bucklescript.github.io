---
title: Function
---

Modeling a JS function is like modeling a normal value:

```ocaml
external encodeURI: string -> string = "encodeURI" [@@bs.val]
let result = encodeURI "hello"
```

```reason
[@bs.val] external encodeURI: string => string = "encodeURI";
let result = encodeURI("hello");
```

We also expose a few special features, which are described below.

## Labeled Arguments

OCaml has labeled arguments (that are potentially optional). These work on an `external` too! You would use them to _fix_ a JS function's unclear usage. Assuming we are modeling this:

```js
function draw(x, y, border) {
   /* border is optional and defaults to false */
}
draw(10, 20)
draw(20, 20, true)
```

It would be nice if, on the BS side, we can bind and call `draw` while labeling things a bit:

```ocaml
external draw: x:int -> y:int -> ?border:bool -> unit -> unit = "" [@@bs.val]

let _ = draw ~x:10 ~y:20 ~border:true ()
let _ = draw ~x:10 ~y:20 ()
```

```reason
[@bs.val] external draw: (~x: int, ~y: int, ~border: bool=?, unit) => unit = "";

draw(~x=10, ~y=20, ~border=true, ());
draw(~x=10, ~y=20, ());
```

Output:

```js
draw(10, 20, true);
draw(10, 20, undefined);
```

We have compiled to the same function, but now the usage is much clearer on the BS side thanks to labels!

**Note**: in this particular case, you need a unit - `()` - after `border` since `border` is an [optional argument at the last position](https://reasonml.github.io/docs/en/function.html#optional-labeled-arguments). Not having a unit to indicate you have finished applying the function would generate a warning.

## Object Method

Functions attached to JS objects require a special way of binding to them. This is done using `bs.send`:

```ocaml
type document (* abstract type for a document object *)
external getElementById: document -> string -> Dom.element = "getElementById" [@@bs.send]
external doc: document = "document" [@@bs.val]

let el = getElementById doc "myId"
```

```reason
type document; /* abstract type for a document object */
[@bs.send] external getElementById: (document, string) => Dom.element = "getElementById";
[@bs.val] external doc : document = "document";

let el = getElementById(doc, "myId");
```

Output:

```js
var el = document.getElementById("myId");
```

In a `bs.send` the object is always the first argument. Actual arguments of the method follow (this is a bit like modern OOP objects).

### Chaining

Have you ever used `foo().bar().baz()` chaining ("fluent api") in JS OOP? We can model this in BuckleScript using the fast pipe operator, which is described in its [own](fast-pipe.md) section.

## Variadic Function Arguments

You might have JS functions that take an arbitrary number of arguments. BuckleScript supports modeling those under the condition that the arbitrary arguments are homogenous (i.e., of the same type). To do so, add `bs.splice` to your `external`.

```ocaml
external join: string array -> string = "" [@@bs.module "path"] [@@bs.splice]
let v = join [| "a"; "b"|]
```

```reason
[@bs.module "path"] [@bs.splice] external join: array(string) => string = "";
let v = join([|"a", "b"|]);
```

Output:

```js
var Path = require("path");
var v = Path.join("a", "b");
```

_`bs.module` will be explained in the [Import & Export](import-export.md) section_.

## Modeling Polymorphic Function

Apart from the above special-case, JS functions are often arbitrarily overloaded in terms of argument types and number. How would you bind to those?

### Trick 1: Multiple `external`s

If you can exhaustively enumerate the many forms an overloaded JS function can take, simply bind to each differently:

```ocaml
external drawCat: unit -> unit = "draw" [@@bs.module "Drawing"]
external drawDog: giveName:string -> unit = "draw" [@@bs.module "Drawing"]
external draw : string -> useRandomAnimal:bool -> unit = "draw" [@@bs.module "Drawing"]
```

```reason
[@bs.module "Drawing"] external drawCat: unit => unit = "draw";
[@bs.module "Drawing"] external drawDog: (~giveName: string) => unit = "draw";
[@bs.module "Drawing"] external draw: (string, ~useRandomAnimal: bool) => unit = "draw";
```

Notice how all three externals bind to the same JS function called `draw`.

### Trick 2: Polymorphic Variant + `bs.unwrap`

If you have the irresistible urge of saying "if only this JS function argument was a variant instead of informally being either `string` or `int`", then good news: we do provide such `external` features through annotating a parameter as a polymorphic variant! Assuming you have the following JS function you would like to bind to:

```js
function padLeft(value, padding) {
  if (typeof padding === "number") {
    return Array(padding + 1).join(" ") + value;
  }
  if (typeof padding === "string") {
    return padding + value;
  }
  throw new Error(`Expected string or number, got '${padding}'.`);
}
```

Conceptually, `padding` is really a variant. Let's model it as such.

```ocaml
external padLeft:
  string
  -> ([ `Str of string
      | `Int of int
      ] [@bs.unwrap])
  -> string
  = "" [@@bs.val]

let _ = padLeft "Hello World" (`Int 4)
let _ = padLeft "Hello World" (`Str "Message from BS: ")
```

```reason
[@bs.val]
external padLeft: (
  string,
  [@bs.unwrap] [
    | `Str(string)
    | `Int(int)
  ])
  => string = "";

padLeft("Hello World", `Int(4));
padLeft("Hello World", `Str("Message from BS: "));
```

Obviously, the JS side could not have an argument that's a polymorphic variant! But here, we are just piggy backing on polymorphic variants' type checking and syntax. The secret is the `[@bs.unwrap]` annotation on the type. It strips the variant constructors and compiles to just the payload's value:

```js
padLeft("Hello World", 4);
padLeft("Hello World", "Message from BS: ");
```

## Constrain Arguments Better

Consider the Node `fs.readFileSync`'s second argument. It can take a string, but really only a defined set: `"ascii"`, `"utf8"`, etc. You can still bind it as a string, but we can use polymorphic variants plus `bs.string` to ensure that our usage is more correct:

```ocaml
external readFileSync:
  name:string ->
  ([ `utf8
   | `useAscii [@bs.as "ascii"]
   ] [@bs.string]) ->
  string = ""
  [@@bs.module "fs"]

let _ = readFileSync ~name:"xx.txt" `useAscii
```

```reason
[@bs.module "fs"]
external readFileSync: (
  ~name: string,
  [@bs.string] [
    | `utf8
    | [@bs.as "ascii"] `useAscii
  ])
  => string = "";

readFileSync(~name="xx.txt", `useAscii);
```

Output:

```js
var Fs = require("fs");
Fs.readFileSync("xx.txt", "ascii");
```

- Attaching `[@bs.string]` to the whole polymorphic variant type makes its constructor compile to a string of the same name.
- Attaching a `[@bs.as "foo"]` to a constructor lets you customize the final string.

Now, passing something like `"myOwnUnicode"`, or other variant constructor names, to `readFileSync` will error correctly.

Aside from string, you can also compile an argument to an integer using `bs.int`, instead of `bs.string`, in a similar way:

```ocaml
external test_int_type:
  ([ `on_closed
   | `on_open [@bs.as 20]
   | `in_bin
   ]
   [@bs.int]) -> int =
  "" [@@bs.val]

let _ = test_int_type `in_bin
```

```reason
[@bs.val]
external test_int_type: (
  [@bs.int] [
    | `on_closed
    | [@bs.as 20] `on_open
    | `in_bin
  ])
  => int = "";

test_int_type(`in_bin);
```

`on_closed` will compile to `0`, `on_open` to `20` and `in_bin` to **`21`**.

## Special-case: Event Listeners

One last trick with polymorphic variants:

```ocaml
type readline

external on:
  readline
  -> ([
      |`close of unit -> unit
      | `line of string -> unit
      ] [@bs.string])
  -> readline = "" [@@bs.send]

let register rl =
  rl
  |. on (`close (fun event -> ()))
  |. on (`line (fun line -> print_endline line))
```

```reason
type readline;

[@bs.send]
external on: (
    readline,
    [@bs.string] [ | `close(unit => unit) | `line(string => unit)]
  )
  => readline = "";

let register = rl =>
  rl
  ->on(`close(event => ()))
  ->on(`line(line => print_endline(line)));
```

Output:

```js
function register(rl) {
  return rl.on("close", (function () {
              return /* () */0;
            }))
            .on("line", (function (line) {
              console.log(line);
              return /* () */0;
            }));
}
```

<!-- TODO: GADT phantom type -->

## Fixed Arguments

Sometimes it is convenient to bind to a function using an `external` while passing predetermined argument values to the JS function:

```ocaml
external process_on_exit:
  (_ [@bs.as "exit"]) ->
  (int -> unit) ->
  unit =
  "process.on" [@@bs.val]

let () = process_on_exit (fun exit_code ->
  Js.log ("error code: " ^ string_of_int exit_code)
)
```

```reason
[@bs.val]
external process_on_exit: (
  [@bs.as "exit"] _,
  int => unit
) => unit = "process.on";

let () = process_on_exit(exit_code =>
  Js.log("error code: " ++ string_of_int(exit_code))
);
```

Output:

```js
process.on("exit", function (exit_code) {
  console.log("error code: " + exit_code);
  return /* () */0;
});
```

The `[@bs.as "exit"]` and the placeholder `_` argument together indicates that you want the first argument to compile to the string `"exit"`. You can also use any JSON literal with `bs.as`: `[@bs.as {json|true|json}]`, `[@bs.as {json|{"name": "John"}|json}]`, etc.

## Curry & Uncurry

Curry is a delicious Indian dish. More importantly, in the context of BuckleScript (and functional programming in general), currying means that a function taking multiple arguments can be applied a few arguments at time, until all the arguments are applied.

```ocaml
let add x y z = x + y + z
let addFive = add 5
let twelve = addFive 3 4
```

```reason
let add = (x, y, z) => x + y + z;
let addFive = add(5);
let twelve = addFive(3, 4);
```

Notice the `addFive` intermediate function. `add` takes in 3 arguments but received only 1. It is interpreted as "currying" the argument `5` and waiting for the next 2 arguments to be applied later on. The corresponding type signatures are:

```ocaml
val add: int -> int -> int -> int
val addFive: int -> int -> int
val twelve: int
```

```reason
let add: (int, int, int) => int;
let addFive: (int, int) => int;
let twelve: int;
```

(In a dynamic language, such as JS, currying would be dangerous since accidentally forgetting to pass an argument does not create an error at compile time.)

### Drawback

Unfortunately, due to JS not having currying because of the aforementioned reason, it is hard for BS multi-argument functions to map cleanly to JS functions 100% of the time; therefore:

1. When all the arguments of a function are supplied (i.e., no currying), BS does its best to compile, for example, a 3-argument call into a plain JS call with 3 arguments.

2. If it is too hard to detect whether a function application is complete\*, BS will use a runtime mechanism (the `Curry` module) to curry as many arguments as we can and check whether the result is fully applied.

3. Some JS APIs (like `throttle`, `debounce` and `promise`) might mess with context, i.e., use the functions `bind` mechanism, carry around `this`, etc. Such implementations clash with the previous currying logic.

\* If the call site is typed as having 3 arguments, we sometimes don't know whether it is a function that is being curried, or if the original one indeed has only 3 arguments.

BS tries to do #1 as much as it can. Even when it bails and uses #2's currying mechanism, it is usually harmless.

**However**, if you encounter #3, heuristics are not good enough: you need a guaranteed way of fully applying a function without intermediate currying steps. We provide such a guarantee through the use of the `[@bs]` "uncurrying" annotation on a function declaration and call site.

### Solution: Guaranteed Uncurrying

If you annotate a function declaration signature on an `external` or `let` with a `[@bs]` (or, in Reason syntax, annotate the start of the parameters with a dot), you will turn that function into an similar-looking one that is guaranteed to be uncurried:

```ocaml
type timerId
external setTimeout: (unit -> unit [@bs]) -> int -> timerId = "setTimeout" [@@bs.val]

let id = setTimeout (fun [@bs] () -> Js.log "hello") 1000
```

```reason
type timerId;
[@bs.val] external setTimeout: ((. unit) => unit, int) => timerId = "setTimeout";

let id = setTimeout((.) => Js.log("hello"), 1000);
```

**Note**: both the declaration site and the call site need to have the uncurry annotation. That is part of the guarantee/requirement.

When you try to curry such a function, you will get a type error:

```ocaml
let add = fun [@bs] x y z -> x + y + z
let addFiveOops = add 5
```

```reason
let add = (. x, y, z) => x + y + z;
let addFiveOops = add(5);
```

Error:

```
This is an uncurried bucklescript function. It must be applied with [@bs].
```

#### Alternative Solution

The above solution is safe, guaranteed, and performant, but sometimes visually a little burdensome. We provide an alternative solution if:

- you are using `external`,
- the `external` function takes in an argument that is another function, and
- you want the user to **not** need to annotate the call sites with `[@bs]` or the dot in Reason.

<!-- TODO: is this up-to-date info? -->

Then try `[@bs.uncurry]`:

```ocaml
external map: 'a array -> ('a -> 'b [@bs.uncurry]) -> 'b array = "" [@@bs.send]
let _ = map [|1; 2; 3|] (fun x -> x+ 1)
```

```reason
[@bs.send] external map: (array('a), [@bs.uncurry] ('a => 'b)) => array('b) = "";
map([|1, 2, 3|], x => x + 1);
```

#### Pitfall

If you try to do this:

```ocaml
let id: ('a -> 'a [@bs]) = ((fun v -> v) [@bs])
```

```reason
let id: (. 'a) => 'a = (. v) => v;
```

You will get this cryptic error message:

```
Error: The type of this expression, ('_a -> '_a [@bs]),
       contains type variables that cannot be generalized
```

The issue here is not that the function is polymorphic. You can use polymorphic uncurried functions as inline callbacks, but you cannot export them (and `let`s are exposed by default unless you hide them with an interface file). The issue here is a combination of the uncurried call, polymorphism and exporting the function. It is an unfortunate limitation of how OCaml’s type system incorporates side-effects, and how BS handles uncurrying.

The simplest solution is in most cases is to just not export it by adding an interface to the module. Alternatively, if you really need to export it, you can do so in its curried form, and then wrap it in an uncurried lambda at the call site. E.g.:

```ocaml
let _ = map (fun v -> id v [@bs])
```

```reason
map(v => id(. v));
```

##### Design Decisions

In general, `bs.uncurry` is recommended; the compiler will do lots of optimizations to resolve the currying to uncurrying at compile time. However, there are some cases the compiler cannot optimize, so these cases will be converted to a runtime check.

This means `[@bs]` is a completely static behavior (no runtime cost), while `[@bs.uncurry]` is more convenient for end users (in some rare cases it might also be slower than `[@bs]`).

## Modeling `this`-based Callbacks

Many JS libraries have callbacks which rely on `this` (the source), for example:

```js
x.onload = function(v) {
  console.log(this.response + v)
}
```

Here, `this` would point to `x` (actually, it depends on how `onload` is called, but we digress). It is not correct to declare `x.onload` of type `unit → unit [@bs]`. Instead, we introduced a special attribute, `bs.this`, which allows us to type `x` as so:

```ocaml
type x
external x: x = "" [@@bs.val]
external set_onload: x -> (x -> int -> unit [@bs.this]) -> unit = "onload" [@@bs.set]
external resp: x -> int = "response" [@@bs.get]

let _ =
  set_onload x begin fun [@bs.this] o v ->
    Js.log(resp o + v )
  end
```

```reason
type x;
[@bs.val] external x: x = "";
[@bs.set] external set_onload: (x, [@bs.this] ((x, int) => unit)) => unit = "onload";
[@bs.get] external resp: x => int = "response";

set_onload(x, [@bs.this] ((o, v) => Js.log(resp(o) + v)));
```

Output:

```js
x.onload = (function (v) {
    var o = this;
    console.log(o.response + v | 0);
    return /* () */0;
  });
```

`bs.this` is the same as `bs`, except that its first parameter is reserved for `this` and for an arity of 0 - there is no need for a redundant `unit` type.
