---
title: Performance
---

BuckleScript considers performance at install time, build time and run time as a serious feature. Here is some more information and tips on keeping the build fast. **Feel free to skip this section** if you are just starting out.

## Under the Hood

`bsb` itself uses a build system under the hood, which is called [Ninja](https://ninja-build.org). Ninja is like Make, but it is cross-platform, minimal, focuses on performance and is destined to be more of a low-level building block than a full-blown build system. In this regard, Ninja is a great implementation detail for `bsb`.

`bsb` reads into `bsconfig.json` and generates the Ninja build file in `lib/bs`. The file contains the low-level `bsc`-related commands, namespacing rules, intermediate artifacts generation and others. It then runs `ninja` for the actual build.

## The JS Wrapper

`bsb` is actually a Node.js wrapper which takes care of some miscellaneous tasks, plus the watcher. The lower-level, watcher-less, true `bsb` is called `bsb.exe`. It can be found in the same directory as `bsb`:

```sh
> bsb -where
/usr/local/lib/node_modules/bs-platform/lib
```

The path varies across systems.

If you don't need the watcher, you can run said `bsb.exe`: `/usr/local/lib/node_modules/bs-platform/lib/bsb.exe`. This side-steps the Node.js startup time, which can be big (in the order of 100 ms).

## Numbers

A raw `bsb.exe` build time on a small project should be around 70 ms. This doubles when you use the more common `bsb` wrapper, which comes with a watcher. Practically, `bsb` is faster since you do not manually run the build at every change (although you should opt for the raw `bsb.exe` for programmatic usage, e.g. inserting `bsb` into your existing JS build pipeline).

No-op builds (when no file has changed) should be around 15 ms . Incremental rebuilds (described below) of a single file in a project is around 70 ms.

Cleaning the artifacts should be instantaneous.

### Extreme Test

We have stress tested `bsb` on a big project of 10,000 files (2 directories, 5000 files each; the first 5000 files have no dependencies, the last 5000 files have 10 dependencies on files from the former directory) using https://github.com/ocaml-omake/omake/blob/perf-test/performance/generate.ml, on a Retina Macbook Pro Early 2015 (3.1 GHz Intel Core i7).

<!-- TODO: better repro -->

- No-op build of 10k files: 800 ms (the minimum amount of time required to check the mtimes of 10k files).
- Clean build: less than 3 minutes.
- Incremental build: depends on the number of the dependents of the file. No dependents means 1 second.

Note that `bsb` is a file-based build system. We do not do in-memory builds, even if that speeds up the build a lot. In-memory builds risk memory leaks, out-of-memory errors and others. The `bsb` watcher, on the other hand, can stay open for days.

## Incrementality

`bsb` does not take whole seconds to run every time. The bulk of the build performance comes from an incremental build, i.e., re-building a previously built project when a few files have changed.

In short, thanks to OCaml, BuckleScript and `bsb`'s architecture, we are able to **only build what is needed**. For example, if `MyFile.ml` is not changed, then it is not recompiled. You can roughly emulate such incrementality in languages like JavaScript, but the degree of correctness is unfortunately low. For example, if you rename or move a JS file, then the watcher might get confused and not pick up the "new" file or fail to clean things up correctly, resulting in you needing to clean your build and restart anew, which defeats the purpose.

## Speeding Up Incremental Builds

OCaml/BuckleScript/Reason uses the concept of interface files (`.mli` or `.rei`) (or, equivalently, [module signatures](https://reasonml.github.io/docs/en/module.html#signatures)). Exposing only what you need naturally speeds up incremental builds. For example, if you change a `.ml` or `.re` file whose corresponding `.mli` or `.rei` file does not expose the changed part, then you have reduced the amount of dependent files you have to rebuild.

<!-- TODO: validate this -->

## Programmatic Usage

Unfortunately, JS build systems are usually the bottleneck for building a JS project nowadays. Having part of the build finish blazingly fast does not matter much if the rest of the build takes seconds or literally minutes. Here are a few suggestions:

- Convert more files into BuckleScript/Reason =). Fewer files going through fewer parts of the JS pipeline helps a ton.
- Be careful with bringing in more dependencies: libraries, syntax transforms, build step loaders, etc. The bulk of these drag down the editing and building experience, which might out-weigh the API benefits they provide.
- Wait for us to create our own super fast linker/bundler.

## Hot Reloading

Hot reloading refers to maintaining a dev server and listening to file changes in a way that allows the server to pipe some delta changes right into the currently running browser page. This provides a relatively fast iteration workflow while working in specific frameworks.

However, hot reloading is fragile by nature, and counts on the occasional inconsistencies (bad state, bad evaluation, etc.) and the heavy dev server setup/config being less of a hassle than the benefits it provides. We err on the side of caution and stability in general, and decided not to provide built-in hot reloading _yet_. **Note**: you can still use the hot reloading facility provided by your JS build pipeline.
