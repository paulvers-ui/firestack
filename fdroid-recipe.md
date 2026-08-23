# F-Droid Build Recipe — `paulvers-ui/firestack`

> **Type:** Go → Android AAR (gomobile bind)  
> **F-Droid note:** firestack is a library (`.aar`), not an APK.  
> It is not distributed via F-Droid directly — it is a **build dependency**
> of `rethink-app` (resolved via JitPack). This recipe documents how to
> reproduce the `.aar` from source so the rethink-app build is fully
> verifiable / self-hostable.

---

## Build environment requirements

| Tool | Version | Notes |
|------|---------|-------|
| Go | 1.22+ | Must match `go.mod` (`go 1.26`) |
| gomobile | latest | `go install golang.org/x/mobile/cmd/gomobile@latest` |
| Android NDK | r26b | Set `$ANDROID_NDK_HOME` |
| Android SDK | API 23+ | `$ANDROID_HOME` |
| Java | 17 | For aar packaging step |

---

## Metadata file: `metadata/github.paulvers-ui.firestack.yml`

```yaml
# firestack is a library — no standalone APK.
# This metadata is for documentation / self-hosted repo use only.

Categories:
  - Connectivity

License: Apache-2.0

WebSite: https://github.com/paulvers-ui/firestack
SourceCode: https://github.com/paulvers-ui/firestack
IssueTracker: https://github.com/paulvers-ui/firestack/issues

AutoName: firestack

Summary: Go-based Android VPN/proxy engine (gomobile AAR)
Description: |-
  firestack is the Go network engine powering Rethink Fork / Brave DNS Fork.
  It implements DNS-over-HTTPS, DNS-over-TLS, WireGuard tunnelling,
  SOCKS5 proxying, and a per-app firewall via Android VpnService.
  Built with gomobile bind into an AAR consumed by the Kotlin app layer.

RepoType: git
Repo: https://github.com/paulvers-ui/firestack
Branch: n2

Builds:
  - versionName: 'n2'
    versionCode: 1
    commit: HEAD    # pin to a specific commit SHA for reproducibility
    subdir: .
    sudo:
      - apt-get install -y openjdk-17-jdk golang-go unzip curl
      - curl -sL https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip -o sdk.zip
      - unzip -q sdk.zip -d $HOME/android-sdk
      - yes | $HOME/android-sdk/cmdline-tools/bin/sdkmanager --sdk_root=$HOME/android-sdk "ndk;26.3.11579264" "platforms;android-35"
    prebuild:
      - export ANDROID_HOME=$HOME/android-sdk
      - export ANDROID_NDK_HOME=$ANDROID_HOME/ndk/26.3.11579264
      - export GOPATH=$HOME/go
      - export PATH=$PATH:$GOPATH/bin:$ANDROID_HOME/cmdline-tools/bin
      - go install golang.org/x/mobile/cmd/gomobile@latest
      - go install golang.org/x/mobile/cmd/gobind@latest
      - gomobile init
    build:
      - make aar   # see Makefile target below
    output: build/firestack.aar

AutoUpdateMode: None
UpdateCheckMode: None
CurrentVersion: 'n2'
CurrentVersionCode: 1
```

---

## Manual build steps

```bash
git clone https://github.com/paulvers-ui/firestack
cd firestack
git checkout n2

# Install gomobile (once)
go install golang.org/x/mobile/cmd/gomobile@latest
go install golang.org/x/mobile/cmd/gobind@latest
gomobile init

# Set up Android NDK
export ANDROID_NDK_HOME=/path/to/ndk/26.3.11579264

# Full AAR build (from Makefile)
make aar

# What `make aar` expands to:
mkdir -p build
env PATH=$HOME/go/bin:$PATH \
  gomobile bind \
    -trimpath -v -x -a \
    -javapkg com.celzero.firestack \
    -androidapi 23 \
    -target=android \
    -tags='android' \
    -overlay=build/overlay.json \
    -ldflags='-checklinkname=0 -w -s -buildid= \
      -X github.com/celzero/firestack/intra/core.Date=$(date -u +%Y%m%d%H%M%S) \
      -X github.com/celzero/firestack/intra/core.Commit=$(git rev-parse --short HEAD)' \
    -gcflags='all=-trimpath=$(pwd)' \
    -o build/firestack.aar \
    github.com/celzero/firestack/intra
```

## Using the built AAR in rethink-app

Copy the output `build/firestack.aar` into `rethink-app/app/libs/firestack.aar`  
then in `app/build.gradle` set:

```groovy
// Switch from JitPack to local file
implementation fileTree(dir: 'libs', include: ['*.aar'])
// Comment out the JitPack dependency:
// implementation "com.github.celzero:firestack:${firestackCommit}@aar"
```

---

## Branches

| Branch | Purpose |
|--------|---------|
| `n2` | Current main development branch |
| `fix-attestation-permissions` | Patch for nilaway/err113 fixes |
| `restore-from-backup` | Emergency backup restore point |
