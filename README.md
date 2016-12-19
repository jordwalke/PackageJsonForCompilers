#PackageJsonForCompilers (`pjc`) Proposal

`PackageJsonForCompilers` is a specification for how `package.json` packages can perform
native compilation, while still enjoying what makes the `npm` workflows great.

Features:
- Cross compilation (within a project, build dependencies for your local machine and/or a mobile device that has a different architecture).
- Work well with symlinks for rapid development.
- Parallelize builds. Recover builds.
- Build awesome/interactive dashboards that explain what is going on when building projects, or changing dependencies.
- Export your entire project to a `Makefile` or shell script that can run on any network isolated machine, even if that machine does not have `node`/`npm`/`opam`/`crate` installed etc.

- `pjc` is not a build system for any particular compiler. It is a "meta build system",
that helps all your individual pacakges' build systems work in harmony together, and then
exposes an approachable *human* interface to that process.
- `pjc` is not a package manager. It only requires that package sources be located entirely on disk
before it begins working. It follows the `npm` directory structure convention, but this isn't
central to `pjc`. What *is* central is that `pjc` tollerate multiple versions of packages existing
simultaneously.
- Package managers like `yarn` could make use of `pjc` packages' `package.json` fields in order to
perform a more optimal installation (deduping more optimally).

The command name we'll use in this doc is `pjc`,
but concretely, we've started a reference implementation of `pjc` called [`esy`](https://github.com/jordwalke/esy) (as in Easy).


> Note: This has little to nothing to do with `node`/`npm` - just by convention, we assume
that sources for dependencies are placed into `node_modules`, but we could have
just as easily named that directory anything else.


## Global Installation of `pjc`

`pjc` is the *last* globally installed program you'll need. It lets you
interact with, build, debug, and explore local project sandboxes.

```
npm install -g pjc
```

## Make and Build a Project

Add `pjc` to your `package.json` project dependencies, and specify how `pjc`
should build your project in the `pjc.build` field.

```json
{
  "dependencies": {
    "pjc": "*"
  },
  "pjc": {
    "build": "yourBuildCommand"
  }
}
```

Install your dependencies, then build your project from the top level package.

```sh
npm install
pjc build
```

When you invoke `pjc build`, all the `pjc.build` commands in the dependency
graph's `package.json` files will be invoked in the right order, at the
right time, in parallel, and with an environment that automatically contains
all the right environment variables that each package needs to build harmoniously.
The majority of this doc describes the meaning of those environment variables
and how to use them to ensure your packages are good citizens.

#### Running Sandboxed Commands

`pjc build` is one of the very few "built in" commands (which builds the entire graph).
Except for the few built in commands, `pjc` can be invoked with *arbitrary* shell commands:

```
pjc which merlin
pjc your command here
```

When you prefix a shell command with `pjc`, it causes the command to "see all the right things"
from your project sandbox. The underlying philosophy of `package.json` projects, is that it's
a bad idea to just install things globally without modeling the fact that your projects depend
on those thing. Furthermore, multiple projects could depend on two different conflicting versions
of a tool, so installing those globally are out of the question. Anytime you make a `package.json`,
you're *defining* a sandbox that correctly models these dependencies and avoids many of those issues.

Prefixing commands like `pjc yourcommand here` allows `yourcommand here` to see the dependencies
that are modeled in `package.json`. That means the command will be able to see a `PATH` augmented
with binaries that your dependencies build, and much much more.

```sh
# Returns empty
which myBuildArtifact.exe

# Returns the path in sandbox successfully 
pjc which myBuildArtifact.exe
```



> When invoked, `pjc` will always search to see if you're in a project directory
that has `pjc` installed, and if so, it will forward all your requests to that
version, otherwise it will use the globally installed version of `pjc`. (This is a minor details,
not worth remembering).



As mentioned, there are a few builtin commands which take precedence over the commands you
supply after `pjc`.

|Built-in Command| Action                                                                     |
|----------------|----------------------------------------------------------------------------|
| `pjc build`    | Build the project in parallel, from the top level package                  |
| `pjc shell`    | Source all environment variables for your sandbox, after `deshell`ing any previously active `pjc shell`|
| `pjc deshell`  | Restore the environment to the way it was before the most recent `pjc shell` |


#### Exporting Environment Variables

The `pjc.env` field allows you to set environment variables that your sandboxed
commands can see, and that your dependers can see.

```json
{
  "dependencies": {
    "pjc": "*"
  },
  "pjc": {
    "build": "yourBuildCommand",
    "envVars": {
      "USER_NAME": "Jack"
    }
  }
}
```

### Further Configuration/Usage:

1. Mark the names of dependencies that are *build time only* in a
`package.json` field called `buildTimeOnlyDependencies`.

2. Enjoy some benefits of `pjc`.
  - `pjc build` traverses the dependency graph and automatically runs each `package.json`'s
  build scripts with a perfectly constructed environment. Each package build will see the right
environment variables. Among others, it will see a `PATH` augmented with binaries built by its dependencies,
along with any other variables its depenencies want it to see.
  - `pjc` also automatically sets environment variables that allow your
build scripts to see `ocamlfind` packages/libraries generated by your immediate `package.json`
dependencies.
  - `pjc` also automatically prepares an `ocamlfind` directory structure `lib/bin/doc` for
  each depdendency to install itself into. `pjc` tells you where this is located by setting
  another special environment variable. (It also generates an `ocamlfind.conf`).
  - `pjc` also sets up a bunch of other helpful environment variables that your build scripts
can see, most of them taken directly from `opam`.

3. Respect the `my_package__target_dir` environment variable in your build scripts.
Your build system should try to only generate artifacts inside of
`my_package__target_dir`. This makes sure your package build works with symlinks, and
multiple simultaneous versions of your package.

4. Respect the `my_package__install` environment variable in your build scripts: So
far your package build can "see" its dependencies at build time via the
automatically created `findlib.conf` / environment variables. That's good but it
doesn't mean *your* package's artifacts will be visible to *other* packages that depend
on *you*! To make your package visible to your dependers via `ocamlfind`, your build system should generate a
`META` file, and *install* it (along with a subset of the artifacts) into
`my_package__install` (`ocamlfind install -destdir $my_package__install
$my_package__name META`). No one wants to mess with yet another config format (`META`)
but there really does have to be *some* format (continue reading to understand why)
to record which artifacts are part of your library.

#### More details about `ocamlfind`.

> Feel free to skip this section

`ocamlfind` is an existing technology that at first seems totally redundant with
`package.json` but serves a different purpose. `ocamlfind` lets you declare `ocamlfind libraries` and
group them into `ocamlfind packages`. Whereas `package.json` packages group together source code
resources under a logical name, `ocamlfind` groups together compiled artifacts for those resources
into a logical name (the `ocamlfind` package).
Usually for every `package.json` package, you'll generate one named `ocamlfind package`,
that defines one `ocamlfind library`.

In addition to grouping together the artifacts under a logical name, these `ocamlfind`
packages track which flags must be passed to the final linking stage of the compiler,
in a way that makes it much easier on everyone's build scripts. The end result is that
if your `package.json` package can generate an `ocamlfind` library, then other `package.json`
projects can easily compile against it. Otherwise, when other packages depend on your package,
they'd be scratching their heads wondering where your build artifacts are located, and
what flags they need to pass to the compiler when compiling against them. I mean,
you can take that approach if you really want to, but why not just
make an `ocamlfind` package instead?
Specifically, when another package depends on my `package.json` package named `myPackageName`,
and when my `package.json` package generates an `ocamlfind` package called `myPackageNameOcamlFind`,
then `findlib`/`ocamlfind` allows the package that depends on my package to just pass `-pkg myPackageNameOcamlFind`.

The file that records which artifacts are grouped into `myPackageName`, is
the `META` file. By generating the `META` file, you've just created a valid `ocamlfind` package.
Now, `pjc` makes it so you will automatically see all those `META` files for your dependencies.
`pjc` supplies an environment variable that helps
build scripts for each `package.json` package know where to install their `findlib`
"package" into, and also populates another environment variable that helps your
`package.json` build script *see* all your package's `ocamlfind` dependencies.

It's super annoying to write the `META` file, so if you're making a new build
system that works well with `PackageJsonForCompilers`, you should just have your
build system just generate it. More details on that later.

Why do we need this? Again, the answer is because it's too difficult for packages to
promise to put their artifacts into predictable locations, so we suggest that they
generate `ocamlfind` packages which say *exactly* where those artifacts are.

#### More details about `pkg-config`.

`pkg-config` is like `ocamlfind`, but for C programs.

## Details

This document describes a proposal for laying out directories for sources, artifacts, as well as some utilities that we plan to provide to compilers/build systems in order for them to work seamlessly within this directory convention. It should ideally create some shared conventions that `ocamlopt`/`ocamlc`/`BuckleScript`/`opam`, and possibly even `clang` packages can abide by in order to seamlessly depend on each other - even in the face of multiple versions, or multiple targetted architectures. Build systems do not have to generate or even be aware of this directory structure, they merely need to use a couple of utilities in order required to work well within it.

General features required:
- Allows multiple versions of packages to coexist (it's important).
- Builds out of source, installs out of source.
  - That enables symlinking of packages (important).
- Plays nice with `ocamlfind`.
- Allows cleaning up of build artifacts trivially, merely by deleting a single `_build` directory.
- Supports `opam` variables and attempts to make it feasible to build an adapter that allows seamless OPAM interop.

Desirable Outcome:
One click depending on `opam` files withing a cross compiled environment - all build systems should do the right thing, and put artifacts in the right place inside of the sandbox, with a single `yarn install` or `npm install`.

In general, we have a top level package, with dependencies inside of `node_modules`, and two directories `_build` and `_install` which each mirror the entire sandbox.
    
    Project Directory
    ========================

    /path/to/MyApp/
     │                       Findlib Installations 
     ├── _install/         ----------------------------------------
     │   └── node_modules/    _install dir mirrors project directory,
     │       └── ...          contains findlib installations for everything.
     │
     ├── _build/             Build Artifacts
     │   ├── src/            ----------------------------------------
     │   │   └── ...         _build dir mirrors entire tree.
     │   └── node_modules/   Contains build artifacts for everything.
     │       └── ...
     │
     ├── package.json        Original Source Files                              
     ├── src/                ----------------------------------------           
     │   ├── files.ml        Your typical source directory with node_modules    
     │   └── ...             containing all dependencies potentially symlinked. 
     │
     └── node_modules/
         └── ...

Expanding out a bit, we see that inside of the `_install`, each package resides
in its mirrored location. Each package has its *own* `findlib` root, with its
own `bin`/`lib` etc. This is very important as it allows multiple versions of a
single package to exist simultaneously.


    Project Directory
    ========================

    /path/to/MyApp/
     │                           Findlib Installations 
     ├─ _install/              ----------------------------------------
     │  ├─ bin/               ╮  First, you have the findlib installation
     │  ├─ lib/               ╯  for the top level package
     │  └─ node_modules/   
     │     ├─ CommonUtil      ╮                    
     │     │  ├─ bin/         ┊  Then transitive dependencies' findlib installations.
     │     │  └─ lib/         ┊
     │     └─ ...             ╯                    
     │     
     │                           Build Artifacts
     ├─ _build/                  ----------------------------------------
     │  │                     ╮  Autogenerated findlib.conf describing the location
     │  ├─ findlib.conf       ┊  of each transitive dependency of the root package. 
     │  │                     ┊  (pointing to locations in _install)
     │  ├─ myApp.exe                                                        
     │  ├─ src/               ┊  Top level package's artifacts at locations 
     │  │  └─ ...             ┊   mirroringtheir original locations         
     │  │                     ╯                                             
     │  └─ node_modules/
     │     ├─ CommonUtil      ╮  
     │     │  ├─ findlib.conf ┊  Artifacts for all node_modules dependencies.
     │     │  └─ src/         ┊ 
     │     │      └─ ...      ┊
     │     └─ ...             ╯
     │
     │                           Original Source Files
     │                           ----------------------------------------
     ├─ package.json             Your typical source directory with node_modules
     ├─ src/                     containing all dependencies potentially symlinked.
     │  ├─ files.ml
     │  └─ ...
     └─ node_modules/
        ├─ ...              
        └─ CommonUtil/ --> /symlink/to/CommonUtil          
                                                                           


Expanding out even further:

    Project Directory
    ========================

    /path/to/MyApp/
     │                                  Findlib Installations                   
     ├─ _install/                     ----------------------------------------
     │   ├─ bin/
     │   │  └─ myApp.exe
     │   ├─ lib/
     │   │  └─ MyApp/                ╮  Because each library has its own
     │   │     ├─ META               ┊  findlib root it has to place another
     │   │     ├─ myApp.cmi          ┊  directory inside of `lib` matching its
     │   │     └─ myApp.cmo          ╯  package name. META file goes in there.
     │   │
     │   └─ node_modules/
     │      ├─ YourProject
     │      │  ├─ bin/
     │      │  └─ lib/
     │      │     └─ YourProject/
     │      │        ├─ META
     │      │        ├─ yours.cmo
     │      │        └─ util.cmo/
     │      └─ CommonUtil
     │         ├─ bin/
     │         └─ lib/
     │            └─ CommonUtil/
     │               ├─ META
     │               ├─ common.cmo
     │               └─ util.cmo/
     │                                  Build Artifacts
     ├─ _build/                         ----------------------------------------
     │  ├─ findlib.conf
     │  ├─ myApp.exe
     │  ├─ src/                      ╮                                                   
     │  │  ├─ myApp.cmo              ┊  In the _build directory, every artifacts
     │  │  ├─ myApp.cmi              ┊  mirrors locations of corresponding source
     │  │  └─ util.cmo               ╯  file - if any.
     │  │
     │  └─ node_modules/
     │     ├─ YourProject
     │     │  ├─ findlib.conf
     │     │  ├─ package.json
     │     │  └─ src/
     │     │     ├─ yours.cmo
     │     │     └─ util.cmo
     │     │
     │     └─ CommonUtil
     │        ├─ findlib.conf
     │        └─ src/
     │           ├─ common.cmo
     │           └─ util.cmo
     │
     │                                  Original Source Files
     │                                  ----------------------------------------
     ├─ package.json
     ├─ src/
     │  ├─ myApp.mli
     │  ├─ myApp.ml
     │  └─ util.ml
     │
     └─ node_modules/
        ├─ YourProject/
        │  ├─ package.json
        │  └─ src/
        │     ├─ yours.ml
        │     └─ util.ml
        │                         
        └─ CommonUtil/ --> /symlink/to/CommonUtil 
                              │             │
                              ╰┄┄┄┄┄┄┄┄┄┄┄┄┄╯
                            symlinks must be supported.        
                            even if they are only simulated via
                            some kind of .links file.                   



## This Is Intended For *Any* Build System.

Most of this is not specific to any particular build system, and build systems
don't need to know about this *exact* directory structure. When `pjc` builds
your package (by looking for its `pjc.build` field), `pjc`
sets up environment variables that handles all the logic of figuring out where
things should go.

## This Is Intended For *Any* Language.
The `findlib` portions of this are optional. Your package builds don't have to use them - but you may be able to adapt `findlib` to your language as well. Take it or leave it - it's not critical to `pjc`.

## Environment Variables

When `my-package` is being built, `my_package__target_dir` and
`my_package__install` are set to the artifact and install destinations for
`my-package`. This helps avoid cluttering up your source files with
artifacts, and keeps it symlink friendly / caching friendly.

|Environment Variable| Meaning                                                            | Equivalent To                        |
|--------------------|--------------------------------------------------------------------|--------------------------------------|
| `my_package__name`      | Normalized name of package name (converts hyphens to underscores)  |                                      |
| `my_package__target_dir`| Where build scripts should place artifacts for `my-package` | Cargo's `CARGO_TARGET_DIR` (loosely) |
| `my_package__install`   | Install root for `my-package`. Contains lib/bin/etc  | OPAM's `prefix` var, but is a dedicated install directory just for this package |

#### More Variables Visible To Your Package Build Scripts

|Environment Var | Meaning                                                                     | Equivalent To             |
|----------------|---------------------------------------------------------------------------- |---------------------------|
| `OCAMLFIND_CONF` | Path to precomputed findlib.conf file in `cur__target_dir`, exposing `cur`'s dependencies|              |
| `pjc__sandbox`      | Path to top level package being installed - the thing you git cloned        |                           |
| `pjc__install_tree`| Path to sandbox install tree, which contains all prefixes, for all packages |                           |
| `pjc__build_tree`  | Path to sandbox build tree, contains all build directories, for all packages|                           |
|`my_package__name`    | Normalized name of the `my-package` (`my_package`) | OPAM's `PKG:name` but not transitive    | No          |
|`my_package__version` | Version of the `my-package`                 | OPAM's `PKG:version` but not transitive | No          |
|`my_package__root`    | Path to root of source tree for `my-package`| Opam's `lib`                |
|`my_package__depends` | Resolved direct dependencies of `my-package`| OPAM's `PKG:depends`                    |             |
|`my_package__bin`     | Binary install directory for `my-package`   | OPAM's `PKG:bin` but not transitive     | No          |
|`my_package__sbin`    | System install Binary directory for `my-package`| OPAM's `PKG:sbin` but not transitive| No          |
|`my_package__lib`     | Library install directory for `my-package`  | OPAM's `PKG:lib` but not transitive     | No          |
|`my_package__man`     | Man install directory for `my-package`      | OPAM's `PKG:man` but not transitive     | No          |
|`my_package__doc`     | Docs install directory for `my-package`     | OPAM's `PKG:doc` but not transitive     | No          |
|`my_package__stublibs`| Path to `stublibs` directory in `my_package__install`  | Opam's `stublibs`           |
|`my_package__toplevel`| Path to `toplevel` directory in `my_package__install`  | Opam's `toplevel`           |
|`my_package__share`   | Share install directory for `my-package`    | OPAM's `PKG:share` but not transitive   | No          |
|`my_package__etc`     | Etc install directory for `my-package`      | OPAM's `PKG:etc` but not transitive     | No          |

#### Dependencies' Variables Visible To Your Package Build Scripts

Your package build scripts also automatically see some helpful environment
variables that describe each of your *immediate* dependencies.

For every env var above that contains `my_package`, an equivalent
env var is created for each of your package's immediate dependency
(`my_dependency__bin` for example).

In addition, a couple of environment variables are automatically augmented in
the following ways, for all dependencies.

|Environment Variable     | Your Package's Dependers See This Value As     | Equivalent To                           | Implemented |
|-------------------------|------------------------------------------------|-----------------------------------------|-------------|
|`PATH`                   | Augmented with `my_dependency__bin`            |                                         | No          |
|`MAN_PATH`               | Augmented with `my_dependency__man`           | OPAM's `PKG:name` but not transitive    | No          |

#### Variables Automatically Visible to You *and* Your Dependencies

Sometimes, you want to make additional values/paths available to your dependencies and that has been
[discussed](#Exporting Environment Variables). That allows you to specify in
*your* `package.json` which values should be visible to *dependers'* `pjc`
build scripts, and *also* your dependers' environment variable configs.


###### Opam Variables That Don't Make Sense To Recreate

|Environment Variable|
|--------------------|
| `root`             |


#### Ultra-Dynamic environment config.

Any time you see an environment variable beginning with `my_package__x` above,
there's another form `cur__x` which represents the respective value for the
*currently building package*. `cur__` is not always the same as `my_package`.

For example, suppose that `A --dependsOn--> B`, and `A` has a config file that
references `cur__root`, and that `B` *also* has a config file referencing
`cur__root`. When `B` is building, `cur__root` references `B`'s source root.
When `A` is building, `A`'s config file's `cur__root` references `A`'s source
root. But if *while* `A` is building, we reference `B`'s configuration, then
`cur__` means `A`'s root. This can happens in practice because (as you'll see),
`A`'s environment variables are set up by consulting `B`'s config.


Many of these variables have OPAM equivalents, with the exception that they are only published by *immediate* dependencies. The reason is that unlike OPAM, we support multiple versions per package in the transitive dependency graph - so some of these variables would no longer be well-defined if we allowed them to be published by transitive dependencies.

The following OPAM variables will be more difficult to implement as environment variables, but we should support them anyways.

|OPAM var        |                                                 | Difficult Because                                                                 | Implemented |
|----------------|-------------------------------------------------|-----------------------------------------------------------------------------------|-------------|
|`PKG:installed` | Whether the package is installed                | Need to scan "optionalDeps" and set false if missing                              | No          |
|`PKG:enable`    | "enable" or "disable" depending on if installed | "                                                                                 | No          |
|`PKG:build`     | Where `my-dependency` ended up being built      | We *tell* packages where to build. They might not respect that - how do we know?  | No          |


Others not yet implemented: (`PKG:hash`, `user`, `group`, `make`, `os`,
`ocaml-version`, `opam-version` `compiler`, `preinstalled`, `switch`, `jobs`,
`ocaml-native`, `ocaml-native-tool`, `ocaml-native-dynli`, `arch` )




## `buildTimeOnlyDependencies`

> Note: These have nothing to do with `devDependencies`.

Suppose `MyPackage` depends on `make-v1.2.2` and `MyDependency` depends on
`make-v1.1.1`. Imagine how unnecessarily difficult it would be if our package
manager refused to let us install/build both of these versions of `make`. The
two versions of `make` really have nothing to do with each other and should
never see each other. They were merely tools that we needed in order to build
each package.

This is very different than our app pulling in two versions of the standard
library at *runtime*. That would legitimately cause conflicts and it's
reasonable for the package manager/build system to tell us to fix the conflict.

`pjc` will support a `package.json` config property called
[`buildTimeOnlyDependencies`](https://github.com/facebook/reason/issues/816#issue-183870566).

While they don't yet make use of it, package managers should take
`buildTimeOnlyDependencies` into account when resolving package versions and
deciding how to best flatten - it's a good opportunity for package managers to
relax their task of resolving to one version per package. But even if package
managers don't take this into account, we can still build as if they did, and
create tooling for for generating shrinkwrap/lock files as if they *did*.  Or,
if you just let `npm`/`yarn` do their thing and install multiple versions,
`pjc` will work well at compile time, and likely even link
time because:

1. `pjc` will support building multiple versions per
package. As usual, `node_modules` ensures that there is (literally) space for
multiple versions of source code for them. The exact shape of the `_build` and
`_install` directories ensures that there is (literally) space for multiple
versions to be built.

2. `pjc` will ensure that if there are multiple versions of
your dependency at build time, your package will see the *right* version.  All
of the environment variables will be set accordingly, and `findlib` will be
configured such that you compile against the *right* version. Multiple versions
might exist, but you won't see the *wrong* ones.

It's normal for `buildTimeOnlyDependencies` to result in multiple versions of
packages that are each compiled into their own quarantined `_build` and
`_install`. It's *not* normal for multiple *versions* of runtime dependencies
though.

Before package managers support the `buildTimeOnlyDependencies` well, you might
get in a scenario where you get a link time failure - it won't be due to any
`buildTimeOnlyDependencies` that you depend on - that link time failure will be
a genuine package conflict you need to fix. If package managers supported
`buildTimeOnlyDependencies` well, you would have still had an error - but it
would have been the package manager letting you know early that there *will* be
a link time failure. Without help from package managers, you'll just hear about
it much later, and with a more cryptic error.

## Cross Compiling


Cross compiling *sounds* similar but is very different, and only a little bit
related to `buildTimeOnlyDependencies`. Cross compiling happens when for
*whatever* reason, a package needs to be compiled to an architecture that is
different than the host architecture. Mobile toolchain compilers are the most
mainstream use case you'll likely encounter.

When installing/building a dependency graph, `pjc` pays
attention to your `TARGET_ARCHITECTURE` environment variable (which you can
export in your `package.json`'s `exportedEnvVars` field). By default
`TARGET_ARCHITECTURE` is assumed to be equal to your host architecture if
unspecified.

That, among other things, influences whether or not a package is cross
compiled.

`pjc` makes "space" to build cross compiled packages
without stepping on your regular artifacts, by suffixing `_install` and
`_build` with the architecture. By default, it's assumed `_install`/`_build`
refer to the host architecture. Here's what that would look like for
`TARGET_ARCHITECTURE=arm64`.


    Project Directory
    ========================

    /path/to/MyApp/
     │                        Findlib Installation 
     ├── _install/            ----------------------------------------
     │   └── node_modules/    _install dir mirrors project directory,
     │       └── ...          contains findlib installations for everything.
     │
     ├── _build/             Build Artifacts
     │   ├── src/            ----------------------------------------
     │   │   └── ...         _build dir mirrors entire tree.
     │   └── node_modules/   Contains build artifacts for everything.
     │       └── ...
     │
     │                       ╮
     ├── _install_arm64/     ┊
     │   └── node_modules/   ┊
     │       └── ...         ┊
     │                       ┊
     ├── _build_arm64/       ┊ Same structure as above, but for packages that must
     │   ├── src/            ┊ be compiled for a different runtime architecture.
     │   │   └── ...         ┊
     │   └── node_modules/   ┊
     │       └── ...         ┊
     │                       ╯
     ├── package.json        Original Source Files
     ├── src/                 ----------------------------------------
     │   ├── files.ml        Your typical source directory with node_modules
     │   └── ...             containing all dependencies potentially symlinked.
     │
     └── node_modules/
         └── ...



It is not immediately obvious how many architectures a package at a specific version
needs to be compiled for.
We get into some interesting scenarios when looking at how package manager
resolution combines with `buildTimeOnlyDependencies`, `TARGET_ARCHITECTURE`.

Consider the following scenario. The package manager (or `shrinkwrap` etc) has
resolved `PackageC` to a single version, though multiple packages depend on it.
There is one location for `PackageC`'s source code (likely in
`node_modules/C`). If nothing is marked as a `buildTimeOnlyDependencies`, then
everything is simple.


If `TARGET_ARCHITECTURE` is empty, (defaulted to host), then no matter what,
`pjc` will build everything according to the simplest
convention.


    Dependency Graph       Artifacts
    ----------------       ---------------------


    ╭─────────╮            PackageA/
    │PackageA │            │
    └─┬─────┬─┘            ├── _install/
      │     │              │   ├── ...
      │     │              │   └── node_modules/
      │     └────┐         │       ├── PackageB
      │          │         │       │   └── ...
      │          │         │       └── PackageC
      │          │         │           └── ...
      │     ╭────▼───╮     ├── _build/
      │     │PackageB│     │   ├── ...
      │     └────────┘     │   └── node_modules/
      │          │         │       ├── PackageB
      │          │         │       │   └── ...
      │          │         │       └── PackageC
      │          │         │           └── ...
    ╭─▼──────────▼─╮       ├── package.json
    │PackageC-1.0.0│       ├── src/
    └──────────────┘       │   └── mainPackageA.ml
                           │
                           └── node_modules/
                               ├── PackageB
                               │   └── ...
                               └── PackageC
                                   └── ...



Similarly, any of the arrows in the above dependency diagram could be listed in
the parent package's `buildTimeOnlyDependencies`, and nothing would change
because no matter what, everything needs to run on the host architecture (build
time or not!) and since there's only one `PackageC` source, we only need to
build it at most once.

If `TARGET_ARCHITECTURE` is `arm64` (different than host), and *nothing* is
marked in `buildTimeOnlyDependencies`, then everything must be clearly be
compiled for `arm64`, and no dependencies need to execute on the host for
building.

    Dependency Graph       Artifacts
    ----------------       ---------------------

    ╭─────────╮            PackageA/
    │PackageA │            │
    └─┬─────┬─┘            ├── _install_arm64/
      │     │              │   ├── ...
      │     │              │   └── node_modules/
      │     └────┐         │       ├── PackageB
      │          │         │       │   └── ...
      │          │         │       └── PackageC
      │          │         │           └── ...
      │     ╭────▼───╮     ├── _build_arm64/
      │     │PackageB│     │   ├── ...
      │     └────────┘     │   └── node_modules/
      │          │         │       ├── PackageB
      │          │         │       │   └── ...
      │          │         │       └── PackageC
      │          │         │           └── ...
    ╭─▼──────────▼─╮       ...
    │PackageC-1.0.0│
    └──────────────┘


Now, consider if `TARGET_ARCHITECTURE` is `arm64` (different than host), and
*only the dependency from `PackageA` to `PackageC`* is marked in
`buildTimeOnlyDependencies`. There's still *one* version of `PackageC`
resolved, but it changes the number of times `pjc` will
compile it.  `PackageA` needs to run `PackageC` at build time (so on the host
architecture) and `PackageB` needs to run `PackageC` at runtime. Since the
runtime architecture is different than the build time architecture,
`pjc` will ensure that at package install time, `PackageC`
is compiled twice into appropriately suffixed `_build`/`_install` directories,
even though there's only one *version* of `PackageC`'s source resolved on disk.

      Dependency Graph       Artifacts
      ----------------       ---------------------

      ╭─────────╮            PackageA/
      │PackageA │            │
      └─┬─────┬─┘            ├── _install/
        │     │              │   └── node_modules/
        │     │              │       └── PackageC    ╮
        │     └────┐         │           └── ...     ┊ Only PackageC is compiled for
        │          │         ├── _build/             ┊ the build host architecture.
    buildTimeOnly  │         │   └── node_modules/   ┊ It's the only one needed at build time
        │          │         │       └── PackageC    ╯
        │     ╭────▼───╮     │           └── ...
        │     │PackageB│     │
        │     └────────┘     ├── _install_arm64/     ╮
        │          │         │   ├── ...             ┊
        │          │         │   └── node_modules/   ┊
        │          │         │       ├── PackageC    ┊ Everything else, is compiled for the
        │          │         │       │   └── ...     ┊ TARGET_ARCHITECTURE. Including C!
      ╭─▼──────────▼─╮       │       └── PackageB    ┊ It is still needed at runtime
      │PackageC-1.0.0│       │           └── ...     ┊ even though it was also needed at
      └──────────────┘       ├── _build_arm64/       ┊ build time.
                             │   ├── ...             ┊
                             │   └── node_modules/   ┊
                             │       ├── PackageC    ┊
                             │       │   └── ...     ╯
                             │       └── PackageB
                             │           └── ...
                             ...




So `pjc`'s directory layout is flexible enough to support
compiling of multiple versions of a *single* package, in their isolated
quarantined areas, but this last example shows that it's also flexible enough
to support a *single* version of a package compiled *multiple times* in the
case of a different `TARGET_ARCHITECTURE`.


Quickly consider if `PackageB` where listed in `buildTimeOnlyDependencies` of
`PackageA`, while `TARGET_ARCHITECTURE` was `arm64` as before.

      ╭─────────╮
      │PackageA │
      └─┬─────┬─┘
        │     │
        │     │
        │     └────┐
        │          │
        │     buildTimeOnlyDependency
        │          │
        │     ╭────▼───╮
        │     │PackageB│
        │     └────────┘
        │          │
        │          │
        │          │
        │          │
      ╭─▼──────────▼─╮
      │PackageC-1.0.0│
      └──────────────┘

- `PackageA` must be compiled for the `TARGET_ARCHITECTURE`.
- `PackageC` must be compiled for the `TARGET_ARCHITECTURE` because it's needed by `PackageA` at runtime.
- `PackageB` must be compiled for the host.
- `PackageC` must be compiled for the host as well! Even though `PackageC` is a
  *runtime* dependency of `PackageB`! When `PackageB` acts as a
  `buildTimeOnlyDependency`, `PackageB`'s "runtime", is not the
  `TARGET_ARCHITECTURE` - it is the *host* architecture.


`pjc` handles all of these cases and provides physical space for all of these
artifacts to exist in harmony, simultaneously.



#### Incremental Package Rebuilds

When performing operations such as adding, updating, or removing installed
package sources into the tree, many packages will require building. It's not
always intuitive which ones will require building.  When you add a package, the
package manager might have needed to change another seemingly unrelated version
of a package to dedupe. So pretty much anytime an operation occurs which could
change anything in the dependency graph, we need to evaluate which packages
require rebuilding.


Since `pjc` is an *overlay* on top of package managers, we don't have/want
hooks into any particular package manager's update/install operations.

Instead, every time we perform an operation such as building, we'll record the
package dependency graph, including where on disk each package is, and where
each of their dependencies live, and write it to `.pjc.latest.json`.  This is
very easy to do with `npm` - after each `pjc` build command we will run `npm la
-json` which outputs the graph, and write it to `.pjc.latest.json`.  Then,
after doing any pacakge `install/update` commands, the *next* `pjc build`
command can do the same (`npm la -json`) and do a diff between
`.pjc.latest.json`, in order to determine which packages changed. We consider
those "dirty" nodes. Then we traverse the dependency graph, and rebuild any
node that is dirty or that *transitively* depends on those dirty nodes. We
build that subgraph in topological order, and in parallel - just as we normally
would for any `pjc build` operation. The internal implementation that
walks the graph, should always just accept a subgraph to walk - and by default
it will just assume the *entire* dependency graph.
The major difference with incremental builds, is that before building each
individual package (with the environment automatically prepared), we first
clean the prior build. If packages respect the artifact/install locations,
it's as easy as removing those directories and recreating them before running
the build scripts.

### Parallelism

The implementation of `PackageJsonForCompilers` will output a makefile or shell
script that builds the minimal set of packages optimally after any
install/change/update. It will build the packages in parallel, but some package
build scripts might reach outside of their build artifacts destination into
other shared locations, and doing so is unsafe. Therefore `PackageJsonForCompilers`
specifices that a `package.json` can have a field `pjc.unsafeBuild` which
will ensure that this package build has an exclusive lock on the build process
and releases that lock upon completion/failure.

### Not Shown In This Spec

- Opinionated build system conventions such as module aliases/namespaces.
- How we would implement the system that walks the dependency graph, asking
  each package to build itself into the correct locations.
- How we know which architecture to specify when building a package (based on
  `buildTimeOnlyDependency`, the host architecture, and "target" architecture).
- How we know how many times to build a package - how many architectures.
- External dependencies.
- How environment variables propagate and are "scoped".

### Miscelaneous Requested Features
- Every time you run `esy build`, it should also dump a key/value of environment variables
  to `./.env` at the root of the project being built. This makes it easy for tooling
  to source that file when it discovers it.
  

## Wait, `npm`? What's going On In the Install Process

All `npm install` does is installs your dependencies' source files into a
directory (`node_modules`) by convention. `pjc build` is what actually builds
everything. Therefore, `pjc build` could easily be made to work in other
package managers that install dependency source files into a known location.

`npm` *does* have a hook for each package to build `postinstall` - and you'd *think* that it
would be suficient - but it has issues (it's not parallel, and it doesn't have
all the great environment variables setup correctly when it executes them, it certainly
does not do us the favor of invoking builds multiple times when building for multiple
architectures (in coordination with `buildTimeOnlyDependencies`)).
Still, wouldn't it be nice to avoid the need to run `pjc build` after `npm install`?
We can add that feature later - it involves some clever tricks (`.nmprc` files), but
it's best to currently focus on getting everything else right.


## Questions:
- Can we make it so that build scripts have two `findlib` toolchains - one for compiling and
one for linking? Initially we'll just be exposing the one findlib configuration for both -
and it will have all their transitive dependencies. That's *needed* for linking. But it
would be better if we *also* exposed a second findlib configuration that packages can use
for compiling, which would ensure that they are not depending on the implementation details
of packages (and therefore compile successfully *misleadingly*).

- What should the visibility of global environment variables be?
We know that we want many variables to be transitively propagated: flags such as default
`CAMLRUNPARAM` (to ensure that by default stack traces are enabled etc). We also use
global transitive propagation to compute the `FINDLIB` joined transitive dependency library paths.
We (in `dependency-env` current - and continuing in `pjc`) provide a field
called `global` in the environment variable config which allows the variable to be visible
to any package that transitively depends on the package that publishes the global var.
However, with `buildTimeOnlyDependencies`, things get more
complicated because we will end up with multiple versions of packages simultaneously and
they will all be trying to set the same globals that are propagated. What convention should be used?
The reference implementation of `pjc` called [`esy`](https://github.com/jordwalke/esy) simplifies
the visibility of environment variables by having a field called `scope`, instead of `global: true/false`.

- The global "scope" of an environment variable causes that variable to be seen by packages that
depend on it transitively. It means your package may be able to see variables defined very
deep into its dependency graph. But what about the opposite? What if, while building our packages,
we want our deep dependencies to see variables that *we* specify? That sounds very bad in general,
and it is, but there's some important use cases where we actually want this. For example, at the
top level app, we may want to set a variable that causes the compiler to compile with debug symbols (`-g`).
Even `dependency-env` doesn't solve this right now. Perhaps we want a few values for `scope` such as
`scope:bubble`, `scope:capture` which control which direction environment variables propagate. Right now
we only have the ability to propagate "upwards" from dependencies to dependers (either one level or unlimited
levels "global"). We would want the other direction as well - "downward" propagation (perhaps either one
level or unlimited so that it mirros the "upward" propagation).

