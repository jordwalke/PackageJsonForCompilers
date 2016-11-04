

# Sub Projects (Ways To Help)

### Creating Our Own `postinstall` process [SOLVED: Solution Coming]

The current `postinstall` process has issues:

- It is invoked for many operations such as `npm link` (and `npm install
  newPackage`), and when invoked for these relatively *small* operations it
  tends to perform the entire `postinstall` process for packages that don't
  even need to be rebuilt when doing `npm link`.
- Many ported packages' builds aren't idempotent, and will fail the second time
  you try to build them. So, when you do a small change like adding one new
  package, all the existing packages get rebuilt and that causes unnecessary
  failures.

For this subproject, the goal is to create our own `postinstall` process for
`PackageJsonForCompilers` that hijacks the main `postinstall` project in a very
specific way - which lets us solve the problems with typical `postinstall`.
What is that specific way? We want all packages to be able to include some
command in their `postinstall` (let's suppose it's called `magicCommand`)
which causes `magicCommand` to behave as a noop for any package except the top
level package being installed.


