---
title: Configuration
---

`bsconfig.json` is the single, mandatory build meta file needed for `bsb`.

**The complete configuration schema is [here](https://bucklescript.github.io/bucklescript/docson/#build-schema.json)**. We will _non-exhaustively_ highlight the important parts in prose here.

## name, namespace

`name` is the name of the library and is used as its "namespace". You can activate namespacing through `"namespace": true` in your `bsconfig.json`. Namespacing is almost **mandatory**; however, in order to preserve backward-compatibility, we have not turned it on by default yet.

**Explanation**: by default, your files, once used as a third-party dependency, are available globally to the consumer. For example, if you have an `Util.re` and the consumer also has a file of the same name, they will clash. Turning on `namespace` avoids this by wrapping all your own project's files into an extra module layer - instead of a global `Util` module, the consumer will see you as `MyProject.Util`. **The namespacing affects your consumers, not yourself**.

In `bsb`, "namespace" is just a fancy term for an auto-generated module that wraps all your project's files (efficiently and correctly, of course!) for third-party consumption.

We do not do folder-level namespacing for your own project; all your own file names must be unique. This is a constraint that enables several features such as fast search and easier project reorganization.

**Note**: the `bsconfig.json` `name` should be the same as the `package.json` `name` to avoid confusing corner-cases. However, this means that you cannot use a camelCased names such as `MyProject`, since `package.json` and npm forbid you to do so (some file systems are case-insensitive). To have the namespace/module as `MyProject`, write `"name": "my-project"`. `bsb` will turn that into the camelCased name correctly.

## sources

Your source files need to be specified explicitly (we don't want to accidentally drill down into some unrelated directories). Examples:

```json
{
  "sources": ["src", "examples"]
}
```
```json
{
  "sources": {
    "dir": "src",
    "subdirs": ["page"]
  }
}
```

```json
{
  "sources": [
    "examples",
    {
      "dir": "src",
      "subdirs": true // recursively builds every subdirectory
    }
  ]
}
```

You can mark your directories as dev-only (e.g. tests). These will not be built and exposed to third-parties, or even to other "dev" directories in the same project:

```json
{
  "sources" : {
    "dir" : "test",
    "type" : "dev"
  }
}
```

## bs-dependencies, bs-dev-dependencies

These are lists of BuckleScript/Reason dependencies. Just like `package.json`'s dependencies, they will be searched for in `node_modules`.

## reason, refmt

`reason` config is enabled by default. To turn on JSX for [ReasonReact](https://reasonml.github.io/reason-react/), specify:

```json
{
  "reason": {"react-jsx": 2},
  "refmt": 3
}
```

The `refmt` config **should be explicitly specified** as `3`, as in the [Reason V3 syntax](https://reasonml.github.io/blog/2017/10/27/reason3.html).

## js-post-build

This is a hook that is invoked every time a file is recompiled. It is good for JS build system interop, but please use it **sparingly**. Calling your custom command for every recompiled file slows down your build and worsens the building experience for even third-party users of your library.

Example:

```json
{
  "js-post-build": {
    "cmd": "/path/to/node ../../postProcessTheFile.js"
  }
}
```

Note that the command's path resolution is done through the following:

- `/myCommand` is resolved into `/myCommand`
- `myCommand/` is resolved into `node_modules/myCommand`
- `./myCommand` is resolved into `myProjectRoot/myCommand`
- `myCommand` is just called as `myCommand`. But note that `bsb` does not read into your environment, so if you put `node` it will not find it unless you specify an absolute path. Alternatively, point to a script that has the header `#!/usr/local/bin/node`.

The command itself is called from inside `lib/bs`.

## package-specs

Configure the output to be either CommonJS (the default), ES6 modules or AMD. For example:

```json
{
  "package-specs": {
    "module": "commonjs",
    "in-source": true
  }
}
```

- `"module": "es6-global"` resolves `node_modules` using relative paths. This is good for development-time usage of ES6 in conjunction with browsers like Safari and Firefox that support ES6 modules today. **No more dev-time bundling**!
- `"in-source": true` generates output alongside source files. If you omit it, it will generate the artifacts into `lib/js`. The output directory is not configurable otherwise.

This configuration only applies to you when you develop the project. When the project is used as a third-party library, the consumer's own `bsconfig.json` `package-specs` overrides the configuration here.

## suffix

Use either `".js"` or `".bs.js"`. We strongly prefer the latter.

### Design Decisions

Generating JS files with the `.bs.js` suffix means that, on the JS side, you can do `const myBuckleScriptFile = require('./theFile.bs')`. The benefits are:

- It is immediately clear that we are dealing with a generated JS file here.
- It avoids clashes with a potential `theFile.js` file in the same folder.
- It avoids the need of using a build system loader for BuckleScript files. This, plus an in-source build, basically means that when you integrate a BuckleScript project into your pure JS codebase, **your build pipeline is not touched at all**.
- The `.bs.js` suffix [lets `bsb` track JS artifacts much better](build-overview.md#tips-tricks).

## warnings

You can selectively turn on or off certain warnings and/or turn them into hard errors. For example:

```json
{
  "warnings": {
    "number": "-44-102",
    "error": "+5"
  }
}
```

The exmaple above turns off warning `44` and `102` (polymorphic comparison), and turns warning `5` (partial application whose result has a function type and is ignored) into a hard error.

The warning numbers are shown in the build output when they are triggered. The complete list is [here](https://caml.inria.fr/pub/docs/manual-ocaml/comp.html#sec281). `100` and up are BuckleScript-specific.

## bsc-flags

These extra flags are passed to the underlying `bsc` call. Notably: `["-bs-super-errors"]` for turning on [cleaning error output](https://reasonml.github.io/blog/2017/08/25/way-nicer-error-messages.html). There is no need to use this flag for a Reason project because it is enabled by default.

## Environment Variables

We heavily advise against the usage of environment variables; but, for certain cases, they are justified.

### Error Output Coloring: `NINJA_ANSI_FORCED`

This is mostly for other programmatic usage of `bsb` where outputting colors is not desired.

When `NINJA_ANSI_FORCED` is set to `1`, `bsb` produces color.
When `NINJA_ANSI_FORCED` is set to `0`, `bsb` does not produce color.
When `NINJA_ANSI_FORCED` is not set, `bsb` may or may not produce color, depending on a smart detection of where it is outputted.

> Note that `bsc`, the barebone compiler, will always be passed `-color always`. See more details in [this issue](https://github.com/BuckleScript/bucklescript/issues/2984#issuecomment-410669163).
