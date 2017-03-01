#Future Improvements

> The following features need more discussion/specification, but are important.




### Future work: Controlled environment

Currently `pjc build` builds packages in the current user's environment ($PATH
and other variables are inherited from the current user's shell session). That
makes it harder to cache builds between different sandboxes and makes builds
less deterministic.

Instead we need to use the restricted environment:

```sh
export PATH=/usr/local/sbin:/usr/sbin:/usr/local/bin:/usr/bin
...

# host platform/architecture
# TODO: add to pjc spec?
export pjc__platform=...
export pjc__architecture=...

# target platform/architecture (for cross compilation)
# TODO: add to pjc spec?
export pjc__target_platform=...
export pjc__target_architecture=...
```

Note that `pjc__{platform,architecture}` is not discussed elsewhere, so
we need to specify the meaning of these.

Currently if you init two `pjc` sandboxes, both will have its own versions of
dependencies (`ocaml`, `ocamlfind`, ...) compiled from sources. We can aleviate
that duplication by having a cache of package installations in `~/.pjc/store` or
something like that.

A cache key for a package is an environment (a set of k-v pairs of env vars) in
which the package was built + cache key of all its deps.

Thus if we are going to build a package we can do a lookup in store first to see
if there's the package build with the same env and deps we are requesting.

### Future work: Package build caching inter-machine

Inter-sandbox build cache is good but we are thinking of inter-machine build
cache too! So that new users could enjoy insanely fast bootstrap.

I think it is possible to populate `~/.pjc/store` with build artifacts from some
external storage over the internet by using the same cache key as described in
the previous section.

### Future-work: Mechanism for sandbox configuration

There should be a way to configure sandbox (or maybe multiple ones at once) with
dependency overrides and custom environment variables.

Note that this differs from what proposed by `pjc` main `README`. Instead
of having a single sandbox which holds builds for different compilation
targets we define a set of sandbox for each target (and sandboxing based on
architecture is just a special case of this).

###### Config format

Configuration could be made via JavaScript in `pjc.config.js`. The example which
just uses the default environment and dependencies:

```js
module.exports = {
  _sandbox: {
    env: env,
    dependencies: dependencies,
  },
};
```

Or using `jsonnet` syntax (which will probably make porting esy impl later to
OCaml/Reason easier as we don't depend on JS runtime):

```js
{
  _sandbox: {
    env: env,
    dependencies: dependencies
  }
}
```

The configuration is effectively saying "create a sandbox in directory
`_sandbox` using the current environment and the current set of dependencies
(defined in `yarn.lock` or `npm-shrinkwrap.json`)".

###### Environment override

If we want to define multiple sandboxes one for host and one for another target
(cross compilation):

```js
{
  _sandbox: {
    env: env,
    dependencies: dependencies
  },
  _sandbox_ios: {
    env: env {
      esy__platform: darwin,
      esy__architecture: arm
    },
    dependencies: dependencies
  }
}
```

###### Dependency overrides
In this example, we use the OCaml multicore compiler in the sandbox:

```js
{
  _sandbox: {
    env: env,
    dependencies: {
      ocaml: "ocaml/ocaml-multicore"
    }
  }
}
```

We may decide to require that you specify a dependency on the specific
package that you override *to*, so that the package manager will make
its source available.

###### Cross Compilation and Sandbox Config
Ideally, `esy` is smart enough to know when it needs to recompile
a package for a different architecture than the host architecture by
looking at `buildTimeOnlyDependencies` (as opposed to using existing
artifacts in the cache for the wrong architecture). It's not clear what
exactly it should do to inform the compiler which architecture it is building for.
Perhaps creating or using an established convention of setting an environment
variable such as `pjc_architecture`, and then expecting compilers to be able to
take it into account is the best approach. If that works, then existing packages
don't need any modification to their build scripts - they don't need to be modified
to pass `--arch` to the compiler. Instead, we only need to create one modification
to the compiler itself to examine the `pjc_architecture`/`ARCH` env var.

Hopefully, that will eliminate the majority of the need to fork packages in order to
make them compatible with building for another architecture. However, that won't always
be sufficient. Some packages (hopefully very rarely) will need some change to their
build command (in `package.json`/`opam` file) in order to correctly compile for
some target architecture. If that's the case then a fork/patch is the only viable
options and we need to provide a way to use that fork - but *only* in the right
circumstances. You might think that we could use the sandbox config to override
some dependencies to use the forked package compatible with the target architecture.



```js
// This does not work!
{
  _sandbox_ios: {
    env: {
      pjc_architecture: 'armv7'
    },
    dependencies: {
      myPackage: "iOSForkOfMyPackage"
    }
  }
}
```

This is not sufficient because it might be the case that we depend on
`myPackage` as a `buildTimeOnlyDependencies`, as well as a runtime dependency.
We are building on an `x86`, but the final artifacts target `armv7`. (Suppose
`pjc_architecture` is how we specify the final *runtime* architecture).

So we may need to extend this named sandbox config feature to allow overriding
packages based on whether or not that package is being built *as* a build time
dependency vs. runtime one.

```javascript
// This does not work!
{
  _sandbox_ios: {
    env: {
      pjc_architecture: 'armv7'
    },
    runtime: {
      dependencies: {
        myPackage: "iOSForkOfMyPackage"
      }
    },
    buildTime: {
    }
  }
}
```

## Runtime Environment Variables

So far we've made the distinction between runtime and buildTime dependencies.
But so far, *all* environment variables have pertained to building of artifacts.
PJC includes a command `pjc cmd` which will run the `cmd` within something
resembling the environment that you *build* the `cwd`'s project within. But what
if the artifacts that are produced expect to have certain environment variables
set when their executables are ran. Those environment variables needn't be
related to the ones that are set when *building* those artifacts. We should likely
create a convention/config for setting up environment variables when *running*
resulting executables, and allow exporting / ejecting binaries embedded in a
boot script that sets those environment variables. It's possible that many/most
of the propagation concepts (global / exclusive / etc) still apply to *runtime*
environment variables, so perhaps in addition to `esy.exportedEnv`, we should also
support `esy.exportedRuntimeEnv`. This is especially important when building systems
that can dynamically link libraries and where those libraries' paths are configured
via environment variables.
