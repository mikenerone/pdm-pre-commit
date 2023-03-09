Pre-commit hooks for pdm Python projects
========================================

## Introduction
This repository provides some public hook implementations for [pre-commit](https://github.com/pre-commit/pre-commit)
to execute code checks using certain analysis tools. They are designed as drop-in replacements for common (and
sometimes "official") hooks found elsewhere, in cases where those tools can be made to work better when run in a more
"`pdm`-aware" way.

### Hooks currently provided
- `mypy` - The mypy type checker

## Usage
Assuming, of course, that you've already installed `pdm` and configured your project, and have `pre-commit` installed,
little configuration is needed. `Mypy` is used as an example here:
1. Ensure that the tool is available in your development environment by adding it as a development dependency:
```shell
pdm add -d mypy
```
2. Add a configuration stanza like this to your list of hooks in your project's `.pre-commit-config.yaml`:
```yaml
    - repo: https://github.com/mikenerone/pdm-pre-commit
      rev: ""
      hooks:
        - id: mypy
```
3. Run `pre-commit autoupdate` in your project directory to automatically update that `rev` entry to the latest
   version of the hook.
4. If you haven't already done so, it's suggested to execute `pre-commit install` in your project directory so that a
   git hook is installed to automatically run your `pre-commit` checks before every `git commit`.

Of course, more advanced configuration is possible. See the documentation for both
[pre-commit](https://github.com/pre-commit/pre-commit) and the tool in question.

### Notes
By default, `pre-commit` explicitly passes only files that have changes staged for commit to the tool. In the case of
`mypy`, this configuration can actually miss problems (e.g. you've changed a function's type annotations, but it is
called using the old signature in an unchanged, and therefore unchecked, file). The provided `mypy` hook supports this
suboptimal configuration because it follows the pattern set by the official hook, but a safer, more complete
configuration would be something like:
```yaml
  - repo: https://github.com/mikenerone/pdm-pre-commit
    rev: "x.x.x.x"
    hooks:
      - id: mypy
        args: [--scripts-are-modules, src/myproject, tests]  # Explicit source roots here
        pass_filenames: false  # ...instead of pre-commit passing changed filenames.
```
Note that this makes the checks take a bit longer (especially the first time), but the analysis result caching in
`mypy` largely addresses that.

## Background
`Pre-commit` normally runs each of its hooks from dedicated environment. Analyzers have access to the
project source code by virtue of starting in the project directory. Unfortunately, some tools also need access to the
project's dependencies in order to do their job fully. `Mypy` is such a tool: ideally, it needs to access dependency
implementations and type stubs in order to cross-check the project code for type consistency.

There are many publicly shared hook implementations, often by the authors of the tools themselves, but they don't
address this problem because there's no consistent way to do so. Consequently, trying to use one of these tools in
`pre-commit` usually requires one of these approaches:
- Use the convenient, public hook without access to the additional files, and live with the resulting loss of
functionality. This is the approach that
[the official `mypy` hook](https://github.com/pre-commit/mirrors-mypy/blob/main/.pre-commit-hooks.yaml) takes with its
`--ignore-missing-imports` argument.
- Use the public hook, but customize the configuration with the `additional_dependencies` setting, duplicating your
list of project dependencies. This will allow `pre-commit` to install them into the dedicated environment (again), but
you now have two places to maintain your dependency list.
- Forgo the convenience of using the shared hook, and instead implement the hook yourself such that the tool is run
from your existing development environment where all the dependencies are already installed (using whatever
environment management approach your project employs).

This repository provides hooks that implement that last approach for `pdm`-managed projects while maintaining the
convenience of being publicly shared.
