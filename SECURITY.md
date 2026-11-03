# Security Notes

This is a fork of [`clauderic/react-sortable-hoc`](https://github.com/clauderic/react-sortable-hoc),
which upstream has deprecated in favour of [`dnd-kit`](https://github.com/clauderic/dnd-kit).
**No upstream security patches are coming**, so the notes below are the current state of this fork.

## Fixed in this repo

- **`package-lock.json` is now committed.** Previously every `npm install` re-resolved
  ~1200 packages from caret ranges, so no two installs were guaranteed identical and a
  compromised transitive release could land silently. All entries are pinned with
  `integrity` hashes and resolve to `registry.npmjs.org`.
- **The dev server binds to loopback.** `server.js` previously listened on all
  interfaces. It relies on `webpack-dev-middleware@1.x`, which has a known path
  traversal ([GHSA-wr3j-pwj9-hqq6](https://github.com/advisories/GHSA-wr3j-pwj9-hqq6)),
  so on a shared network it exposed arbitrary file reads. Binding to `127.0.0.1`
  removes the network path.
- **CI installs with `npm ci --ignore-scripts`** on a supported Node LTS, instead of
  an unpinned `npm install` on Node 5 (EOL June 2016).

## Known, unfixed

### Dependency vulnerabilities

`npm audit` reports **170 advisories (67 critical, 47 high)**. These are almost entirely
build- and dev-time dependencies; none are reachable from the library code shipped to
consumers (`dist/`). The runtime source has no `eval`, no `innerHTML`/`dangerouslySetInnerHTML`,
and no `child_process` usage.

They are not patched because the toolchain is pinned to 2017: `npm audit fix --force`
cascades into a webpack 1 -> 5 and Babel 6 -> 7 migration. The root advisories are
`babel-traverse` (arbitrary code execution at build time), `loader-utils`, `minimist`,
`tar`, `underscore`, and `webpack-dev-middleware`.

### Install scripts

Four packages run install scripts: `core-js`, `es5-ext`, `fsevents`, `node-sass`.
`--ignore-scripts` is safe in CI because CI only lints. **It is not sufficient for a
full local build** — `node-sass@3.x` needs its postinstall step to fetch a native
binding, and it is long past EOL. Use `npm ci` without the flag locally if you must
build, and prefer a container.

### Recommended install

```sh
npm ci --ignore-scripts   # lint / CI
npm ci                    # only when you need to build; runs native postinstalls
```

## Items to verify outside this repo

These cannot be checked from the source tree and are the likeliest paths to real access:

1. **Travis CI OAuth grant.** `.travis.yml` never held encrypted secrets, but the
   GitHub OAuth grant issued when Travis was connected may still be active with write
   scope. Travis `.org` is defunct. Verify at
   `github.com/organizations/<org>/settings/oauth_application_policy` and revoke if present.
2. **GitHub Pages.** The `gh-pages` branch contains an accidental 66k-file `node_modules`
   commit (upstream, 2017) including prebuilt `.node` native binaries for macOS x64.
   It contains no credentials, but if Pages is still enabled it serves those publicly.
3. **Downstream pinning.** This fork never renamed the package: `package.json` still
   reads `react-sortable-hoc@0.6.8` pointing at the upstream repo, so it is consumed via
   git reference rather than npm. Confirm consumers pin a tag or commit SHA, not a branch.

## Reporting

Report suspected vulnerabilities to the repository owners directly rather than opening a
public issue.
