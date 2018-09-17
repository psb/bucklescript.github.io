---
title: Automatic Interface Generation
---

"Interface files" (`.mli` and `.rei` files) are the "public description" of their corresponding "implementation files" (`.ml` and `.re`, respectively). They are exposed as documentation and contain nothing but type declarations. Since a file is a module, an interface file is essentially a [module signature](https://reasonml.github.io/docs/en/module.html#signatures).

## Tips & Tricks

You do not have to explicitly write an interface file; by default, one will be inferred from the implementation file (just like how a module's type can be inferred when you hover over it) and **every binding from the file will be exported**. After you finish iterating on your project, we encourage that you:

- Explicitly add interface files to the files that are meant to be public.
- Add docblock comments on top of each binding to serve as documentation.
- Make some types abstract - simply don't expose every binding from the interface file.

Some types will have to be copy-pasted from the implementation file, which gets tedious. This is why we let you **automatically generate interface files**, after which you can tweak whatever you want.

For a file `src/MyUtils.ml`, run:

```sh
bsc lib/bs/src/MyUtils-MyProject.cmi
```

Where `MyProject` is your project's [namespace](build-configuration.md#name-namespace). If it's not enabled, it will be just `MyUtils.cmi`). `.cmi` is the OCaml/BuckleScript file that contains some [compiled type info](https://reasonml.github.io/community/faq#compiled-files).

The above command outputs a boilerplate `.mli` interface to stdout. If you are using Reason, turn it into `.rei` syntax by doing:

```sh
bsc -bs-re-out lib/bs/src/MyUtils-MyProject.cmi
```

_If you do not have `bsc` globally available, use the one provided locally in `node_modules/bs-platform/lib/bsc.exe`_.

**Note**: the generated boilerplate might contain the strings `"BS-EXTERNAL"` or `"BuckleScript External"`. This happens when you have used `@bs` externals in your implementation file. It is a temporary flaw; you need to manually turn these `"BS-EXTERNAL"` back into the right `@bs` externals for now. We will correct this in the future.
