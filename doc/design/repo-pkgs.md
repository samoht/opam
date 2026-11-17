# Repository-qualified package names

A switch can select several repositories, ranked by priority. opam stacks them
into one package index, keyed by `name.version`: when two repositories define
`foo.1.0`, the higher-priority one wins and the other definition becomes
unreachable (`OpamRepositoryState.build_index`). `opam list
--columns=repository` and `opam show` name the repository a package comes from,
but the action list does not (#6852). The user cannot ask for a shadowed
definition, and cannot install a fork of `foo` published in a separate
repository, except by renaming the fork (`hypirion-aws` for a fork of `aws`) or
by pinning it.

This design lets a package name carry the repository it comes from.
`opam repository add myfork URL` adds a repository as today, and
`opam install foo@myfork` installs a version of `foo` that `myfork` defines.
The dependencies of `foo@myfork` use `myfork`'s own packages first, and a
package in one repository can depend on a package in another.

## The constraint

A switch has one prefix, and a package `foo` installs under paths derived from
its name. Its build runs `dune build -p foo` and produces `foo.install`, its
files land in `lib/foo` and `doc/foo`, findlib knows it as `foo`, and opam keeps
its metadata in `.opam-switch/packages/foo.1.0/`. Two packages called `foo`
cannot be installed in the same switch, whatever repository they come from.
The qualifier therefore selects where the definition of a package comes from,
and the **base name** (`foo` in `foo@myfork`) stays its identity on disk, at
build time and in the solver.

## Model

### Names

The **index** is today's stack of the selected repositories, where each
`name.version` comes from the highest-priority repository that defines it.

A **qualified name** `foo@R` means the versions of `foo` that repository `R`
defines, and no others. Package names cannot contain `@`, so the name splits at
the first `@`. The left part parses as today, as a package name, a
`name.version` or an atom. The right part is the qualifier, whatever
characters it contains.
`foo@myfork`, `foo.1.0@myfork` and `foo>=1.0@myfork` are valid on the command
line, and `"foo@myfork" {>= "1.0"}` in an opam file. `foo@R` where `R` defines
no `foo` is an error that names the repository.

A **bare name** `foo` means the installed `foo` when there is one, and `foo`
in the index otherwise. A switch that never uses a qualified name therefore
behaves as today.

The qualifier never reaches a variable or a filter. `foo:installed`,
`%{foo:lib}%` and `opam var foo:version` use the base name, which is unique in
a switch.

### Repositories

opam already models a repository as a name and a URL. A qualifier names a
repository in one of three ways, tried in this order.

1. **A URL.** A qualifier containing `://` is a repository URL, as in
   `opam install tcpip@git+https://github.com/mirage/opam-overlays.git`. A
   qualifier of the form `host/path`, whose first segment contains a dot and
   does not start with one, is a git repository over https:
   `tcpip@github.com/mirage/opam-overlays` means the URL above, for any host.
   `./x` and `../x` are therefore no URLs, and neither is the ssh form
   `user@host:path`, which is written `ssh://user@host/path`.
2. **An alias.** A `repositories:` field is a list of aliases, each a short
   name for a URL in either form:

   ```
   repositories: [
     "mirage" {"git+https://github.com/mirage/opam-overlays.git"}
   ]
   ```

   It is valid in an opam file, where it applies to that file, and in a `repo`
   file, where it applies to every opam file of the repository. The aliases of
   the opam file come first. Each file uses only its own aliases and those of
   its repository, so two repositories can give the same alias different URLs.
3. **A name.** Otherwise the qualifier is the name of a configured
   repository, the one given to `opam repository add`. Names differ from one
   machine to the next, so they are valid on the command line and in the opam
   file of a project only. In an opam file that a repository publishes, a
   name that is neither a URL nor a declared alias is an error, which names
   it and shows the `repositories:` field that declares it.

A URL, written directly or through an alias, binds to the selected repository
whose URL matches, whatever its name. Two URLs match when they have the same
host, path and branch. The transport, the user and a trailing `.git` do not
count, and local paths match by path. When several selected repositories match, the
highest-priority one wins. Output prints the name of the repository.

When the URL matches no selected repository, `opam install` proposes to add
the repository to the current switch, under the alias or under the last
component of its path (`opam-overlays`). A repository configured for another
switch is selected without a new fetch. The repository stays selected after
the command, since the packages installed from it need it for later upgrades.
It takes the lowest priority, so the index keeps every definition it had and
gains only the base names that no other selected repository defines. opam
asks first, so no file adds a repository without the user's confirmation; this
keeps the dependency injection that AltGr raised against `provides:` (#314)
under the user's control. The question requires unsafe-yes
(`OpamConsole.confirm ~require_unsafe_yes`): `--yes` declines it, and only
`--confirm-level=unsafe-yes` accepts it unattended. When the user declines, a
qualifier on the command line is an error, and the packages that depend on the
repository are unavailable, with a warning that names it.

A qualifier that is neither a URL, an alias nor the name of a configured
repository is an error (#3809, #3292). In a dependency, it also makes the
packages that carry it unavailable. A repository whose URL is the recorded
source of an installed package raises no warning when it is not selected.

### Sources

The **source** of a base name is where its versions come from in one solve,
either the index or one repository. A switch holds one package per base name,
so each base name has one source. opam chooses the sources before it calls the
solver, by walking the dependencies, over all versions, of the requested and
the installed packages. For a base name `n`, the first rule that matches
applies.

1. **The command line.** `n@R` gives `n` the source `R`, and
   `opam install --index n` gives it the index.
2. **Qualified dependencies.** A dependency on `n@R`, including one inside a
   disjunction, gives `n` the source `R`.
3. **Installed packages.** An installed `n` keeps its source, unless a package
   that depends on it changes source in this solve.
4. **Own packages first.** A bare dependency on `n`, from a package whose
   source is a repository `R`, gives `n` the source `R` when `R` defines `n`.
5. **Otherwise**, `n` has the index as its source.

When the dependencies of rule 2 name several repositories for `n`, the rule
does not apply. When those of rule 4 do, opam stops before the solve, names
the repositories, and asks for `opam install n@R`.

A qualified dependency is a requirement: a package version that depends on
`n@R` is unavailable in a solve where the source of `n` is another repository
or the index, and when `R` names no repository. The requirement compares
URLs, so it holds for a source recorded in the switch state whose repository
is no longer selected. Such a source, or one that no longer defines `n`,
takes its definitions from the index.

Rules 3 and 5 give today's behaviour to a switch that never uses a qualified
name, and rule 3 makes a choice last. A package that followed by rule 4 keeps
its source after the package that brought it is removed;
`opam install --index n` returns it to the index.

Take `default`, with the higher priority, and `samoht`, a fork. `default`
defines `tls.1`, `x509.1` and `x509.2`; `samoht` defines its own `tls.1` and
`x509.1`; in both, `tls.1` depends on `x509`.

- `opam install tls` installs `default`'s `tls.1` and `x509.2` from the index,
  as today.
- `opam install tls@samoht` gives `tls` the source `samoht` (rule 1). Since
  `tls` changes source, `x509` follows it (rules 3 and 4). opam reinstalls
  `tls.1` from `samoht` and replaces `x509.2` with `samoht`'s `x509.1`, the
  only version `samoht` defines.
- `opam install tool`, where `tool` depends on `tls` and `x509`, uses the
  installed `tls@samoht` and `x509@samoht` (rule 3). `opam install tls` is a
  no-op.
- `opam install --index tls` returns `tls` to the index (rule 1), and `x509`
  follows it back (rules 3 and 5).

### Dependencies between repositories

Take three repositories, `default`, `mirage` and `samoht`, where the `repo`
file of `samoht` has the `mirage` alias above. `samoht`'s `app.1` depends on
`tcpip@mirage` and on `fmt`.

- `opam install app@samoht` gives `app` the source `samoht` (rule 1) and
  `tcpip` the repository that the `mirage` alias binds to (rule 2). When no
  selected repository matches, opam proposes to add it. The dependencies of
  `tcpip` come from `mirage` first, those of `app` from `samoht` first
  (rule 4), and `fmt`, which neither defines, comes from the index (rule 5).
- If the user added the `mirage` repository under the name `overlays`, the
  URL still binds it, and output shows `tcpip@overlays`.
- If both repositories define `cstruct`, and both `app` and `tcpip` depend on
  it, rule 4 names two repositories. opam stops before the solve, and
  `opam install app@samoht cstruct@overlays` settles it; rule 3 keeps it
  settled.
- The opam file of a project can depend on `app@samoht`, with the name.

### Pins

A pin replaces the definition of a base name after its source is chosen, as it
replaces the definition from every repository today, so `opam pin add tls URL`
applies whatever the source of `tls`. `opam pin add tls@samoht URL` also gives
the pinned `tls` the source `samoht`, so its dependencies come from `samoht`
first, and the source remains after the pin is removed. `pin-depends:` accepts
qualified names with the same meaning, resolved with the aliases of its opam
file.

### Identity

Each base name has one source per solve, so the solver sees base names only,
one package per `name.version`, in a universe of the size it sees today. A
change of source at the same version is a reinstallation, which opam already
schedules when the definition of an installed package changes.

The switch state records a package whose source is the index with its bare
name, and any other qualified by the URL of its repository, as in
`"tcpip.7.0@git+https://github.com/mirage/opam-overlays.git"`. The URL binds
as in Repositories each time the switch loads, so renaming a repository leaves
the switch state unchanged. A switch that never uses a qualified name has the
switch state of today and needs no migration. The switch invariant keeps base
names: `opam switch create s ocaml-variants.5.3.0@samoht` gives the package
its source by rule 1, and rule 3 keeps it.

## Command line

### Arguments

- **Requests.** `install` (including `--deps-only`), `upgrade`, `show`,
  `source`, `tree`, `lock`, `switch create` and the patterns of `list` accept
  qualified names and apply rule 1. `install` gains `--index`, which gives the
  named packages the index as their source. On the command line, the test for
  a local path (`atom_or_local` in `src/client/opamArg.ml`) applies to the part
  before the `@`, so a URL qualifier keeps its directory separators.
- **Installed packages.** `remove`, `reinstall`, `pin remove` and `pin edit`
  act on the installed or pinned package. For `remove`, `reinstall` and
  `upgrade`, a qualifier that does not match the source of the installed
  package is an error.
- **`pin`.** `opam pin add tls@samoht TARGET` behaves as in Pins. With one
  argument, `opam pin add user@host:path` keeps its meaning today, a target.
- **Files.** The opam file of a project, given as `opam install .` or
  `foo.opam`, may use qualified names in its dependencies; its `name:` field
  stays a base name.
- **Base names.** Every other argument takes a base name. This covers
  variables, `var`, `config`, `switch set-invariant`, `--formula`, `lint --package`, the selection
  options of `list`, and `opam admin`. Plugins (`opam foo`) come from the
  index.

### Output

Commands print the bare name of a package whose source is the index, and its
name qualified by the name of the repository otherwise, or by the URL when no
selected repository matches. A user who never writes a qualified name sees
today's output.

- **Action lists** annotate a package whose source is a repository, as they
  annotate a pinned one: `tls 1 (samoht)`. A change of source shows as a
  reinstallation. A package that several selected repositories define carries
  the repository the index takes it from: `x509 2 (default)` (#6852).
- **`list`**, **`show`** and **`pin list`** print the qualified name. The
  `repository` column of `list` gives the source of a qualified package.
- **`--json`** output carries the qualified name, plus the repository and the
  base name as separate fields.
- **Solver errors** list the installed packages whose source is a repository
  and suggest `opam install --index`.

### Repositories

- **`repository remove`** asks before removing a repository that installed
  packages come from. Those packages then count as coming from the index.
- **`repository priority`** changes the index, as today. Packages installed
  with a qualified name keep their source.
- **`opam admin`** works on one repository. `admin check` reports a dependency
  on another repository as external, and `admin lint` checks that every
  qualifier is a URL or a declared alias.

## Files and formats

- **opam files.** opam-repository decides by policy whether its packages may
  depend on other repositories.
- **Switch state.** A qualified name in `switch-state` needs a format version
  bump, and older opam versions cannot read a switch that holds one.
- **Lock files and switch exports** name packages as the switch state does,
  qualified by repository URL, so they work on a machine where the repository
  has another name, or is not yet added. With `--freeze`, the URL of a git
  repository carries the commit it was at (`URL#commit`), as package URLs
  already do.
- **`opam switch export --full`** embeds the opam file of every installed
  package, keyed by the name the selections use. Each embedded file also
  gains, in its own `repositories:` field, the URL of every alias and
  repository name that its dependencies use, so its qualified dependencies
  resolve without the repositories. `opam switch import` installs from those
  files whatever repositories are selected, without proposing to add one. A
  package whose URL matches no selected repository keeps its qualified name,
  so exporting the imported switch writes the same file.
- **orb** builds with `opam switch create --repos`, `opam install` and
  `opam switch export --full --freeze`, and rebuilds with
  `opam switch import` into a switch whose only repository is empty. The
  rebuild therefore installs from the embedded opam files, and its export
  matches that of the build. orb looks up the root package in the embedded
  files by the name the selections give, which works because both use the
  same name. It answers every question with `--confirm-level=unsafe-yes`,
  which accepts the proposal to add a repository. A `repositories:` field in
  the opam file of a MirageOS unikernel replaces `x-mirage-extra-repo`.
- **Other tools.** Plugins that link the opam libraries get qualified names
  from parsed opam files and the base name from `OpamPackage.Name.base`. Tools
  that validate names with their own grammar reject `@` (opam-0install,
  opam-repo-ci, ocaml.org, opam-monorepo). They are affected only when a file
  they read contains a qualified name.

## Build

Only base names reach the build. `%{name}%` and `OPAM_PACKAGE_NAME` expand to
the base name, so `dune build -p %{name}%` and the lookup of `%{name}%.install`
work unchanged. Every path of the switch (`.opam-switch/packages/`, `build/`,
`sources/`, `NAME.changes`) uses the base name.

## Implementation

1. `OpamPackage.Name`: accept a qualifier after the first `@`, and add `base`,
   `repository` and `qualify`. `OpamPackage.of_string` and `to_string`,
   `OpamFormula.atom_of_string` and the `OpamArg` converters split at the
   first `@`.
2. `OpamFile.OPAM` and `OpamFile.Repo`: the `repositories:` field.
3. `OpamSwitchState`: `sources` maps a base name to the URL of its
   repository. `select_sources` resolves the qualifiers and chooses the source
   of each base name before the solve. The versions of a name come from the
   index or from its repository (`repo_opams`). `universe` gives the solver
   base names and makes unavailable the versions whose qualified dependencies
   disagree. The selections are read and written with URL-qualified names.
4. `OpamCommands`: `select_sources` in every command that accepts qualified
   names, `install --index`, and the proposal to add a repository.
5. Output: the origin annotation, `list`, `show`, `pin list` and solver
   errors.

Each step lands with reftests under `tests/reftests/`.

## Related discussions

- #5974: scopes in package identifiers, as npm's `@org/pkg`. Here a publisher
  scope is a repository, one per organisation.
- #6852: show the origin repository in action lists.
- #314 and `provides.md`: aliases and replacements.
- #1879: package identity.
- #6385: renaming a repository.
- #4003: `pin-depends` is not transitive.
- #3809 and #3292: per-switch repository selection.
