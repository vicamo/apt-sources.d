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

    {series}    selected suite alias: devel for target-tracking devel sources,
                concrete codename for LTS or pinned-codename policies
    {version}   Ubuntu version number (24.04)
    {suite}     suite chosen by X-Pool-Suite-Policy
    {archive}   archive mirror ( old-releases for EOL series )
    {security}  security mirror ( old-releases for EOL series )

## Attributes

    # X-Pool-Scope: ubuntu      expand across every Ubuntu series directory
    # X-Pool-Scope: fixed       emit verbatim to one fixed directory
    # X-Pool-Output: <dir>      target dir for a fixed-scope file (e.g. stable)
    # X-Pool-Suite-Policy: codename | lts | <concrete codename>
          codename  {suite} = the target codename (default)
          lts       {suite} = the target codename if it is an LTS, else the
                    latest LTS at or before it (for upstreams that only ship
                    for LTS releases, e.g. hashicorp)
          <codename> pin {series}, {suite}, and {version} to that release
    # X-Pool-Assume-Available: yes | no
          bypass availability checks for this definition (default: no)

For every public third-party Ubuntu definition (including PPAs), rebuild probes
`dists/<codename>/Release` from the target release backwards and uses the newest
published candidate among releases at least as new as the oldest pool target.
The candidate supplies `{version}`. With the default `codename` policy, sources
under `devel/` keep `devel` in `{series}` and `{suite}`; concrete output
directories use the selected candidate's codename. Availability checks use the
candidate's concrete suite without changing the devel output alias.
The `lts` policy selects only LTS candidates and keeps the selected LTS codename
even under `devel/`. A pinned-codename policy likewise keeps its concrete suite
and version. Literal `Suites:` values in a definition are preserved and checked
as written. Fixed-scope definitions remain verbatim. These rules also apply
when `X-Pool-Assume-Available: yes` bypasses repository checks.
Ubuntu archive definitions containing `{archive}` or `{security}` retain their
release-specific suites and the `devel` alias.

Only HTTP 404 triggers fallback. Network errors, malformed metadata and other
HTTP errors (including 401/403) stop the rebuild before generated files are
written. Fixed-scope definitions remain verbatim and are not remapped to
Ubuntu releases.
If no candidate is published, rebuild emits a disabled source (`Enabled: no`)
and reports it, keeping apt from requesting an unavailable repository.

Add `# X-Pool-Assume-Available: yes` to an individual definition to bypass its
repository checks and assume the target release is published (or the latest
LTS under LTS policy). This can emit unpublished repositories; it is an
explicit override.
All definitions are generated, including those requiring credentials. Use
`X-Pool-Assume-Available: yes` for these repositories to skip availability
requests; credentials remain the user's responsibility. Ubuntu's `devel`
discovery still fetches archive metadata.

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
