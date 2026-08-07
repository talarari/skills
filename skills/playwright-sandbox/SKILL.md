---
name: playwright-sandbox
description: Make Playwright/Chromium load pages inside a Claude Code cloud sandbox, where every navigation otherwise fails with net::ERR_CONNECTION_RESET. Use whenever browser automation is needed in a remote/cloud container and page.goto fails, hangs, or returns an error page — including when the browser "can't reach the internet" but curl works fine, or when a site returns 403 only to the browser. TRIGGER when the user says "playwright doesn't work", "the browser can't load any page", "ERR_CONNECTION_RESET", "Executable doesn't exist at /opt/pw-browsers", or asks to set up browser automation in a cloud session.
allowed-tools: [Bash, Read, Write, Edit]
---

# Playwright in a Claude Code cloud sandbox

Browser automation is broken by default here. Apply the fix below **before** debugging anything —
it takes one command, and the failure it prevents looks like a dozen other problems.

## Apply the fix

Run this as-is. It detects the browser, derives its policy directory, and writes the policy:

```bash
CHROME=$(find /opt/pw-browsers ~/.cache/ms-playwright -maxdepth 3 -type f -name chrome 2>/dev/null | head -1)
POLICY=$(strings "$CHROME" 2>/dev/null | grep -oE '^/etc/(opt/)?chrom[a-z_]*/policies' | sort -u | head -1)
mkdir -p "$POLICY/managed" && echo '{"EncryptedClientHelloEnabled": false}' > "$POLICY/managed/ech.json"
echo "policy: $POLICY/managed/ech.json   browser: $("$CHROME" --version)"
```

Then launch the browser **already in the image**, by path — not Playwright's default:

```js
const browser = await chromium.launch({ executablePath: '/opt/pw-browsers/chromium' });
```

Both parts are required. The full binary without the policy resets; the headless shell with the
policy resets. Playwright's default build is `chrome-headless-shell`, which ignores managed policies
entirely. `/opt/pw-browsers/chromium` is a symlink to the full `chrome` binary, so it reads the
policy — do not repoint it at `chromium_headless_shell-*`.

### Do not use `channel: 'chromium'`, and do not run `playwright install`

`channel: 'chromium'` resolves to `chromium-<revision>`, where the revision is pinned by *your
installed Playwright version* — not by what the image actually has. A fresh `npm install playwright`
pulls a newer Playwright than the image was built against, so the launch dies on a path that was
never there:

```
Executable doesn't exist at /opt/pw-browsers/chromium-1234/chrome-linux64/chrome
```

That is a version mismatch, not a missing browser: 1234 is what Playwright 1.62 pins, 1194 is what
the image ships. `npx playwright install chromium` "fixes" it by downloading a second browser, which
is slow, may be blocked by egress policy, and is explicitly discouraged in these sandboxes. The
`executablePath` symlink sidesteps the pinning and keeps working when either side moves.

### Complete working example

Copy-paste runnable. `npm install playwright` first (the package only — the browser is already
there), then `node script.js`.

```js
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch({ executablePath: '/opt/pw-browsers/chromium' });
  const context = await browser.newContext({
    // Not optional on most real sites — see "403 to the browser only" below.
    userAgent: 'Mozilla/5.0 (Linux; Android 14; Pixel 7) AppleWebKit/537.36 ' +
               '(KHTML, like Gecko) Chrome/141.0.0.0 Mobile Safari/537.36',
  });
  const page = await context.newPage();

  const r = await page.goto('https://example.com/', { waitUntil: 'domcontentloaded' });
  console.log(r.status(), await page.title());   // expect 200 / "Example Domain"

  await page.screenshot({ path: 'shot.png' });
  await browser.close();
})();
```

### Verify

Screenshot it and **look at the image** with the Read tool. Chrome renders network errors as an
ordinary page whose `<title>` is just the hostname, so a title check alone passes on an error page.

To confirm the policy is what is doing the work, move it away and watch the same script fail:

```bash
mv /etc/chromium/policies/managed/ech.json /tmp/ech.bak && node script.js   # net::ERR_CONNECTION_RESET
mv /tmp/ech.bak /etc/chromium/policies/managed/ech.json && node script.js   # 200 Example Domain
```

## Make it survive a new container

The policy lives in `/etc`, outside the repo, so it dies with the container. To stop every future
session from rediscovering this, add it to a `SessionStart` hook.

In `.claude/settings.json` (create the `hooks` block if absent):

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume",
        "hooks": [
          { "type": "command", "command": "bash \"$CLAUDE_PROJECT_DIR\"/scripts/cloud-setup.sh" }
        ]
      }
    ]
  }
}
```

And in that script:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Only ever "true" in a cloud session, so this never touches /etc on a local machine.
[ "${CLAUDE_CODE_REMOTE:-}" = "true" ] || exit 0

setup_browser_policy() {
  local chrome policy_dir
  chrome=$(find /opt/pw-browsers ~/.cache/ms-playwright -maxdepth 3 -type f -name chrome 2>/dev/null | head -1)
  [ -n "$chrome" ] || return 0

  policy_dir=$(strings "$chrome" 2>/dev/null | grep -oE '^/etc/(opt/)?chrom[a-z_]*/policies' | sort -u | head -1)
  [ -n "$policy_dir" ] || return 0

  mkdir -p "$policy_dir/managed" 2>/dev/null || return 0
  echo '{"EncryptedClientHelloEnabled": false}' > "$policy_dir/managed/ech.json" 2>/dev/null || return 0
}

# Never let a browser fix stop a session from starting.
setup_browser_policy || true
```

Three things that matter if you adapt it: gate on `CLAUDE_CODE_REMOTE` so a developer's own `/etc`
is never modified; return 0 on every branch and add `|| true` at the call site, because under
`set -e` a missing `strings` or a read-only `/etc` would otherwise abort the whole hook; and put it
*after* dependency installation, since deps matter more than optional browser support.

## Why — read only if the fix didn't work

The egress proxy sends a TCP RST on any TLS handshake carrying the **ECH extension** (`0xfe0d`,
Encrypted Client Hello), which Chrome sends as GREASE by default. The CONNECT tunnel opens fine
(`200 Connection Established`) and the reset lands the instant the ClientHello arrives. `curl`,
`openssl` and Node reach the same hosts through the same proxy only because none of them send ECH.

So this is **not** a network-policy block, **not** a broken CONNECT (a public issue claims the proxy
lacks CONNECT support — it doesn't), and **not** a CA-trust problem. Never "fix" it by disabling TLS
verification, unsetting `HTTPS_PROXY`, or switching to an HTTP-library scraper.

Two traps if it still resets:

- **`--disable-features=EncryptedClientHello` does not work.** The extension is still on the wire.
  It must be the policy file.
- **The policy path is image-specific and a wrong path fails silently**, which is indistinguishable
  from the fix not working. Plain Chromium reads `/etc/chromium/policies`; Chrome for Testing reads
  `/etc/opt/chrome_for_testing/policies`. That is why the command above derives it from the binary
  instead of hardcoding. Check what actually landed:
  `ls -la "$POLICY/managed/" && "$CHROME" --version`

## Second problem — 403 to the browser only

Once pages load, CDNs and WAFs (CloudFront especially) still block the default headless User-Agent,
which contains the literal string `HeadlessChrome`. The tell is `curl` getting 200 while the browser
gets 403. Set a real UA on the context:

```js
const ctx = await browser.newContext({
  userAgent: 'Mozilla/5.0 (Linux; Android 14; Pixel 7) AppleWebKit/537.36 ' +
             '(KHTML, like Gecko) Chrome/141.0.0.0 Mobile Safari/537.36',
});
```

For `curl` against the same hosts, send a browser UA **and** `--compressed`, or you get raw gzip
bytes.

## Upstream

The real bug is the proxy resetting on ECH; the policy file is a workaround. A permanent fix belongs
in the sandbox image or the proxy itself, so it works out of the box in every repo rather than
needing this skill.
