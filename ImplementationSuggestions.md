# Implementation Suggestions


These are merely suggestions / thoughts about how to implement some of the conventions specified in the main `README`.


## Symlinks:

We can use the `npm link` command, but that forms another dependency on `npm` which would be nice to avoid, and it might be tricky on `Windows` and various build systems (few build systems are tested on symlinks).

Instead, we can create our own workflow like `dependency-env link MyDependency /path/to/local/MyDependency` that doesn't use actual symlinks, and instead places a plain text file `./node_modules/MyDependency.link` containing the destination.  Various build systems may not even need to be concerned with this.  The main issue is that `npm postinstall` scripts don't know how to process this file and walk down to these "linked" projects to build them before our project is build. That is probably okay, because we may want to bypass the default `postinstall` process anyways so that we can maximally parallelize the builds.  Furthermore, if we completely bypass the `npm` `postinstall`, then individual package build systems won't need to understand the `.link` convention, because our custom `postinstall` process will `cd` to the actual directory of `MyDependency`, and as always set the appropriate environment variables such as `cur__target_dir` and `cur__install` back in the original top level package root.
