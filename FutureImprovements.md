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
    dependencies: dependencies {
      ocaml: "ocaml/ocaml-multicore"
    }
  }
}
```
