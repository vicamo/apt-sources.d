# Pool source definitions

`pool/<series>/…` holds the actual deb822 `.sources` files apt consumes.
Those trees are **generated** — do not edit them by hand. Edit the
definitions here and run `.local/bin/rebuild`.

## Layout

    .local/definitions/        source of truth (this directory)
    .local/bin/rebuild         regenerate pool/<series> + .local/links
    .local/bin/check           validate structure + idempotency

A definition is a normal `.sources` stanza plus `# X-Pool-*` comment
attributes (stripped from the output) and `{…}` placeholders.

## Placeholders

    {series}    series codename (noble); "devel" for the dev-series dir
    {version}   Ubuntu version number (24.04)
    {suite}     suite chosen by X-Pool-Suite-Policy
    {archive}   archive mirror ( old-releases for EOL series )
    {security}  security mirror ( old-releases for EOL series )

## Attributes

    # X-Pool-Scope: ubuntu      expand across every Ubuntu series directory
    # X-Pool-Scope: fixed       emit verbatim to one fixed directory
    # X-Pool-Output: <dir>      target dir for a fixed-scope file (e.g. stable)
    # X-Pool-Suite-Policy: codename | lts
          codename  {suite} = the target codename (default)
          lts       {suite} = the target codename if it is an LTS, else the
                    latest LTS at or before it (for upstreams that only ship
                    for LTS releases, e.g. hashicorp)

Release history comes from `ubuntu-distro-info --all -f`, supplemented by
`https://archive.ubuntu.com/ubuntu/dists/devel/Release` on every rebuild.
The archive's `Codename` and `Version` determine which release `devel` aliases;
`devel/` remains a separate output directory even if its concrete codename
already has a directory. Future entries in distro-info do not advance the alias.
Concrete series directories missing from local distro-info are supplemented
using their own `dists/<series>/Release` files. Archive metadata supplies the
version and Ubuntu's even-year April release rule supplies LTS status.
Rebuild requires network access and fails before writing output if release
metadata cannot be fetched or is invalid, rather than guessing the alias.
EOL series (archived to old-releases) are listed in
`EOL_SERIES` inside `rebuild` — distro-info data on this host is too stale to
detect EOL automatically.

## Covering the four pain points

1. Different suites per system  — each series dir is regenerated from one
   definition, so adding a series is a single `mkdir <series>` +
   `rebuild`.
2. Different enabled sets        — enable per host by symlinking the wanted
   `.local/links/*.sources` into `/etc/apt/sources.list.d/`.
3. Non-codename suites/URIs      — `{version}` (microsoft-prod) and
   `Suite-Policy: lts` (hashicorp) express releases that don't follow the
   codename.
4. Repos not tracking Ubuntu     — `Scope: fixed` (jessie/slack, stable/*,
   mozilla) pin a fixed suite with no series expansion.

## Workflow

    edit .local/definitions/…
    .local/bin/rebuild          # regenerate everything
    .local/bin/check            # structure + drift (CI runs this)
    git add -A && git commit
