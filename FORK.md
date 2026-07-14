# Fork notes (Soborbo/mxroute-cli)

Fork of [t-rhex/mxroute-cli](https://github.com/t-rhex/mxroute-cli), pinned so that upstream
changes only reach our machines when we deliberately merge and re-audit them.

Upstream is a single-maintainer project (0 stars, 0 other forks) and ships a self-update
command that runs `npm install -g mxroute-cli@latest`. Our `config.json` holds a DirectAdmin
Login Key, a Cloudflare DNS:Edit token and a mailbox password, so a compromised upstream
release would be costly. Installing from this fork removes that automatic path.

## Audit of upstream v1.2.3 (commit 60c9b4e), 2026-07-14

- Rebuilt all 102 `dist` files from source: byte-identical to the published npm tarball
  (after line-ending normalisation). The npm package is a faithful build of the public source.
- No `preinstall` / `install` / `postinstall` scripts in the package or in any of its 176
  transitive dependencies.
- Every outbound host in the code is either MXroute's own infrastructure or a DNS provider API
  (Cloudflare, Porkbun, Hetzner, DigitalOcean, GoDaddy, Vercel). No unexplained endpoints.
- Slack / Discord / Telegram notification targets are supplied by the user at runtime, not
  hardcoded.
- IMAP talks raw TLS straight to `{server}.mxrouting.net:993`.
- Sending is **not** direct SMTP: `mxroute send` POSTs `{server, username, password, ...}` as
  JSON to `https://smtpapi.mxroute.com/`. That host is a subdomain of the vendor's own domain,
  so the mailbox password goes to MXroute — who already hold it. Accepted, but it is the reason
  a dedicated sending mailbox is preferable to a human's primary account.
- `npm audit` reports issues in `hono` (via the official MCP SDK), `shell-quote` (via
  `ink` -> `react-devtools-core`), `lodash`, `ws`, `path-to-regexp` and `fast-uri`. All are
  server-exposure or DoS class and unreachable from stdio MCP + outbound-only CLI use.

## Changes on top of upstream

`package.json` only:

- `prepare`: `husky` -> `npm run build`. `dist/` is gitignored, so the repo has no compiled
  output of its own.
- `postbuild`: `chmod +x ...` -> equivalent `node -e` call. `chmod` does not exist in the shell
  npm uses on Windows, which broke the build there.

## Installing

`npm install -g git+https://github.com/Soborbo/mxroute-cli.git#<sha>` does **not** work: npm runs
`prepare` before the runtime dependencies are present, so `tsc` fails on `Cannot find module
'chalk'`. Upstream never had to support installing from git, because it publishes to npm. Build
and install the tarball instead — same pin, and it is byte-for-byte what `npm publish` would ship:

```bash
git clone https://github.com/Soborbo/mxroute-cli.git && cd mxroute-cli
npm ci && npm run build && npm pack
npm install -g ./mxroute-cli-<version>.tgz
```

Verify the global install is this fork and not the registry copy — the registry copy has
`"prepare": "husky"`:

```bash
node -p "require(require('child_process').execSync('npm root -g').toString().trim() + '/mxroute-cli/package.json').scripts.prepare"
# -> npm run build
```

## Updating

Do not run `mxroute update` — it runs `npm install -g mxroute-cli@latest`, which would replace
this fork with the upstream registry build and silently undo the pin.

```bash
git fetch upstream && git log --oneline HEAD..upstream/main   # review every commit
git merge upstream/main                                       # then re-run the audit above
npm ci && npm run build && npm pack && npm install -g ./mxroute-cli-<version>.tgz
```
