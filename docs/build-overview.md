---
title: Overview
---

BuckleScript comes with a build system, `bsb`, that is meant to be fast, lean and used as the build system of the BuckleScript/Reason community.

`bsb` provides a few templates to quickly start a new project:

```sh
bsb -init my-directory-name
```

To generate a Reason project:

```sh
bsb -init my-directory-name -theme basic-reason
```

Feel free to inspect the various files in the newly generated directory. To see all the templates available, do:

```sh
bsb -themes
```

<!-- TODO: clean up themes -->

The build description file is called `bsconfig.json`. Every BuckleScript project needs one.

**To build a project**, run:

```sh
bsb -make-world
```

Add `-w` to keep the built-in watcher running. Any new file change will be picked up and the build will re-run.

**Note**: third-party libraries (in `node_modules`) are not watched because doing so may exceed the Node.js watcher count limit. If you are doing quick and dirty modifications inside `node_modules`, you have to do `bsb -clean-world -make-world` to rebuild them.

**To build only yourself**, use `bsb -make`.

`bsb -help` to see all the available options.

## Artifacts Cleaning

If you ever get into a stable build for edge-case reasons, use:

```sh
bsb -clean-world
```

Or `bsb -clean` to clean only your own artifacts.

## Editor Support

`bsb` generates a `.merlin` file, which is used by various [editor plugins](https://reasonml.github.io/docs/en/editor-plugins.html) to power things like autocomplete, type hinting, diagnosis, etc.

**Bonus**: you can directly pipe the `bsb` terminal error messages into VSCode by setting the config [here](https://github.com/reasonml-editor/vscode-reasonml#bsb).

### Tips & Tricks

A typical problem with traditional build systems is that they are not resilient against the user moving/deleting source files, and most do not clean up the old artifacts correctly after such a user action\*. `bsb` is, unfortunately, no different **unless** you turn on `"suffix": ".bs.js"` in `bsconfig.json`. Once turned on, we can correctly track which JS artifact belongs to which source file, even after source file moving/deletion.

## Design Decisions

\* One such build system that tracks these correctly and efficiently is [Tup](http://gittup.org/tup/). See the (rather accessible!) paper [here](http://gittup.org/tup/build_system_rules_and_algorithms.pdf). Unfortunately, Tup's implementation uses FUSE, and other systems, which we cannot safely use on every platform.
