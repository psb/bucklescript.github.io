---
title: Interop with JS Build System
---

If you come from JS, chances are that you already have a build system in your existing project. Here is an overview of the role `bsb` would play in your build pipeline, if you want to introduce some BS code into the codebase.

## Popular JS Build Systems

The JS ecosystem uses a few build systems: [browserify](http://browserify.org/), [rollup](https://github.com/rollup/rollup), [webpack](https://webpack.js.org/), etc. The latter's probably the most popular of the three (as of December 2017 =P). These build systems do both the compilation and the linking (i.e., bundling many files into one or few files).

BuckleScript and `bsb` only take care of the compilation step; it maps one ml/re/mli/rei file into one JS output file. As such, in theory, no build system integration is needed from our side. From the webpack watcher's perspective, for example, the JS files BuckleScript generates are almost equivalent to your hand-written JS files. We also recommend that you initially check in those BS-generated JS files, as this workflow means:

- You can introduce BuckleScript and/or Reason silently into your codebase without disturbing existing build infrastructure.
- You have a visual diff of the performance and correctness of your JS file when you update the ml/re files and the JS artifacts change.
- You can let teammates hot-patch the JS files in emergency situations without needing to first start learning BS/Reason.
- You can remove BuckleScript and Reason completely from your codebase and things will still work (if your company decides to stop using us for whatever reason).

The slight disadvantage of such an approach is that you need to keep both a `bsb` watcher and a webpack watcher open. However, if you activate [VSCode-Reasonml's `bsb` feature](https://github.com/reasonml-editor/vscode-reasonml#bsb), you avoid the former rather seamlessly.

For what it is worth, you can also turn `bsb` into an automated step in your build pipeline (e.g. into a webpack loader, like our [deprecated bs-loader here](https://github.com/reasonml-community/bs-loader)).

### Tips & Tricks

You can make BuckleScript JS files look even more idiomatic through the 'in-source' and 'suffix' configuration in `bsconfig.json`:

```json
{
  "package-specs": {
    "module": "commonjs", // or whatever module system your project uses
    "in-source": true
  },
  "suffix": ".bs.js"
}
```

This will:

- Generate the JS files alongside your BS source files.
- Use the file extension `.bs.js`, so that you can require these files on the JS side through `require('./MyFile.bs')`, without needing a loader.

## Use Loaders on BS Side

"What if my build system uses a CSS/png/whatever loader and I'd like to use it in BuckleScript?"

Loaders are indeed troublesome; in the meantime, please use `[%raw {|require('./myStyles.css')|}]`, for example, at the top of your file. This just uses [`raw`](embed-raw-javascript.md) to compile the snippet into an actual JS require.
