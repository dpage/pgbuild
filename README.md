# pgbuild

GitHub Actions workflows that build PostgreSQL and its dependencies for the
pgAdmin build farm, on both Windows and macOS. A number of build tools are
included alongside the libraries so that the Windows builds are reproducible,
though those are generally downloaded as pre-built utilities rather than
compiled here.

This repository began life as [dpage/winpgbuild](https://github.com/dpage/winpgbuild),
which built the Windows side only, and its history is preserved here. The macOS
builds replace a set of Jenkins jobs that pgAdmin is retiring as it moves its
build farm onto GitHub Actions.

## Build status

| Tree | Status |
|------|--------|
| Windows | [![Build All (Windows)](https://github.com/pgadmin-org/pgbuild/actions/workflows/build-all-windows.yml/badge.svg)](https://github.com/pgadmin-org/pgbuild/actions/workflows/build-all-windows.yml) |
| macOS | [![Build All (macOS)](https://github.com/pgadmin-org/pgbuild/actions/workflows/build-all-macos.yml/badge.svg)](https://github.com/pgadmin-org/pgbuild/actions/workflows/build-all-macos.yml) |

Per-package status is on the Actions tab; with a leaf workflow for each package
on each platform there are rather too many of them to list usefully here.

## Licence

The build scripts and workflows in this repository are released under the
PostgreSQL licence; see [LICENSE](LICENSE).

That licence covers **this repository only**. The release assets these
workflows publish are builds of third-party software, each of which carries its
own upstream licence, and redistributing them means complying with those rather
than with the licence above. Every package records its upstream licence in
[manifest.json](manifest.json), as a `licence` URL pointing at the canonical
licence text and an `spdx_license` expression naming it; the same information
is emitted into each artifact as an SPDX 2.3 document under `MANIFESTS/`, so a
downstream consumer can read it out of the tarball or zip without coming back
here.

## Layout

GitHub does not support subdirectories under `.github/workflows/`, so the
platform lives in the filename rather than in a directory. Every leaf workflow
is named `<package>-<platform>.yml`, giving `openssl-windows.yml`,
`openssl-macos.yml` and so on, and each platform has its own orchestrator.

| Workflow | What it does |
|----------|--------------|
| `build-all.yml` | Convenience dispatcher; starts both orchestrators and returns |
| `build-all-windows.yml` | Nightly Windows DAG, calling the 19 Windows leaves in dependency order |
| `build-all-macos.yml` | Nightly macOS DAG, calling the 5 macOS leaves in dependency order |
| `manifest.yml` | Reusable workflow that reads pinned versions out of `manifest.json` |
| `<package>-windows.yml` | One Windows package |
| `<package>-macos.yml` | One macOS package, built for both architectures |

The macOS builds install into `/opt/pgbuild/<package>`, and the binaries
therefore carry absolute `install_name` references into that prefix; anything
that relocates them into an application bundle will need to rewrite those.

### Reusable workflow limits

GitHub Actions caps a workflow at 20 unique reusable workflows across its
entire call tree. `build-all-windows.yml` sits exactly on that cap, with its 19
leaves plus `manifest.yml`, which every leaf calls and which counts once;
`build-all-macos.yml` uses 6, being its 5 leaves plus `manifest.yml`. Splitting
the two platforms apart is what keeps either of them buildable at all, and it
is also why `build-all.yml` dispatches the two orchestrators through the API
instead of calling them with `uses:`, since doing the latter would come to 27
unique workflows and the run would simply be rejected.

Adding a further Windows package will need `build-all-windows.yml` broken into
staged orchestrators dispatched the same way. There is plenty of room on the
macOS side.

## Platforms and architectures

Windows builds target x64 and run on `windows-latest`. macOS builds are
produced separately for each architecture rather than as universal binaries:
arm64 on `macos-15`, and x86\_64 on `macos-15-intel`. Both are free hosted
runners for public repositories.

Every macOS build sets `MACOSX_DEPLOYMENT_TARGET=14.0`, because pgAdmin
supports macOS 14 (Sonoma) and above. The Jenkins jobs never set it at all and
so inherited whatever the builder's SDK happened to default to, which tied the
supported floor to the build machine; pinning it makes that floor explicit.

## Releases and artifact naming

Each workflow publishes its output as a rolling prerelease under a stable tag,
so that consumers can download a known URL and always get the most recent
build. Tags carry the platform and, on macOS, the architecture:

```
openssl-win64-latest
openssl-macos-arm64-latest
openssl-macos-x86_64-latest
postgresql-18-win64-latest
postgresql-18-macos-arm64-latest
postgresql-18-macos-x86_64-latest
dependencies-win64-latest
```

Windows assets are `.zip`, matching the platform's conventions and the existing
downstream tooling. macOS assets are `.tar.gz`, which is the only one of the
two that preserves the dylib version symlinks and executable bits that anything
linking against these trees depends on.

PostgreSQL release tags carry the major version only, so `postgresql-18-...`
tracks whatever minor release `manifest.json` currently pins. Build artifacts,
as opposed to releases, carry the full version and so are unambiguous.

## Automation

The two orchestrators run nightly on a schedule, Windows at 00:00 UTC and macOS
at 02:00 UTC, because GitHub only retains build artifacts for 90 days and the
releases need to stay fresh. Every leaf workflow is also individually
dispatchable, and a leaf run that cannot find a dependency artifact from its own
run falls back to downloading that dependency's rolling release, which is what
makes standalone dispatch work at all.

## Version information

Versions for every package are pinned in [manifest.json](manifest.json), which
both platforms read through the shared `manifest.yml` reusable workflow. There
is one entry per package regardless of how many platforms build it.

## Adding a package

First add an element to the array in [manifest.json](manifest.json). The
elements are more or less in alphabetical order, libiconv being the exception.

```json
{
    "name": "diffutils",
    "version": "2.8.7-1",
    "source": "https://gnuwin32.sourceforge.net/packages/diffutils.htm",
    "licence": "https://git.savannah.gnu.org/cgit/diffutils.git/tree/COPYING",
    "spdx_license": "GPL-3.0-or-later"
}
```

The workflows use `name` and `version` to drive the build, and `source`,
`licence` and `spdx_license` to populate the SPDX manifest that ships inside
each artifact.

Then add an output to `manifest.yml`, in both the `outputs` section of the
`workflow_call` trigger and the `outputs` of the `set_versions` job:

```yaml
DIFFUTILS_VERSION:
    description: "diffutils version"
    value: ${{ jobs.set_versions.outputs.output18 }}
```

```yaml
    output18: ${{ steps.step1.outputs.DIFFUTILS_VERSION }}
```

A GitHub output named `uppercase($name)_VERSION` is then available to any
workflow that calls `manifest.yml`.

## Using these workflows from elsewhere

All of the workflows can be called from another workflow, though you will have
to provide your own `manifest.json`.

## PostgreSQL build configuration

The macOS PostgreSQL builds are configured with `--with-openssl`,
`--with-gssapi`, `--with-zstd`, `--with-lz4` and `--without-icu`. The two
compression options are new relative to the Jenkins jobs, which had neither,
and they close
[pgadmin-org/pgadmin4#9425](https://github.com/pgadmin-org/pgadmin4/issues/9425),
where macOS users could not restore a zstd-compressed dump that Windows users
could. `--with-zstd` only exists from PostgreSQL 15 onwards, so it is omitted
on 14.

`make check` is not run on macOS. It builds a temporary install and relies on
`DYLD_LIBRARY_PATH` to point the new binaries at the matching libpq, but System
Integrity Protection strips every `DYLD_*` variable from a protected process's
environment, so that libpq is never found. It appeared to pass under Jenkins
only because earlier runs had left a libpq behind in the real installation
directory for the binaries to fall back on. The workflows install first and then
run `make installcheck` against a server started from the installed tree, whose
binaries resolve their libraries through absolute `install_name` references and
need no `DYLD_*` at all.

MIT Kerberos is likewise built without running its own `make check`; the reasons
are in a comment in `krb5-macos.yml`.

## Windows GSSAPI

Currently supported versions of PostgreSQL should build, and all dependencies
are included automatically *except* for MIT Kerberos on the older Windows
branches; see
[this thread](https://www.postgresql.org/message-id/CA%2BOCxoxwsgi8QdzN8A0OPGuGfu_1vEW3ufVBnbwd3gfawVpsXw%40mail.gmail.com)
for the background.

## TODO

The following dependencies are yet to be completed on Windows:

* Perl
* Python
* TCL

The following have not been supported on Windows but potentially could be in
the future under Meson:

* Bonjour
* LLVM
* Readline (or libedit, as it is not GPL)
