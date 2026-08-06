# Security Policy

## Reporting a vulnerability

**Email [anthony@thebananastandard.xyz](mailto:anthony@thebananastandard.xyz).**

Please do not open a public issue for a security problem.

You should get a reply within 48 hours. If it is confirmed, you will get a fix
timeline and credit in the release notes unless you would rather not be named.

> If a previous version of this file pointed you at GitHub Security Advisories on
> `The-Banana-Standard/canopy`, that link was broken: the repository is private,
> so it 404s for anyone who does not already have access — which is everyone who
> would have needed it. Email works.

Canopy is maintained by The Banana Standard LLC. There is no bug bounty. It is a
small project and you will be talking to the person who wrote the code.

## What Canopy is, which is most of the threat model

Canopy is a desktop terminal multiplexer for agentic CLIs. **It spawns shells and
agent binaries with your user's full privileges**, in whichever directory you
point them at, and text sent to a terminal reaches that process's stdin.

That is the product, not a flaw. The consequence worth stating plainly: **the
webview is a trust boundary, not a sandbox.** Anything that achieves script
execution in the renderer is equivalent to local code execution as you. Reports
that establish renderer script execution are therefore high severity even when
the immediate effect looks cosmetic.

Mitigations that exist: `default-src 'self'` with `script-src 'self'`,
`withGlobalTauri: false`, no `innerHTML`/`eval`/`new Function` anywhere in the
frontend, and no telemetry, analytics or crash reporter of any kind.

## Where the code lives

| Component | In scope | Notes |
|---|---|---|
| Desktop app (Rust + webview) | Yes | The bulk of it |
| `cloud/hub` | Yes | WebSocket hub + HTTP control plane, deployed |
| `cloud/account` | Yes | Sign-in; holds the Clerk keys |
| `cloud/web` | Yes | Read-only thread viewer, loaded by untrusted invite holders |
| `cloud/site` | Yes | Landing site; serves both updater manifests |
| Third-party agent CLIs | No | Report to Anthropic, OpenAI, Google, GitHub |

The source repository is private. If you need to point at code, a file and line
number from a release artifact — or a plain description — is fine.

## Everything that leaves your machine

There are exactly three network destinations, and nothing else:

1. **The update manifest.** An anonymous HTTPS GET of `updater.json`, first at
   `canopyagents.com` and then at `canopy-site-xi.vercel.app` if that fails.
   Automatic — delayed on launch, then daily — and switchable in Settings. It
   carries no identifier, no account and no query string; the only fingerprint
   is the updater plugin's own `User-Agent`.
2. **The release host,** `github.com/The-Banana-Standard/canopy-releases`, and
   only after you press Update.
3. **The sync hub** — over `wss://` when you share or join a thread, and over
   `https://` when you sign in, sign out, or open Settings while signed in.

Anything else that leaves is a URL handed to your default browser. The shell
plugin restricts those to `http(s)`, `mailto:` and `tel:` — no `file://`, no
custom schemes.

Sharing a thread sends the display name you chose. Signing in sends a device
label you can edit, and associates this machine with the account you approved in
your own browser.

## Credentials and data at rest

Credentials live in your **OS keyring**, in four allowlisted slots: three AWS
provider secrets and the sync/device token. Any other key is refused. The device
token is stored on the hub only as `sha256(token)`, so a database dump there
hands nobody a working credential.

The local SQLite database is **unencrypted** and holds **no credentials** — but
"no credentials" is the wrong reassurance on its own. It also holds your project
paths, indexed file paths, extracted symbol names and signatures, README text,
and summaries of your agent sessions. File *contents* are not stored, and the
indexer honours `.gitignore` (which is why `.env` files are absent from it — that
is deliberate and must stay that way). A project that is not a git repository
has no `.gitignore` to honour.

`localStorage` holds terminal tab state and notification settings. No
credentials. File tabs are deliberately never persisted, so unsaved secrets never
land there.

## Known and intended — please don't report these as bugs

**An invite link is a bearer credential.** Anyone holding it can join that one
thread, at the role the invite grants, with no account and no Canopy install.
That is the design. Credentials ride the URL fragment so they never reach a
server log or a `Referer`. What a guest *cannot* do is the actual security
boundary: reach another thread, mint invites, change their own role, or
impersonate the agent. Invites are revocable by the host. **Report failures of
those properties** — not the property that a link is sufficient.

**Dangerous mode exists and is opt-in.** It launches an agent with its permission
prompts disabled. Such tabs are never persisted, and a dangerous thread can never
be shared, because it has no consent gate to share.

**The secret scrubber is best-effort.** When you share a thread, published events
are transformed — host paths relativized, tool payloads optionally hidden,
credential-shaped strings redacted by pattern plus an entropy backstop. It is a
mitigation, not a guarantee. A credential shape it misses is worth telling us
about, but it is a gap in a best-effort filter rather than a broken promise.

**The dev auth secret in `cloud/hub/scripts/mint-token.mjs` is public on
purpose.** Production refuses to boot if the real secret is unset or equal to it,
and the dev minter throws for any non-loopback hub.

## Known and *not* intended — already on the list

Stated here so you can skip the write-up, and so nobody assumes they are secret:

- **The SQL plugin capability is unscoped.** The webview holds `sql:allow-load`
  and `sql:allow-execute` with no path scope. The app only ever opens its own
  database, but the capability permits more than that — including writing the
  `projects` table that the file editor's containment check reads.
- **The CSP permits any loopback port.** A join link naming `ws://127.0.0.1:<port>`
  will open a socket to a local process. The account token is deliberately
  withheld on that path.
- **`read_claude_md` and the workspace indexer** take a caller-supplied directory
  without checking it against the projects table.

New detail on any of these is still welcome — a working exploit chain
particularly so.

## The updater is the most sensitive thing here

Update bundles are verified against a **single minisign public key compiled into
the application** before anything is installed, and the install is refused unless
the artifact URL lives under the release the manifest names, on the one release
host every published Canopy has ever used. That second check closes a downgrade
that needs no key compromise at all — anyone able to write the manifest could
otherwise re-serve a genuinely-signed *older* bundle under a higher version
number.

The webview is granted none of the updater plugin's own commands; the entire
reachable surface is three Rust commands.

**A signing-key compromise is the worst outcome in this project.** If you find
anything that touches the signing key, the manifest, the release host or the
verification path, say so in your first sentence and it will be treated as
critical.

## Supported versions

| Version | Supported |
|---------|-----------|
| 0.9.x   | Yes       |
| < 0.9   | No — please update |

Canopy auto-updates on macOS (Apple Silicon). On other platforms, update from
[canopyagents.com](https://canopyagents.com).
