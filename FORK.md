# oviron/Throne

Fork of [throneproj/Throne](https://github.com/throneproj/Throne) whose core
change is the TLS fingerprint library of its Xray core: `core/go.mod` replaces
`github.com/refraction-networking/utls` with the `oviron-refraction` branch of
[oviron/utls](https://github.com/oviron/utls), which adds a Firefox 158 parrot
(refraction-networking/utls#427, #433).

What that means for the fingerprint of Xray-style profiles:

| value | upstream 1.4.2 | this fork |
|---|---|---|
| `firefox` | Firefox 148 | Firefox 158 |
| `chrome` | Chrome 133 | Chrome 155 |
| `hellofirefox_120` | Firefox 120 | Firefox 120, same ClientHello |

Firefox 158 is the ClientHello of current Firefox stable in normal browsing mode
and carries X25519MLKEM768, so Xray REALITY servers from 26.9 on accept it.

sing-box keeps the stock `github.com/metacubex/utls`, so sing-box-style profiles
send upstream fingerprints (`firefox` is Firefox 120). Replacing it with the
oviron/utls branch built on MetaCubeX v1.9.0 broke Hysteria2 (QUIC fails with
`missing initial_source_connection_id`), and the sing-box REALITY client removes
X25519MLKEM768 from any fingerprint anyway.

`core/go.mod` also requires `github.com/klauspost/compress` v1.20.4, where
upstream 1.4.2 has v1.20.1: before v1.20.2 a crafted zstd stream panics the
decoder on its own goroutine and takes the core down
([GHSA-fpmh-frq8-7rj8](https://github.com/klauspost/compress/security/advisories/GHSA-fpmh-frq8-7rj8)),
and both uTLS copies decompress zstd-compressed certificates.

Two more changes: the updater checks this repository's releases and reads
version `<upstream>-oviron.<n>` as `<upstream>`, and
`.github/workflows/oviron-release.yml` builds only macOS arm64 and Windows x64,
with the steps of upstream's `build.yml` for them. Upstream's `build.yml` is
disabled in this repository.

## Versions

Branch `oviron` sits on an upstream release tag. A fork release is tagged
`<upstream>-oviron.<n>` (for example `1.4.2-oviron.1`); pushing the tag builds
`Throne-<tag>-macos-arm64.zip` and `Throne-<tag>-windows64.zip` and publishes the
release. The updater compares only the upstream part of the version, so it offers a new
upstream base but not the next `-oviron.<n>` of the same base.

When GitHub cannot acquire a macOS arm64 runner (the job is cancelled with no steps
run), build the existing tag on an Intel runner, which builds universal and thins the
executable to arm64:
`gh workflow run oviron-release.yml --ref oviron -f tag=<tag> -f macos_runner=macos-15-intel`.

## New upstream release

`OLD` is the upstream tag the branch sits on, `NEW` the new one. Upstream bumps
its dependencies in every release, so the rebase stops on `core/go.mod` and
`core/go.sum`: take upstream's files and apply the replace again, with the
latest `v*-refraction.*` tag of oviron/utls, and keep `klauspost/compress` at
v1.20.2 or later. Carry upstream's changes to the
macOS arm64 and Windows x64 build steps over to `oviron-release.yml`:
`git diff OLD NEW -- .github/workflows/build.yml script/` shows them.

```sh
export GOTOOLCHAIN=go1.27.3   # GO_VERSION of the release workflow
git fetch upstream --tags
git rebase --onto NEW OLD oviron
git checkout NEW -- core/go.mod core/go.sum
# tidy sees the gRPC imports only with core/gen generated
(cd core/gen && protoc -I . --go_out=. --go-grpc_out=. libcore.proto)
(cd core && go mod edit -go=1.27 \
  -replace github.com/refraction-networking/utls=github.com/oviron/utls@v1.8.3-refraction.2 &&
  go mod tidy)
git add core/go.mod core/go.sum && git rebase --continue
git push --force-with-lease origin oviron
git tag NEW-oviron.1 && git push origin NEW-oviron.1
```

When upstream moves to a uTLS release with Firefox 158 of its own, the fork is
no longer needed.
