---
title: Fast Pipe
---

BuckleScript has a special `|.` (or `->` for Reason) pipe syntax for dealing with various situations. This operator has many uses.

## Pipelining

The pipe takes the item on the left and puts it as the first argument of the item on the right, which is great for building data processing pipelines:

```ocaml
a
|. foo b
|. bar
```

```reason
a
->foo(b)
->bar
```

is equal to

```ocaml
bar(foo a b)
```

```reason
bar(foo(a, b))
```

## JS Method Chaining

JavaScript's APIs are often attached to objects and are chainable, like so:

```js
const result = [1, 2, 3].map(a => a + 1).filter(a => a % 2 === 0);

asyncRequest().setWaitDuration(4000).send();
```

Assuming we do not need the chaining behavior shown above, we could bind to each case using `bs.send`:

```ocaml
external map : 'a array -> ('a -> 'b) -> 'b array = "" [@@bs.send]
external filter : 'a array -> ('a -> 'b) -> 'b array = "" [@@bs.send]

type request
external asyncRequest: unit -> request = ""
external setWaitDuration: request -> int -> request = "" [@@bs.send]
external send: request -> unit = "" [@@bs.send]
```

```reason
[@bs.send] external map : (array('a), 'a => 'b) => array('b) = "";
[@bs.send] external filter : (array('a), 'a => 'b) => array('b) = "";

type request;
external asyncRequest: unit => request = "";
[@bs.send] external setWaitDuration: (request, int) => request = "";
[@bs.send] external send: request => unit = "";
```

You would then use them like this:

```ocaml
let result = filter (map [|1; 2; 3|] (fun a -> a + 1)) (fun a -> a mod 2 = 0)

let () = send(setWaitDuration (asyncRequest()) 4000)
```

```reason
let result = filter(map([|1, 2, 3|], a => a + 1), a => a mod 2 == 0);

send(setWaitDuration(asyncRequest(), 4000));
```

This looks much worse than the JS counterpart! Now we need to read the actual logic "inside-out". We also cannot use the `|>` operator here because the object comes _first_ in the binding. But `|.` and `->` work!

```ocaml
let result = [|1; 2; 3|]
  |. map(fun a -> a + 1)
  |. filter(fun a -> a mod 2 == 0)

let () = asyncRequest () |. setWaitDuration 400 |. send
```

```reason
let result = [|1, 2, 3|]
  ->map(a => a + 1)
  ->filter(a => a mod 2 === 0);

asyncRequest()->setWaitDuration(400)->send;
```

## Pipe Into Variants

This works:

```ocaml
let result = name |. preprocess |. Some
```

```reason
let result = name->preprocess->Some
```

We turn this into:

```ocaml
let result = Some(preprocess(name))
```

```reason
let result = Some(preprocess(name))
```
