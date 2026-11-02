# oviron/Throne

Fork of [throneproj/Throne](https://github.com/throneproj/Throne) whose core
change is the TLS fingerprint library of its Xray core: `core/go.mod` replaces
`github.com/refraction-networking/utls` with the `oviron-refraction` branch of
[oviron/utls](https://github.com/oviron/utls), which adds a Firefox 158 parrot
(refraction-networking/utls#427, #433).

What that means for the fingerprint of Xray-style profiles:

| value | upstream 1.3.2 | this fork |
|---|---|---|
| `firefox` | Firefox 148 | Firefox 158 |
| `chrome` | Chrome 133 | Chrome 155 |
| `hellofirefox_120` | Firefox 120 | Firefox 120, same ClientHello |

Firefox 158 is the ClientHello of current Firefox stable in normal browsing mode
and carries X25519MLKEM768, so Xray REALITY servers from 26.9 on accept it.

sing-box keeps the stock `github.com/metacubex/utls`, so sing-box-style profiles
send upstream fingerprints (`firefox` is Firefox 120). Replacing it with the
oviron/utls branch built on MetaCubeX v1.9.0 broke Hysteria2 (QUIC fails with
`missing initial_source_connection_id`) and crashed the core on exit, and the
sing-box REALITY client removes X25519MLKEM768 from any fingerprint anyway.

Two more changes: the updater checks this repository's releases, and
`.github/workflows/oviron-release.yml` builds macOS arm64 only. The upstream
`build.yml` (manual dispatch) is not registered in this repository and does not run.

## Versions

Branch `oviron` sits on an upstream release tag. A fork release is tagged
`<upstream>-oviron.<n>` (for example `1.3.2-oviron.2`); pushing the tag builds
`Throne-<tag>-macos-arm64.zip` with `SHA256SUMS` and publishes the release. The
updater compares only the upstream part of the version, so it offers a new
upstream base but not the next `-oviron.<n>` of the same base.

`1.3.2-oviron.1` also replaced the sing-box utls and is affected by the
Hysteria2 failure above; use `1.3.2-oviron.2` or later.

## New upstream release

```sh
git fetch upstream --tags
git rebase --onto NEW OLD oviron
# tidy sees the gRPC imports only with core/gen generated
(cd core/gen && protoc -I . --go_out=. --go-grpc_out=. libcore.proto)
(cd core && go mod tidy)
git tag NEW-oviron.1 && git push origin oviron NEW-oviron.1
```

When upstream moves to a uTLS release with Firefox 158 of its own, the fork is
no longer needed.
