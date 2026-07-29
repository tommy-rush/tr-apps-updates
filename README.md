# TR Apps — Update Host

Public Sparkle update host for Tommy Rush's standalone macOS apps.
Served by **GitHub Pages** at the custom domain in `CNAME`:

```
https://update.session.am/
```

## THERE IS EXACTLY ONE SESSION KEY FEED

```
https://update.session.am/session-key/appcast.xml
```

That file is the only Session Key appcast in this repo. Do not add a second one.
There used to be two (`session-key/appcast.xml` and `trkeybpm/appcast.xml`) kept in sync by a
generator, and they drifted: one carried `github.io` enclosure URLs while the other carried
`update.tommy-rush.com` ones. The second file and its generator are gone.

| App | Feed |
|-----|------|
| Session Key | `https://update.session.am/session-key/appcast.xml` |
| Session Arranger for Pro Tools | `https://update.session.am/ptarranger/appcast.xml` |

## Every older build still reaches that one feed. Here is exactly how.

A shipped build bakes `SUFeedURL` into `Info.plist` and it can never be changed for a copy that
is already installed. Three different URLs are in the wild. These were read out of the shipped
bundles, not out of documentation:

| shipped builds | baked SUFeedURL |
|---|---|
| TR KeyBpm 2.3.0–2.5.0, Session Key 1.0, 1.0.1, 1.0.4 b251 | `https://tommy-rush.github.io/tr-apps-updates/trkeybpm/appcast.xml` |
| Session Key b256 and later (b291, b293, …) | `https://update.tommy-rush.com/session-key/appcast.xml` |
| Session Arranger for Pro Tools (b55) | `https://tommy-rush.github.io/tr-apps-updates/ptarranger/appcast.xml` |

They resolve like this:

```
tommy-rush.github.io/tr-apps-updates/<path>
  --301-->  update.session.am/<path>                  GitHub Pages custom-domain redirect

update.tommy-rush.com/<path>
  --301-->  update.session.am/<path>                  Cloudflare rule, tommy-rush.com zone

update.session.am/trkeybpm/appcast.xml
  --301-->  update.session.am/session-key/appcast.xml Cloudflare rule, session.am zone
```

Sparkle follows 301s, so every generation lands on the one feed.

### The two Cloudflare rules are load-bearing. Deleting either orphans installed copies.

- zone `tommy-rush.com` (`377bcb59cc3d02be7d246f6061a1606b`), ruleset
  `8591e4b133e541bca7f7aaae02da16ec`, rule `8fc3a2c4b0084c06b9ded93c7bf39518`
  — `update.tommy-rush.com` → `update.session.am`, path preserved.
- zone `session.am` (`1b16a73bc8e5badc35ba938ee807ac2d`), phase
  `http_request_dynamic_redirect`
  — `update.session.am/trkeybpm/appcast.xml` → `/session-key/appcast.xml`.

`update.tommy-rush.com` must also keep its proxied Cloudflare DNS record. The redirect runs at
the edge before any origin fetch, so it works even though GitHub Pages no longer answers for
that hostname.

## Signing

All downloads are EdDSA (ed25519) signed. Public half:

```
vnk2NfEHVQaozUBcYIf2pbmtXtKYoA51/QeJSjxu+sg=
```

The private key lives in the macOS login Keychain on Tommy's MBP and is backed up in
1Password ("Sparkle EdDSA Key - TR Apps") + `~/.tr-apps-sparkle/`. Apps embed the public
key as `SUPublicEDKey` and **refuse any update whose signature does not verify**.

Verify an enclosure signature without the private key:

```sh
python3 verify-ed.py "$PUBKEY_B64" "$SIG_B64" path/to/App.zip
```

```python
# verify-ed.py
import base64, subprocess, sys, os, tempfile
pub = base64.b64decode(sys.argv[1]); sig = base64.b64decode(sys.argv[2])
spki = bytes.fromhex('302a300506032b6570032100') + pub   # Ed25519 SubjectPublicKeyInfo
b = base64.b64encode(spki).decode()
pem = "-----BEGIN PUBLIC KEY-----\n" + "\n".join(b[i:i+64] for i in range(0, len(b), 64)) + "\n-----END PUBLIC KEY-----\n"
with tempfile.TemporaryDirectory() as d:
    p = os.path.join(d, 'k.pem'); open(p, 'w').write(pem)
    s = os.path.join(d, 's.bin'); open(s, 'wb').write(sig)
    sys.exit(subprocess.run(['openssl', 'pkeyutl', '-verify', '-pubin', '-inkey', p,
                             '-rawin', '-in', sys.argv[3], '-sigfile', s]).returncode)
```

Run it against a known-bad signature too. A verifier that has never returned failure has not
been shown to discriminate.

## Releasing a new version

1. Build, notarize, staple the `.app`.
2. `ditto -c -k --keepParent "App.app" App-X.Y.zip` (or a notarized and stapled `.dmg`).
3. `sign_update <artifact>` → take the `sparkle:edSignature` and `length`.
4. Add one `<item>` to that app's appcast, newest first. Never truncate the older items;
   they are the downgrade and history path.
5. Commit the artifact and the xml together, push.

Enclosure URLs in new items must be `https://update.session.am/...`.

Owner approval is the release gate. A staged item with a real `pubDate` looks exactly like an
approved one, so no script can tell them apart.
