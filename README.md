# wrapper-lite

A lightweight single-port HTTP wrapper for Apple Music decryption.

It provides five endpoints on one HTTP port: M3U8, Key, Lyrics, License, and WebPlayback.

> **Recommended way to run**: use the `wrapper-lite-qemu` launcher — it boots a
> self-contained QEMU guest and forwards the HTTP API to the host, so you don't
> need a rooted device or a native chroot. See
> [Run with QEMU](#run-with-qemu-recommended).

## Build

```bash
wget https://dl.google.com/android/repository/android-ndk-r23b-linux.zip
unzip android-ndk-r23b-linux.zip
mkdir -p build && cd build
cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_POLICY_VERSION_MINIMUM=3.5
make -j$(nproc)
```

Debug build:

```bash
cmake .. -DCMAKE_BUILD_TYPE=Debug
```

Build outputs:

- `rootfs/system/bin/lite` — Android-layer binary
- `wrapper-lite` — privileged host launcher
- `wrapper-lite-rootless` — rootless host launcher (user namespaces)

## Run natively

```bash
# Login, cache tokens, then exit
./wrapper-lite-rootless --login user:pass --code-from-file --base-dir /data

# Start the HTTP service
./wrapper-lite-rootless --base-dir /data --host 0.0.0.0 --port 12340
```

2FA code:

- Interactive prompt when a TTY is available.
- With `--code-from-file`, the code is read from `data/2fa.txt`.

## Run with QEMU (recommended)

`wrapper-lite-qemu` is the recommended launcher. It boots the prebuilt guest
(the `qemu/` assets) in QEMU and forwards the guest's HTTP service to the host,
so no rooted device or native chroot is required. Prebuilt releases already
bundle QEMU plus its firmware in `qemu/bin/` for Linux, macOS, Windows and
Android.

```bash
# one-time: build the launcher (or download a prebuilt release)
c++ -std=c++11 -O2 -o wrapper-lite-qemu wrapper-lite-qemu.cpp

# login first (forwards --login to the guest lite), then serve
./wrapper-lite-qemu --login user:pass --code-from-file
./wrapper-lite-qemu
```

The launcher locates the QEMU binary in this order:

1. `--qemu-bin <path>` / `QEMU_BIN`
2. `qemu/bin/` (bundled)
3. `PATH`

The bundled QEMU is preferred so that the firmware (`-L qemu/bin`), the QEMU
modules (`QEMU_MODULE_DIR`) and the support libraries (`qemu/lib`) always come
from the same package. `-L`, `LD_LIBRARY_PATH` and `QEMU_MODULE_DIR` are only
adjusted for the bundled QEMU; a system QEMU keeps its own libraries, modules
and firmware.

A package never ships glibc (`libc.so.6`, `libm.so.6`): the ELF interpreter is
always the host's and cannot be replaced through `LD_LIBRARY_PATH`, so the
loader and libc must stay a matched pair from the host. Only non-glibc support
libraries are bundled, in `qemu/lib/` (`qemu/bin/` on Android, where the Termux
packages place them next to the binary).

See [QEMU launcher arguments](#qemu-launcher-arguments) for the full option
list.

## Docker

```bash
docker build -t wrapper-lite:local .
docker run --privileged -p 12340:12340 \
  -v ./rootfs/data:/app/rootfs/data \
  -e USERNAME=... -e PASSWORD=... \
  wrapper-lite:local
```

## HTTP API

All responses use:

```json
{"code":0,"msg":"SUCCESS","data":{...}}
```

| Endpoint | Method | Parameters |
|----------|--------|------------|
| `/m3u8` | GET | `adamId` |
| `/key` | GET | `adamId`, `uri` (required; the prefetch `skd://itunes.apple.com/P000000000/s1/e1` is rejected unless `adamId=0`) |
| `/lyrics` | GET | `adamId`, optional `language`, optional `syllable` (`1`=word-timed syllable-lyrics with translations/romanization, `0`=line-timed lyrics converted in-wrapper **and still carrying translations/romanization as whole-line text**), optional `script` (transliteration script, default `en-Latn`) |
| `/webplayback` | GET | `adamId`, optional `full` (`1`: also return Apple's complete webPlayback JSON), optional `language` (e.g. `en-US`), optional `library` (`1`: `adamId` is a library id). See [Webplayback and account tokens](#webplayback-and-account-tokens) |
| `/me` | GET | no parameters. This account's session tokens; **disabled unless `--expose-tokens`**. See [Webplayback and account tokens](#webplayback-and-account-tokens) |
| `/license` | POST | JSON: `adamId`, `challenge`, `uri`, optional `drm-type` (`wv`=Widevine (default), `pr`=PlayReady) |
| `/status` | GET | returns `regions` (list of storefront codes this wrapper can serve) and `capabilities` (`webplayback-full`, plus `tokens` when `--expose-tokens` is on) so clients can adapt |

## Webplayback and account tokens

Song tags (artist, album, genre, composer, copyright, track/disc numbers, ...)
come from Apple's *webPlayback* response: `songList[0].assets[0].metadata`.
Asking Apple for it normally needs the account's `media-user-token`, which is why
downloaders ask for a cookies file. wrapper-lite already holds that token, so it
can provide the data itself in two ways.

**1. `GET /webplayback?adamId=<id>&full=1`: no tokens leave the service.**
The default response is unchanged (`{"adamId", "m3u8"}`). With `full=1` it also
contains Apple's complete JSON under `webplayback`, exactly what a client with a
cookies file would have received:

```json
{"code":0,"msg":"SUCCESS","data":{
  "adamId":"1869136178",
  "m3u8":"https://aod.itunes.apple.com/.../prog.m3u8",
  "webplayback":{"songList":[{"songId":"1869136178","assets":[{"flavor":"28:ctrp256",
                  "metadata":{"itemId":"1869136178","artistName":"...","sort-artist":"...", "...":"..."}}]}]}}}
```

`language` (e.g. `en-US`) and `library=1` (for tracks of the user's own library)
are sent to Apple the way gamdl sends them. With `full=1` the call succeeds even
when the response has no HLS url, because the metadata alone is enough for tags.
Apple-side failures (`failureType`, a `dialog`, no `songList`) answer
`404 webplayback not available`.

**2. `GET /me` (opt-in): the data a cookies file would hold.**

```json
{"code":0,"msg":"SUCCESS","data":{
  "auth":{"state":"logged_in","dev_token":"...","dev_token_source":"web",
          "music_user_token":"..."},
  "storefront":"us","regions":["us"]}}
```

`dev_token` is the Bearer token for Apple Music's web API (the one the browser
uses, scraped from music.apple.com), `music_user_token` is the value of the
`media-user-token` cookie. `dev_token_source` is `android` when the web token
could not be fetched and the Android token is used as a fallback (accepted for
webPlayback and lyrics, rejected for licenses). Together they let a client call
Apple Music's API as this account without a cookies file.

> **Security.** Anyone who can read `/me` can act as this Apple Music account
> (library, playlists, ...). That is why it answers `403` unless the service was
> started with `--expose-tokens`, and why the flag prints a warning at start-up.
> The endpoint cannot restrict itself to local clients: behind the QEMU launcher
> every connection reaches the guest from the NAT gateway, not from loopback. So
> the port itself must stay private. Check your setup before enabling the flag:
>
> - **Native:** do not combine it with `--host 0.0.0.0` (used in the example
>   above); the default `127.0.0.1` is fine.
> - **QEMU:** the launcher's `--host` / `LITE_QEMU_HOST` defaults to `127.0.0.1`;
>   keep it.
> - **Docker:** the `docker run -p 12340:12340` example and the shipped
>   `compose.yaml` publish the port on **all** host interfaces. Change it to
>   `127.0.0.1:12340:12340` first.
>
> Prefer option 1 whenever the client only needs tags. The response is sent with
> `Cache-Control: no-store` and tokens are never written to the service log.

`--expose-tokens` is forwarded like any other `lite` argument by the QEMU and
native launchers (`./wrapper-lite-qemu --expose-tokens`); with Docker pass it as
the container command (`command: ["--expose-tokens"]` in compose).

**Lyrics for downloaders.** gamdl-style tools read lyrics from the song's
metadata, which Apple only fills in for an authenticated request, so a client
without cookies gets none. `GET /lyrics?adamId=<id>&syllable=0` supplies them
with this service's own session. Use `syllable=0` for such tools: it returns
line-timed TTML in which every `<p begin end>` holds the whole line as direct
text (word `<span>`s stripped, background vocals merged), which is what their
parsers read; translations and romanization stay in `<text>` elements outside the
`<p>` lines, so they never leak into the lyrics. The default `syllable=1`
(word-timed) is for karaoke-style clients: a parser that reads `<p>` text sees
nothing in it. A track without lyrics answers `404 lyrics not found`.

## lite arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `--login user:pass` | — | login, cache tokens, then exit |
| `--code-from-file` | off | read 2FA code from file |
| `--host` | `127.0.0.1` | listen address |
| `--port` | `12340` | listen port |
| `--base-dir` | `data` | data directory |
| `--device-info` | auto | override the entire device-info string (see [Device info](#device-info)) |
| `--proxy` | — | proxy URL |
| `--debug` | off | debug logging; disables SSL verify in Release |
| `--log-level` | `info` | `debug`, `info`, `warn`, `error` |
| `--log-file` | — | log file path |
| `--token-refresh-interval` | `1800` | background refresh interval in seconds |
| `--expose-tokens` | off | enable `GET /me`, which serves this account's tokens (see the security note below) |

## Device info

`lite` identifies itself to Apple's servers with a device-info string of the
form:

```
ClientIdentifier/VersionIdentifier/PlatformIdentifier/ProductVersion/DeviceModel/BuildVersion/LocaleIdentifier/LanguageIdentifier/AndroidID
```

Example: `Music/5.0.2/Android/10/Pixel 8/7663314/en-US/en-US/<android-id>`.

The trailing `AndroidID` is derived automatically:

- During `--login`, it is the FNV-1a 64-bit hash of the account username and is
  persisted to `<base-dir>/ANDROID_ID`, so each account keeps a stable device
  identity across runs.
- In service mode (no username), the persisted `ANDROID_ID` is reused; if none
  exists, a built-in default is used.

Pass `--device-info <string>` to override the entire string explicitly.

## QEMU launcher arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `--host <addr>` | `127.0.0.1` | host address the forwarded port binds to (`0.0.0.0` to expose) |
| `--host-port <port>` | `12340` | host port |
| `--guest-port <port>` | `12340` | guest port |
| `--guest-host <addr>` | `0.0.0.0` | address the guest lite listens on (must be reachable by QEMU forwarding) |
| `--memory <MB>` | `512` | guest memory in MB |
| `--smp <N>` | `2` | guest CPU count |
| `--accel <accel>` | auto | force acceleration (`kvm`, `hvf`, `whpx`, `tcg`) |
| `--qemu-bin <path>` | auto | QEMU binary path |

Environment variables (fallbacks; command-line flags take precedence):

| Variable | Default | Description |
|----------|---------|-------------|
| `LITE_QEMU_HOST` | `127.0.0.1` | host address the forwarded port binds to |
| `HOST_PORT` | `12340` | host port |
| `GUEST_PORT` | `12340` | guest port |
| `LITE_GUEST_HOST` | `0.0.0.0` | address the guest lite listens on |
| `MEMORY` | `512` | guest memory in MB |
| `SMP` | `2` | guest CPU count |
| `LITE_QEMU_ACCEL` | auto | force acceleration |
| `QEMU_BIN` | auto | QEMU binary path |

## Build notes

- Release: SSL verification is enabled by default. Pass `--debug` to disable it temporarily.
- Debug: SSL verification is disabled by default.

## License

This project is licensed under the [MIT License](LICENSE).
