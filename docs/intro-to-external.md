---
title: Intro to External
---

Many parts of the interop system use a concept called `external`, so we will specially introduce it here.

`external` is a keyword for declaring a value in BuckleScript/OCaml/Reason:

```ocaml
external myCFunction : int -> string = "theCFunctionName"
```

```reason
external myCFunction : int => string = "theCFunctionName";
```

It is like a `let`, except that the body of an external is, as seen above, a string. That string usually has a specific meaning depending on the context. For native OCaml, it usually points to a C function of the same name. For BuckleScript, these externals are usually decorated with certain `[@bs.blabla]` attributes.

Once declared, you can use an `external` as a normal value.

BuckleScript `external`s are turned into the expected JS values, inlined into their callers during compilation **and completely erased afterward**. You won't see them in the JS output. It's as if you wrote the generated JS code by hand! There is no performance cost either, naturally.

**Note**: do also use `external`s and the `[@bs.blabla]` attributes in the interface files; otherwise, the inlining will not happen.

## Special Identity External

One external worth mentioning is the following one:

```ocaml
external myShadyConversion : foo -> bar = "%identity"
```

```reason
external myShadyConversion : foo => bar = "%identity";
```

This is a final escape hatch which does nothing but convert from the type `foo` to `bar`. In the following sections, if you ever fail to write an `external`, you can fall back to using this one. However, try not to.
