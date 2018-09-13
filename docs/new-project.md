---
title: New Project
---

Once you have installed BuckleScript (as described in the [previous section](installation.md)), you can generate a lightweight project template:

```sh
bsb -init my-new-project
```

Alternatively, to start a new [Reason](https://reasonml.github.io) project:

```sh
bsb -init my-new-project -theme basic-reason
```

To compile and run the project you just created:

```sh
cd my-new-project
npm run build
node src/demo.bs.js
```

And that's it! You have just built the project once and ran the JS output. Feel free to inspect the JS output, add a few files in `my-new-project/src`, etc. Hopefully we have delivered on our promise of being a lean toolchain with clean output =).

We will explain `bsb`, the build system called in the above `npm run build` command, in a later section.

## Editor Integration

We share the same editor integration as Reason. See the setup [here](https://reasonml.github.io/docs/en/global-installation.html).
