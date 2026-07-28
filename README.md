# TR Apps — Update Host

Public Sparkle update host for Tommy Rush's standalone macOS apps.

Each app has its own folder with an `appcast.xml` feed and signed `.zip` downloads,
served via **GitHub Pages** at `https://tommy-rush.github.io/tr-apps-updates/<app>/appcast.xml`.

## Apps

| App | Feed URL |
|-----|----------|
| TR Key/BPM | `https://tommy-rush.github.io/tr-apps-updates/trkeybpm/appcast.xml` |
| PT Arranger | `https://tommy-rush.github.io/tr-apps-updates/ptarranger/appcast.xml` (reserved) |
| TR Rack | `https://tommy-rush.github.io/tr-apps-updates/trrack/appcast.xml` (reserved) |

## Signing

All downloads are EdDSA (ed25519) signed with the Sparkle key whose **public** half is:

```
vnk2NfEHVQaozUBcYIf2pbmtXtKYoA51/QeJSjxu+sg=
```

The private key lives in the macOS login Keychain on Tommy's MBP and is backed up in
1Password ("Sparkle EdDSA Key - TR Apps") + `~/.tr-apps-sparkle/`. Apps embed the public
key as `SUPublicEDKey` and **refuse any update whose signature does not verify**.

## Session Key has TWO feed files. One is generated. Read this before editing either.

`session-key/appcast.xml` is **canonical** — it is the only Session Key feed a human edits.

`trkeybpm/appcast.xml` is **GENERATED** — never hand-edit it. It exists because every build
up to and including b251, which is the entire install base, bakes the OLD feed path:

```
b251 Info.plist SUFeedURL = https://tommy-rush.github.io/tr-apps-updates/trkeybpm/appcast.xml
```

(verified at runtime — the app itself prints `updater feed: …/trkeybpm/appcast.xml`). That URL
301s to `update.tommy-rush.com/trkeybpm/appcast.xml`, which is a **separate file** from
`session-key/appcast.xml`. A release published only into `session-key/appcast.xml` therefore
reaches **zero** installed copies. The SUFeedURL moved to `/session-key/` at b256, so once a
user takes ONE update off the trkeybpm feed they migrate to the canonical feed permanently.

Keep them in sync with the generator, which also gates the feed:

```
tools/mirror-appcast.sh                 # regenerate the mirror
tools/mirror-appcast.sh --check         # verify; exit 1 on drift  (run before every push)
tools/mirror-appcast.sh --from-head     # generate from HEAD, ignoring staged edits
tools/mirror-appcast.sh --verify-sigs   # also EdDSA-verify EVERY enclosure (fail-closed)
```

Fail-closed gates it applies to the canonical feed before mirroring: well-formed XML with no
DTD/ENTITY; no placeholder `pubDate` (`REPLACE-AT-PUSH`, empty, `TBD`/`TODO`/…) and every
`<item>` carries one; every `<enclosure>` https with a non-empty `sparkle:edSignature` and
`length` — parsed as XML, not grepped, so `url = 'http://…'` cannot slip through. With
`--verify-sigs`, a missing artifact, a length mismatch, or a failed signature is a FAILURE,
never a skip.

Its placeholder check is a **placeholder detector, not an approval gate**: an item staged with
a real date looks exactly like an approved one. Approval is the release gate (owner tests and
approves the copy), not this script.

Artifacts (zip/dmg) are committed **once**, into `session-key/` — both feeds already point
their enclosures at that directory, so only the XML is mirrored.

## Releasing a new version (per app)

1. Build + notarize + staple the `.app`.
2. `ditto -c -k --keepParent "App.app" App-X.Y.zip` (or build a notarized+stapled `.dmg`).
3. `sign_update <artifact>` → copy the `sparkle:edSignature` + `length`.
4. Add a new `<item>` to that app's `appcast.xml` (newest first). For Session Key that means
   `session-key/appcast.xml` and **only** that file.
5. Session Key only: `tools/mirror-appcast.sh` then `tools/mirror-appcast.sh --check`.
6. Commit the artifact + both xml files, push.
