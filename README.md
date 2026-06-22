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

## Releasing a new version (per app)

1. Build + notarize + staple the `.app`.
2. `ditto -c -k --keepParent "App.app" App-X.Y.zip`
3. `sign_update App-X.Y.zip` → copy the `sparkle:edSignature` + `length`.
4. Add a new `<item>` to that app's `appcast.xml` (newest first), commit the zip + xml, push.
