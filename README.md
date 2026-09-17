# pgbuild

GitHub Actions workflows that build PostgreSQL and its dependencies for the
pgAdmin build farm, on both Windows and macOS. The Windows side also packages a
handful of build tools, namely Meson, Ninja, pkgconf, winflexbison and
diffutils, so that its builds are reproducible; most of those are downloaded as
pre-built utilities rather than compiled here.

This repository began life as [dpage/winpgbuild](https://github.com/dpage/winpgbuild),
which built the Windows side only, and its history is preserved here. The macOS
builds replace a set of Jenkins jobs that pgAdmin is retiring as it moves its
build farm onto GitHub Actions.

## Licence

The build scripts and workflows in this repository are released under the
PostgreSQL licence; see [LICENSE](LICENSE).

That licence covers **this repository only**. The release assets these
workflows publish are builds of third-party software, each of which carries its
own upstream licence, and redistributing them means complying with those rather
than with the licence above. Every package records its upstream licence in
[manifest.json](manifest.json), as a `licence` URL pointing at the canonical
licence text and an `spdx_license` expression naming it.
## Build status

These are the workflows that run on a schedule, so these are the badges that
mean anything.

| Workflow | Status |
|----------|--------|
| Build All (Windows) | [![Build All (Windows)](https://github.com/pgadmin-org/pgbuild/actions/workflows/build-all-windows.yml/badge.svg)](https://github.com/pgadmin-org/pgbuild/actions/workflows/build-all-windows.yml) |
| Build All (macOS) | [![Build All (macOS)](https://github.com/pgadmin-org/pgbuild/actions/workflows/build-all-macos.yml/badge.svg)](https://github.com/pgadmin-org/pgbuild/actions/workflows/build-all-macos.yml) |
| Set package versions | [![Set package versions](https://github.com/pgadmin-org/pgbuild/actions/workflows/manifest.yml/badge.svg)](https://github.com/pgadmin-org/pgbuild/actions/workflows/manifest.yml) |

The per-package workflows deliberately carry no badges. Each is called by its
orchestrator rather than run in its own right, and GitHub attributes a
`workflow_call` run to the caller, so a package badge shows either "no status"
or, worse, a stale green from whenever somebody last dispatched it by hand.
Either way it says nothing about last night's build. The badges above cover
them: when a package fails, its orchestrator goes red.

## What gets built

The Windows side builds more than pgAdmin itself needs, on the principle that
if we are building PostgreSQL's dependencies anyway then the results may as
well be available to anyone else who wants them. macOS is expected to catch
up, so a "not yet built" below is a gap rather than a decision.

| Package | Windows | macOS |
|---------|---------|-------|
| PostgreSQL | [`postgresql-windows.yml`](.github/workflows/postgresql-windows.yml) | [`postgresql-macos.yml`](.github/workflows/postgresql-macos.yml) |
| PostgreSQL (dev) | [`postgresql-dev-windows.yml`](.github/workflows/postgresql-dev-windows.yml) | not yet built |
| OpenSSL | [`openssl-windows.yml`](.github/workflows/openssl-windows.yml) | [`openssl-macos.yml`](.github/workflows/openssl-macos.yml) |
| MIT Kerberos | [`krb5-windows.yml`](.github/workflows/krb5-windows.yml) | [`krb5-macos.yml`](.github/workflows/krb5-macos.yml) |
| zstd | [`zstd-windows.yml`](.github/workflows/zstd-windows.yml) | [`zstd-macos.yml`](.github/workflows/zstd-macos.yml) |
| lz4 | [`lz4-windows.yml`](.github/workflows/lz4-windows.yml) | [`lz4-macos.yml`](.github/workflows/lz4-macos.yml) |
| zlib | [`zlib-windows.yml`](.github/workflows/zlib-windows.yml) | system |
| ICU | [`icu-windows.yml`](.github/workflows/icu-windows.yml) | not yet built |
| gettext | [`gettext-windows.yml`](.github/workflows/gettext-windows.yml) | not yet built |
| libiconv | [`libiconv-windows.yml`](.github/workflows/libiconv-windows.yml) | system |
| libxml2 | [`libxml2-windows.yml`](.github/workflows/libxml2-windows.yml) | not yet built |
| libxslt | [`libxslt-windows.yml`](.github/workflows/libxslt-windows.yml) | not yet built |
| ossp-uuid | [`ossp-uuid-windows.yml`](.github/workflows/ossp-uuid-windows.yml) | not yet built |
| Dependency bundle | [`bundle-deps-windows.yml`](.github/workflows/bundle-deps-windows.yml) | not yet built |
| diffutils | [`diffutils-windows.yml`](.github/workflows/diffutils-windows.yml) | not yet built |
| Meson | [`meson-windows.yml`](.github/workflows/meson-windows.yml) | not yet built |
| Ninja | [`ninja-windows.yml`](.github/workflows/ninja-windows.yml) | not yet built |
| pkgconf | [`pkgconf-windows.yml`](.github/workflows/pkgconf-windows.yml) | not yet built |
| winflexbison | [`winflexbison-windows.yml`](.github/workflows/winflexbison-windows.yml) | not yet built |

"system" means the copy in macOS itself, in `/usr/lib`, which PostgreSQL links
directly. It does not mean Homebrew. Nothing here depends on Homebrew being
installed, deliberately: the point of publishing these builds is that they work
for somebody using MacPorts or Fink, or nothing at all. The macOS PostgreSQL
build goes further and puts its own include directories ahead of Homebrew's, so
that a runner image which happens to ship Homebrew copies of zstd and lz4
cannot quietly get them compiled in.

ICU is the one to watch there: the macOS build currently configures
`--without-icu`, so adding it means building our own and turning the flag
round, not linking whatever the machine happens to have.

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

The macOS builds install into `/opt/pgbuild/<package>`, and the resulting
binaries carry absolute `install_name` references into that prefix; anything
that relocates them into an application bundle will need to rewrite those.

### Reusable workflow limits

GitHub Actions caps a workflow at 20 unique reusable workflows across its whole
call tree, and that cap is what shapes the structure here.

`manifest.yml` is a single shared reusable workflow. Every leaf calls it to read
its pinned version out of `manifest.json`, but because the cap counts *unique*
workflows rather than calls, it contributes one to the total no matter how many
leaves call it. `build-all-windows.yml` therefore comes to exactly 20: its 19
Windows leaves plus `manifest.yml`. It is full, and adding a twentieth Windows
package will mean breaking it into staged orchestrators dispatched through the
API rather than called with `uses:`.

`build-all-macos.yml` comes to 6, being its 5 macOS leaves plus `manifest.yml`,
so there is plenty of room on that side.

Splitting the platforms apart is what keeps either tree buildable, since a
single combined orchestrator would have come to 25. It is also why
`build-all.yml` dispatches the two orchestrators through the API instead of
calling them: calling them would nest their trees inside its own, reaching 27,
and the run would be rejected outright.

## Platforms and architectures

Windows builds target x86\_64 and run on `windows-latest`. macOS builds are
produced separately for each architecture rather than as universal binaries:
arm64 on `macos-15`, and x86\_64 on `macos-15-intel`. All three are free hosted
runners for public repositories.

Every macOS build sets `MACOSX_DEPLOYMENT_TARGET=14.0`, because pgAdmin
supports macOS 14 (Sonoma) and above. The Jenkins jobs never set it at all and
so inherited whatever the builder's SDK happened to default to, which tied the
supported floor to the build machine; pinning it makes that floor explicit.

## Releases and artifact naming

Each workflow publishes its output as a rolling prerelease under a stable tag,
so that consumers can download a known URL and always get the most recent
build. Tags carry the platform and, on macOS, the architecture:

Every tag is `<package>-<platform>-<architecture>-latest`, with the platform and
the architecture named separately so that either can vary:

```
openssl-windows-x86_64-latest
openssl-macos-arm64-latest
openssl-macos-x86_64-latest
postgresql-18-windows-x86_64-latest
postgresql-18-macos-arm64-latest
postgresql-18-macos-x86_64-latest
dependencies-windows-x86_64-latest
```

Build artefacts follow the same scheme with the full version in place of
`latest`, so `openssl-3.5.8-windows-x86_64` and `openssl-3.5.8-macos-arm64`.

Windows spells its architecture out as `x86_64` rather than leaving it implied
in a `win64` suffix, even though x86\_64 is the only Windows architecture built
today, because the architecture being separable is the whole point; see the
roadmap below.

Windows assets are `.zip`, matching the platform's conventions and the existing
downstream tooling. macOS assets are `.tar.gz`, which is the only one of the two
that preserves the dylib version symlinks and executable bits that anything
linking against these trees depends on.

PostgreSQL release tags carry the major version only, so `postgresql-18-...`
tracks whatever minor release `manifest.json` currently pins. Build artefacts,
as opposed to releases, carry the full version and so are unambiguous.

A rolling release is created once and thereafter updated in place: each build
moves the tag onto its own commit and replaces the asset, rather than deleting
the release and making a new one. The release therefore keeps its identity and
its creation date across builds, so read the asset's timestamp, not the
release's, to tell how fresh a download is. Creating a release on this
repository has proved unreliable from within Actions, returning 403 for long
stretches on a token that was demonstrably allowed to do it, and confining that
call to the first build of a given package is the practical way around that; the
macOS workflows retry regardless, and check that what they published is not a
draft, since a draft release has no tag and cannot be downloaded.

### Software bills of materials

Every build writes an SPDX 2.3 document into `MANIFESTS/<package>.spdx.json`
inside its own install tree, generated from `manifest.json` at the commit being
built, so the package name, version, upstream homepage and SPDX licence
expression travel inside the artifact. Because the manifest lands in the
install tree rather than beside it, a bundler that simply extracts its
dependencies' artifacts, as `bundle-deps-windows.yml` does, collects their
manifests for free; the two PostgreSQL workflows additionally copy the
dependency manifests alongside their own. A downstream consumer can therefore
read the licence position out of the tarball or zip without coming back here.

This is why `source` and `spdx_license` in `manifest.json` are not merely
informational: changing them changes what ships.

## Automation

The two orchestrators run nightly on a schedule, Windows at 00:00 UTC and macOS
at 02:00 UTC. Each is a single-run DAG that builds its whole tree in dependency
order within one run, which is what lets a downstream job consume an upstream
job's artifact directly. The individual leaf workflows have no cron of their
own; they are triggered by the orchestrator, or dispatched by hand.

Every leaf is also independently dispatchable. A leaf run that cannot find a
dependency artifact from its own run falls back to downloading that
dependency's rolling release instead, which is what makes a standalone dispatch
work at all.

The point of running nightly is currency rather than survival. Release assets do
not expire: `dpage/winpgbuild` still serves its `postgresql-13-latest` asset,
published in January 2026, for a PostgreSQL major version the manifest stopped
building some time ago. What does expire is the Actions artifacts the jobs pass
between each other, which this repository retains for 90 days, that being
GitHub's maximum. So a nightly rebuild is about picking up upstream minor
releases and security fixes promptly, and about knowing that the build still
works, rather than about stopping the published assets from vanishing.

## Version information

Versions for every package are pinned in [manifest.json](manifest.json), which
both platforms read through the shared `manifest.yml` reusable workflow. There
is one entry per package regardless of how many platforms build it, so a version
bump lands once and applies everywhere.

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

The workflows use `name` and `version` to drive the build, and `name`, `version`,
`source` and `spdx_license` to populate the SPDX manifest that ships inside each
artifact. `licence`, the URL of the human-readable licence text, is the one
field that is purely informational.

Then declare the version in `manifest.yml`, which needs two additions because a
reusable workflow's outputs have to be threaded up from the step that sets them,
through the job, to the `workflow_call` trigger. Add an entry under
`on.workflow_call.outputs`:

```yaml
DIFFUTILS_VERSION:
    description: "diffutils version"
    value: ${{ jobs.set_versions.outputs.output18 }}
```

and a matching one under `jobs.set_versions.outputs`:

```yaml
    output18: ${{ steps.step1.outputs.DIFFUTILS_VERSION }}
```

The `outputN` names are arbitrary plumbing and carry no meaning beyond being
unique; pick the next free number, which today means `output19`, since
`manifest.json` has 18 packages and `manifest.yml` currently runs from `output1`
to `output18`. What actually does the matching is the name: the `Set versions`
step iterates over the packages and writes `uppercase($name)_VERSION=$version`
for each, so the step output is keyed by the package name from `manifest.json`
and must be spelled the same way on both lines above.

Any workflow that calls `manifest.yml` can then read
`needs.get-versions.outputs.DIFFUTILS_VERSION`.

Finally, write the leaf workflow itself as `<package>-<platform>.yml` and add it
to the appropriate orchestrator. Check the call-tree arithmetic above first if
the platform is Windows, because that tree has no headroom left.

## Using these workflows from elsewhere

All of the workflows can be called from another workflow, though you will have
to provide your own `manifest.json`.

## PostgreSQL build configuration

### macOS

Configured with `--with-openssl`, `--with-gssapi`, `--with-zstd`, `--with-lz4`
and `--without-icu`, against the OpenSSL, MIT Kerberos, zstd and lz4 trees built
by the other four macOS workflows.

The two compression options are new relative to the Jenkins jobs, which had
neither, and they close
[pgadmin-org/pgadmin4#9425](https://github.com/pgadmin-org/pgadmin4/issues/9425),
where a macOS user could not restore a zstd-compressed dump that a Windows user
could produce without trouble. `--with-zstd` only exists from PostgreSQL 15
onwards, so it is omitted on 14 and that branch gets lz4 support alone.

`make check` is not run on macOS. It builds a temporary install and relies on
`DYLD_LIBRARY_PATH` to point the new binaries at the matching libpq, but System
Integrity Protection strips every `DYLD_*` variable from a protected process's
environment, so that libpq is never found. It appeared to pass under Jenkins
only because earlier runs had left a libpq behind in the real installation
directory for the binaries to fall back on. The workflow installs first and then
runs `make installcheck` against a server started from the installed tree, whose
binaries resolve their libraries through absolute `install_name` references and
need no `DYLD_*` at all.

MIT Kerberos is likewise built without running its own `make check`; the reasons
are in a comment in `krb5-macos.yml`.

### Windows

PostgreSQL 17 and above are built with Meson, and 16 and below with the older
MSVC scripts, since that is where upstream moved.

GSSAPI is enabled, and MIT Kerberos is built and bundled like any other
dependency, but only from PostgreSQL 18 onwards. Building libpq on Windows with
both OpenSSL and GSSAPI turned on fails to compile on the older branches,
because `<wincrypt.h>` defines `X509_NAME` and clobbers OpenSSL's typedef;
upstream fixed that with the commit adding `src/include/libpq/pg-gssapi.h`,
which is in 18 and master but was not back-patched. PostgreSQL 17 and earlier
are therefore built with GSSAPI disabled, and the `postgresql-dev-windows.yml`
build of master has it enabled unconditionally. See
[this thread](https://www.postgresql.org/message-id/CA%2BOCxoxwsgi8QdzN8A0OPGuGfu_1vEW3ufVBnbwd3gfawVpsXw%40mail.gmail.com)
for the background.

## Not built here

Perl, Python and TCL are not built or pinned by this repository on either
platform, so PL/Perl, PL/Python and PL/Tcl are not part of what these workflows
supply. On Windows the pre-18 MSVC builds set them explicitly to `undef`. Adding
them would mean a workflow each, and on Windows there is no room left in the
orchestrator's call tree.

Bonjour, LLVM and readline are not configured on either platform either, though
the situation differs between them. Bonjour and readline are Windows problems
that largely evaporate on macOS, where Bonjour is native and libedit ships with
the system; neither is currently requested by the macOS `configure` line. LLVM,
and so JIT compilation, is absent from both.

ICU is not a gap so much as a deliberate difference. The Windows side builds it
and links against it, named explicitly in `config.pl` for the MSVC builds and
picked up through pkg-config for the Meson ones, and ships the ICU DLLs
alongside the binaries; the macOS build passes `--without-icu` and does not
build it at all.

## Roadmap

### ARM64 Windows

Windows on ARM64 has been asked for
([dpage/winpgbuild#17](https://github.com/dpage/winpgbuild/issues/17)) and is
not far-fetched, since GitHub now offers free `windows-11-arm` hosted runners to
public repositories, so the machines to build it on are already available at no
cost.

Nothing here builds it yet, and this is not a commitment to. What has been done
is to leave the door open: the naming scheme carries the platform and the
architecture as separate, explicit components on both platforms, so an ARM64
Windows build slots in as `openssl-windows-arm64-latest` beside the existing
`openssl-windows-x86_64-latest` without renaming anything or breaking a single
consumer. Getting that wrong would have been free to fix today and expensive
once anything downstream had started resolving these tags.

The obstacle is not naming but the call-tree cap described above: a second
Windows architecture means a second set of leaves, and `build-all-windows.yml`
has no room for them. Whoever takes it on will need to split that orchestrator
first, most likely one per architecture in the way the platforms are split
today.
