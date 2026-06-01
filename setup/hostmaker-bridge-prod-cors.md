# podmaker-bridge — production CORS allowlist

`podmaker-bridge` is the loopback daemon that exposes locally
installed credential CLIs (aws-vault, pass, sops, op, bw, gcloud,
az) to the SaaS UI. The browser fetches against
`http://127.0.0.1:7766` from whichever PodMaker domain the
operator is signed into, so the daemon ships with a CORS allowlist
that hard-rejects any other origin.

The default allowlist covers local dev + the staging URL only:

```
https://podmaker.test,http://podmaker.test,https://app.podmaker.sh,https://localhost:3000
```

If your production SaaS lives on a different domain — vanity URL,
EU mirror, customer-branded white label — extend the allowlist
before rolling the daemon out to that audience.

## Override via env

The CLI reads `PODMAKER_BRIDGE_ORIGINS` (comma-separated, no
spaces, no trailing slash):

```sh
PODMAKER_BRIDGE_ORIGINS="https://app.podmaker.sh,https://hm-eu.example.com,https://customer-portal.example.io" \
    podmaker-vault-bridge
```

## Override per-service

**systemd unit**

```ini
[Service]
Environment=PODMAKER_BRIDGE_ORIGINS=https://app.podmaker.sh,https://hm-eu.example.com
```

**macOS launchd plist**

```xml
<key>EnvironmentVariables</key>
<dict>
    <key>PODMAKER_BRIDGE_ORIGINS</key>
    <string>https://app.podmaker.sh,https://hm-eu.example.com</string>
</dict>
```

**brew services**

The Homebrew formula's service stanza passes its env through
`$HOMEBREW_PREFIX/etc/podmaker-vault-bridge.env`:

```sh
echo 'PODMAKER_BRIDGE_ORIGINS=https://app.podmaker.sh,https://hm-eu.example.com' \
    >> "$(brew --prefix)/etc/podmaker-vault-bridge.env"
brew services restart podmaker-vault-bridge
```

## Bearer token for shared workstations

When multiple operators share a workstation (rare, but happens on
build machines) flip the optional bearer:

```sh
PODMAKER_BRIDGE_TOKEN="$(openssl rand -hex 32)" podmaker-vault-bridge
```

The SaaS UI prompts the operator for the token on first connection
and stores it in localStorage. Without the bearer, any browser tab
on an allowed origin can hit the bridge.

## Verifying the allowlist

```sh
# Probe from the same machine, pretending to be the SaaS:
curl -s -H "Origin: https://app.podmaker.sh" \
    http://127.0.0.1:7766/v1/sources -i | head

# Should include:
#   Access-Control-Allow-Origin: https://app.podmaker.sh
#   Access-Control-Allow-Credentials: true

# A rejected origin gets the JSON body but no Allow-Origin header:
curl -s -H "Origin: https://attacker.example" \
    http://127.0.0.1:7766/v1/sources -i | head
```

Browsers refuse cross-origin XHR when the `Access-Control-Allow-Origin`
header is missing or doesn't match — that's the defense.

## Browser → localhost: mixed-content note

Modern browsers treat `http://127.0.0.1` as a Secure Context, so a
page served over HTTPS can still call the loopback daemon without
a mixed-content warning. This is a deliberate carve-out in the
specification and the design here depends on it. Do not bind the
daemon to anything other than `127.0.0.1` — the Public Suffix List
+ Secure Context carve-out only covers loopback.
