---
title: Null, Undefined & Option
---

BuckleScript itself does not have the notion of `null` or `undefined`. This is a _great_ thing, as it wipes out an entire category of bugs. No more `undefined is not a function`, and `cannot access foo of undefined`!

However, the **concept** of a potentially nonexistent value is still useful, and safely exists in our language.

We represent the existence and nonexistence of a value by wrapping it with the `option` type. Here is its definition from the standard library:

```ocaml
type 'a option = None | Some 'a
```

```reason
type option('a) = None | Some('a)
```

It means "a value of type option is either None (nothing) or that actual value wrapped in a Some". You might wonder why we cannot drop that `Some` wrapper and just do the following:

```ocaml
type 'a option = None | 'a
```

```reason
type option('a) = None | 'a
```

So that the value can be either `None` or `theActualValue`. While convenient, there are many reasons why this is undesirable later on. We will spare you the explanation here.

## Example

Here is a normal value:

```ocaml
let licenseNumber = 5
```

```reason
let licenseNumber = 5
```

To represent the concept of "maybe null", you would turn this into an `option` type by wrapping it. For the sake of a more illustrative example, we will put a condition around it:

```ocaml
let licenseNumber =
  if personHasACar then
    Some(5)
  else
    None
```

```reason
let licenseNumber =
  if (personHasACar) {
    Some(5);
  } else {
    None;
  };
```

Later on, when another piece of code receives such value, it will be forced to handle both cases:

```ocaml
match licenseNumber with
| None -> print_endline "The person doesn't have a car"
| Some(number) -> print_endline ("The person's license number is " ^ (string_of_int number))
```

```reason
switch (licenseNumber) {
| None => print_endline("The person doesn't have a car")
| Some(number) =>
  print_endline("The person's license number is " ++ string_of_int(number))
};
```

By turning your ordinary number into an `option` type, and by forcing you to handle the `None` case, the language effectively removed the possibility for you to mishandle, or forget to handle, a conceptual `null` value!

## Usage

**The `Option` helper module** is [here](https://bucklescript.github.io/bucklescript/api/Belt.Option.html).

## Interoperate with JavaScripts `undefined` and `null`

The `option` type is common enough that we special-case it when compiling to JavaScript:

```ocaml
Some 5
```

```reason
Some(5)
```

simply compiles down to `5`, and

```ocaml
None
```

```reason
None
```

compiles to `undefined`! If you have got a string, for example, in JavaScript that you know might be `undefined`, type it as `option(string)` and you're done! Likewise, you can send a `Some(5)` or `None` to the JS side and expect it to be interpreted correctly =)

### Caveat 1

The option-to-undefined translation is not perfect because, on the BS side, `option` values can be composed:

```ocaml
Some (Some (Some 5))
```

```reason
Some(Some(Some(5)))
```

This still compiles to `5`. However, the following gets troublesome:

```ocaml
Some None
```

```reason
Some(None)
```

This is compiled into the following JS:

```js
Js_primitive.some(undefined);
```

What is this `Js_primitive` thing? Why can't this compile to `undefined`? Long story short, when dealing with the polymorphic `option` type (i.e., `option('a)`, for any `'a`), many operations become tricky if we do not mark the value with some special annotation. If this does not make sense, do not worry; just remember the following rule:

- **Never, EVER, pass a nested `option` value (e.g. `Some(Some(Some(5)))`) into the JS side.**
- **Never, EVER, annotate a value coming from JS as `option('a)`. Always give the concrete, non-polymorphic type.**

### Caveat 2

Unfortunately, lots of times, your JavaScript value might be _both_ `null` or `undefined`. Unfortunately, in that case, you cannot type such value as `option(int)`, for example, since our `option` type only checks for `undefined` and not `null` when dealing with a `None`.

#### Solution: More Sophisticated `undefined` & `null` Interop

To solve this, we provide access to more elaborate `null` and `undefined` helpers through the [`Js.Nullable`](https://bucklescript.github.io/bucklescript/api/Js.Nullable.html) module. This somewhat works like an `option` type, but is different from it.

#### Examples

To create a JS `null`, use the value `Js.Nullable.null`. To create a JS `undefined`, use `Js.Nullable.undefined`. (You can naturally use `None` too, but that's not the point here; the `Js.Nullable.*` helpers wouldn't work with it.)

If you are receiving, for example, a JS string that can be `null` and `undefined`, type it as:

```ocaml
external myId: string Js.Nullable.t = "myId" [@@bs.module "MyConstant"]
```

```reason
[@bs.module "MyConstant"] external myId: Js.Nullable.t(string) = "myId"
```

To create such a nullable string from our side (presumably to pass it to the JS side for interop purposes), do:

```ocaml
external validate: string Js.Nullable.t -> bool = "validate" [@@bs.module "MyIdValidator"]
let personId: string Js.Nullable.t = Js.Nullable.return "abc123"

let result = validate personId
```

```reason
[@bs.module "MyIdValidator"] external validate: Js.Nullable.t(string) => bool = "validate";
let personId: Js.Nullable.t(string) = Js.Nullable.return("abc123");

let result = validate(personId);
```

The `return` part "wraps" a string into a nullable string in order to make the type system understand and track the fact that, as you pass this value around, it is not just a string, but a string that can be `null` or `undefined`.

#### Convert to/from `option`

`Js.Nullable.fromOption` converts from a `option` to `Js.Nullable.t`. `Js.Nullable.toOption` does the opposite.
