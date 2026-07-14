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

`package.json` only, so that `npm install -g git+https://github.com/Soborbo/mxroute-cli.git#<sha>`
works — upstream never had to support installing from git, because it publishes to npm:

- `prepare`: `husky` -> `npm run build`. `dist/` is gitignored, so a git install produced a
  package with no compiled output. `prepare` is the hook npm runs after cloning.
- `postbuild`: `chmod +x ...` -> equivalent `node -e` call. `chmod` does not exist in the
  Windows shell npm uses, which broke the build there.

## Updating

Do not run `mxroute update` — it would pull upstream npm and undo the pin.

```bash
git fetch upstream && git log --oneline HEAD..upstream/main   # review every commit
git merge upstream/main                                       # then re-run the audit above
npm install -g "git+https://github.com/Soborbo/mxroute-cli.git#<new-sha>"
```
