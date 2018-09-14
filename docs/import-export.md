---
title: Import & Export
---

## Export

BuckleScript can compile to:

- CommonJS (`require('myFile')`).
- ES6 modules (`import * from 'myFile'`).
- AMD (`define(['myFile'], ...)`).

The output format is configurable in the BS build system. (The build system is called `bsb` and is described in a [later section](build-overview.md).)

By default, every `let` binding is exported. If their values are safe to use on the JS side, you can directly require the generated JS file and use them (see the JS file itself!).

To only export a few selected `let`s, simply add an [interface file](https://reasonml.github.io/docs/en/module.html#signatures) that hides some of the let bindings.

### Export an ES6 default value

If your JS project is using ES6 modules, you are likely exporting and importing some default values:

```js
// student.js
export default name = "Al";
```

```js
// teacher.js
import studentName from 'student.js';
```

Technically, since a BuckleScript file maps to a module, there is no such thing as a "default" export, only named ones. However, we have made an exception to support default in a module when you do the following:

```ocaml
(* FavoriteStudent.ml *)
let default = "Bob"
```

```reason
/* FavoriteStudent.ml */
let default = "Bob"
```

<!-- TODO: playground link on the result -->

You can then require the default value as normal on the JS side:

```js
// teacher2.js
import studentName from 'FavoriteStudent.js';
```

**Note**: the above JS snippet _only_ works if you are using ES6 import/export syntax in JS. If you are still using `require`, you will need to do:

```js
let studentName = require('FavoriteStudent').default;
```

## Import

Use `bs.module`. It is like a `bs.val` that accepts a string that is the module name or path.

```ocaml
external dirname: string -> string = "dirname" [@@bs.module "path"]
let root = dirname "/User/chenglou"
```

```reason
[@bs.module "path"] external dirname : string => string = "dirname";
let root = dirname("/User/chenglou");
```

Output:

```js
var Path = require("path");
var root = Path.dirname("/User/chenglou");
```

**Note**: the string inside `bs.module` can be anything: `"./src/myJsFile"`, `"@myNpmNamespace/myLib"`, etc.

### Import a Default Value

By omitting the payload to `bs.module`, you automatically bind to the whole JS module:

```ocaml
external leftPad: string -> int -> string = "./leftPad" [@@bs.module]
let paddedResult = leftPad "hi" 5
```

```reason
[@bs.module] external leftPad : string => int => string = "./leftPad";
let paddedResult = leftPad("hi", 5);
```

Output:

```js
var LeftPad = require("./leftPad");
var paddedResult = LeftPad("hi", 5);
```

#### Import an ES6 Default Value

This is a recurring question so we will answer it here. **If your JS project is using ES6**, it is highly likely that you are using Babel to compile it to regular JavaScript. Babel's ES6 default export actually exports the default value under the name `default`. You can bind to it like this:

```ocaml
external studentName: string = "default" [@@bs.module "./student"]
let _ = Js.log studentName
```

```reason
[@bs.module "./student"] external studentName : string = "default";
Js.log(studentName);
```

Output:

```js
var Student = require("./student");
console.log(Student.default);
```

### Tips & Tricks

When the name you are using on the BS side matches the name of the JS value, you can use the empty string shorthand:

```ocaml
external dirname: string -> string = "" [@@bs.module "path"]
```

```reason
[@bs.module "path"] external dirname : string => string = "";
```
