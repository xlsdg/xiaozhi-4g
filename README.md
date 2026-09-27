# 4G dual-network firmware (lckfb SZPI ESP32-S3 + ML307)

Firmware for the LCKFB SZPI ESP32-S3 board (立创·实战派) with an ML307 4G module attached.
It runs as a dual-network device: WiFi by default, 4G after a button switch. Everything except
networking is upstream [78/xiaozhi-esp32](https://github.com/78/xiaozhi-esp32) code.

- Flash from the browser: <https://xlsdg.org/xiaozhi-4g/>
- Downloads: [Releases](https://github.com/xlsdg/xiaozhi-4g/releases)
- Updates: manual over USB only, see [Updates](#updates)
- Two branches, see [Repository layout](#repository-layout):
  `main` (this branch: CI, flasher page, docs) and `dual` (upstream + the port commit)

## Repository layout

| Branch | Content |
|---|---|
| `main` (default) | `.github/workflows/firmware.yml`, `pages/`, this README. No upstream code. |
| `dual` | The newest upstream release tag plus the port commit on top (2 files). |

Your whole firmware change is always `git show origin/dual` (or `git log <tag>..origin/dual`
if it ever grows beyond one commit). Nothing on `main` ever conflicts with upstream.

## Hardware

| Board | ML307 module (bottom-left 5P header) |
|---|---|
| 5V / 3V3 / GND / IO10 / IO11 | 5V / EN / GND / TXD / RXD |

EN is tied to 3V3, so the module is always powered. There is no power control on this header.

## Flashing

```bash
esptool.py -p /dev/ttyUSB0 -b 460800 erase_flash
esptool.py -p /dev/ttyUSB0 -b 460800 write_flash 0x0 merged-binary.bin
```

`erase_flash` wipes NVS, so WiFi credentials, the activation token and the selected network
type are reset. After flashing, the device boots into WiFi configuration mode (AP named
`Xiaozhi-...`); configure WiFi there, and use 4G only after switching networks.

`merged-binary.bin` is a complete image (bootloader + partition table + app + assets) flashed
at offset `0x0`.

Or from a browser, no toolchain needed (Chrome or Edge on desktop; Safari and Firefox cannot
flash over USB): <https://xlsdg.org/xiaozhi-4g/>. The page is served by GitHub Pages together with
the latest build's `merged-binary.bin` (release downloads redirect without CORS headers, so the
browser cannot fetch them directly) and writes it at `0x0`, with an **Erase device** option. Its
**Logs & Console** entry shows the serial log. It is the quickest fix for a board that no longer
boots. Older builds are only on [Releases](https://github.com/xlsdg/xiaozhi-4g/releases); flash
those with esptool.

## Button gestures (BOOT)

| Gesture | Action |
|---|---|
| 1 click | toggle chat / enter WiFi config mode during startup |
| 2 clicks | toggle device-side AEC |
| hold | press-to-talk (talk while held) |
| 4 clicks | switch between WiFi and 4G, then reboot |

The network type is persisted in NVS. Double click is taken by AEC and long press by
press-to-talk, which is why the network switch uses 4 clicks.

## Updates

There is no automatic update. Reflash over USB (browser page or esptool) to upgrade.

The device still uses the official server (`https://api.tenclass.net/xiaozhi/ota/`, the Kconfig
default) at boot for activation and server configuration. The firmware reports the board type
`lichuang-dev-dual`. Upstream's own type (`lichuang-dev`) is deliberately not used: the official
OTA server keys firmware on that type and would replace this build with a WiFi-only one.

## How CI works

`.github/workflows/firmware.yml` lives on `main`, runs daily (scheduled workflows only run from
the default branch), on manual dispatch, and when the workflow file itself changes on `main`:

1. Resolves the newest upstream release tag (`vX.Y.Z`, optionally `_N`; pre-release tags are
   ignored) and checks out `dual`.
2. If `dual` is not based on that tag yet, rebases it onto the tag. A conflict fails the run with
   the conflicting files listed.
3. Skips everything when a release for the resulting commit already exists (unless `force`).
4. Builds in `espressif/idf:v6.1` with upstream's own tool:
   `python scripts/build.py lckfb/szpi-esp32s3 --name lichuang-dev-dual --language zh-CN --zip`
5. Force-pushes the rebased `dual` (`--force-with-lease`), so the branch tip always equals the last
   firmware that built successfully.
6. Publishes a release tagged `<upstream tag>-4g.<short sha>` on that commit with
   `merged-binary.bin` and the zip, then prunes everything except the 3 newest `-4g.` releases.
   Release tags keep the commits of earlier rebases alive.
7. Deploys `pages/index.html`, `merged-binary.bin` and a generated esp-web-tools `manifest.json`
   to GitHub Pages (Settings > Pages > Source: GitHub Actions).

Changing only `pages/index.html` does not create a new build, so redeploy it with
`gh workflow run firmware.yml -f force=true`.

A push to `dual` does not trigger a build (the workflow file is not on that branch). After changing
the port by hand, run `gh workflow run firmware.yml`.

Upstream's own workflows exist only on `dual`, where they never run on schedule; `Build Boards` is
additionally disabled. If a future upstream release adds a push-triggered workflow, disable it with
`gh workflow disable <name>`.

When a rebase conflicts:

```bash
git fetch origin dual && git switch dual && git reset --hard origin/dual
git fetch --no-tags https://github.com/78/xiaozhi-esp32.git "refs/tags/<tag>:refs/tags/<tag>"
git rebase <tag>            # resolve, check the 5 invariants below, git rebase --continue
git push --force-with-lease origin dual
gh workflow run firmware.yml
```

Notes for editing this workflow, all learned the hard way:

- The IDF container's default shell is `dash`, so every `run:` using `set -o pipefail` or `source`
  needs `shell: bash`.
- Under `pipefail`, `... | head -1` fails the step with exit 141 (SIGPIPE). Use `awk 'NR==1{...}'`.
- `scripts/build.py` invokes `idf.py` with `sys.executable`, so the IDF virtualenv must be active:
  `source "$IDF_PATH/export.sh"` before running it, otherwise `python` is the system interpreter and
  `idf.py` fails on missing modules.
- `force` on manual dispatch rebuilds even when a release for the current `dual` commit exists.

## The port: 5 invariants

Upstream restructures boards regularly (the board directory moved from `main/boards/lichuang-dev`
to `main/boards/lckfb/szpi-esp32s3`, and the board file is touched every few weeks). Rather than
maintaining a patch text, the port is defined by what must be true after every rebase. All of it
lives in one file: `main/boards/lckfb/szpi-esp32s3/lichuang_dev_board.cc` (the board identity
lives in the `config.json` next to it).

1. The board class derives from `DualNetworkBoard`, not `WifiBoard`. `Board::GetInstance()` picks
   the concrete board at compile time, so this line is the only place where "the network can be
   WiFi or 4G at runtime" can be expressed.
2. The constructor passes the ML307 UART pins: `DualNetworkBoard(GPIO_NUM_11, GPIO_NUM_10, ...)`.
   `0` selects WiFi as the initial network type.
3. Anything that only exists on `WifiBoard` is reachable. Here that is `EnterWifiConfigMode()`,
   provided as a private wrapper that forwards to `GetCurrentBoard()` and is a no-op on 4G. New
   upstream call sites of `WifiBoard`-only methods will not compile until they are routed the
   same way.
4. Exactly one gesture calls `SwitchNetworkType()` (4 clicks).
5. Hardware virtuals (`GetDisplay`, `GetAudioCodec`, `GetCamera`, `GetBacklight`) stay on this
   class. The inner network board is a separate `Board` whose display is a no-op; the application
   only ever uses the outer singleton. Copying a whole board file (as the old patch did) freezes
   camera/audio/touch code and loses every upstream fix, so do not.

After each rebase, this one-liner (run on `dual`) is the contract check:

```bash
grep -n 'class LichuangDevBoard\|DualNetworkBoard(\|EnterWifiConfigMode\|WifiManager::\|SwitchNetworkType' \
  main/boards/lckfb/szpi-esp32s3/lichuang_dev_board.cc
```

Expected: one constructor call, `EnterWifiConfigMode` defined only by the wrapper (upstream call
sites resolve to it), one
`SwitchNetworkType`, zero `WifiManager::`, and the class not deriving from `WifiBoard`. Any extra
hit means upstream added a WiFi-specific call that has to be routed through the wrapper.

## Inspecting the port

```bash
git fetch origin dual
git show --stat origin/dual          # the port commit and its 2 files
git show origin/dual                 # the full diff against upstream
```

On GitHub: the latest commit on the `dual` branch. Release tags (`v<upstream>-4g.<sha>`) point at
the port commit each firmware was built from.

## Living reference implementations

Both are upstream boards, updated by upstream when the dual-network API changes:

- `main/boards/yunliao-s3/yunliao_s3.cc` — dual network, AEC, multiple-click gesture, same
  constructor form.
- `main/boards/zhengchen/cam-ml307/zhengchen_cam_board_ml307.cc` — dual network, camera, AEC.

## Local build

On a checkout of `dual`, with ESP-IDF 6.1 (or the container):

```bash
docker run --rm -v "$PWD":/project -w /project espressif/idf:v6.1 \
  bash -c 'python scripts/build.py lckfb/szpi-esp32s3 --name lichuang-dev-dual --language zh-CN --zip'
```

Output: `build/merged-binary.bin`, and `releases/v<version>_lckfb-lichuang-dev-dual.zip`.
