# Install Guide — by account type and permissions

This guide exists because of a measured failure, not a hypothetical one: an adopter on Windows ran
`npm install -g` three times — once from an elevated PowerShell — and hit the same `EPERM` every
time, because their npm global prefix pointed inside `C:\Program Files\nodejs`. The local install
worked on the first try. Find your row, run its one command.

## Which situation are you in?

| Situation | Privileges needed | Command |
|---|---|---|
| **Using it in one project** (most people) | none | `npm install --save-dev @holmes-lab/holmes-kit` then `npx holmes-kit init` |
| Company-managed PC / restricted account | none | same as above — no system directory is touched |
| CI / container | none | same as above, plus `--prefer-online` right after a release |
| Want the CLI across many projects (`-g`) | depends on your prefix | **check first**: `npm config get prefix` ↓ |

The local install is the path verified end to end, on macOS and Windows, against the public
registry. The wiring `init` writes uses absolute paths, so nothing needs to be on `PATH`.

## Before `npm install -g`: check your prefix

```
npm config get prefix
```

| Result looks like | Verdict |
|---|---|
| `%APPDATA%\npm`, `~/.npm-global`, `/opt/homebrew`, an nvm/fnm/volta directory | user-writable — `npm install -g @holmes-lab/holmes-kit` works as-is |
| `C:\Program Files\nodejs`, `/usr/local` | protected — `-g` dies with `EPERM` **before any package file arrives**. Move the prefix (below). **Do not elevate.** |

`npx holmes-kit doctor` (after a local install) runs this exact check for you — the
`global prefix` line names the directory and the remedy.

### Moving the prefix to user space — one-time setup

Windows (PowerShell):

```powershell
npm config set prefix "$env:APPDATA\npm"
[Environment]::SetEnvironmentVariable('Path', "$([Environment]::GetEnvironmentVariable('Path','User'));$env:APPDATA\npm", 'User')
# open a NEW terminal, then:
npm install -g @holmes-lab/holmes-kit
```

macOS / Linux:

```bash
npm config set prefix "$HOME/.npm-global"
export PATH="$HOME/.npm-global/bin:$PATH"   # add to your shell profile too
npm install -g @holmes-lab/holmes-kit
```

A Node version manager (`nvm`, `fnm`, `volta`) achieves the same by keeping the whole toolchain
under your home directory.

### Why elevation is the wrong fix

npm's own error text ends with *"try running the command again as root/Administrator."* Do not
follow it here, for two reasons:

1. **It may not even work.** The reported failure recurred from an elevated PowerShell — antivirus
   and Windows Controlled Folder Access block protected-folder writes regardless of elevation.
2. **When it works, it is worse.** `better-sqlite3` declares
   `install: prebuild-install || node-gyp rebuild` — under an elevated `-g`, that downloads and
   executes, or invokes a compiler, **with system privileges**. Keeping installs in user space is
   what contains a compromised dependency.

## Troubleshooting, by the error you actually see

### `EPERM … mkdir C:\Program Files\nodejs\node_modules\@holmes-lab`

```
npm error code EPERM
npm error syscall mkdir
npm error path C:\Program Files\nodejs\node_modules\@holmes-lab
```

Your global prefix is a protected directory. The failure happens while npm creates the scope
folder — **before a single package file is transferred** — so no package version can fix it, and
neither can this one. Either drop `-g` (the local install needs none of this) or move the prefix
(one-time setup above).

### `notarget No matching version found`

```
npm error code ETARGET
npm error notarget No matching version found for @holmes-lab/holmes-kit@<version>
```

Your npm metadata cache predates the release — measured minutes after publishing 0.1.11, the
registry already listed the version while a default-cache install still refused it. Add
`--prefer-online`, or retry in a few minutes.

### `better-sqlite3` fails to build

The one dependency that may need a toolchain. The 8 tree-sitter grammars ship prebuilt binaries
(`darwin-arm64`, `darwin-x64`, `linux-x64`, `win32-x64`) and compile nothing; `better-sqlite3`
downloads a prebuild at install time and **falls back to compiling** when none matches your
platform and Node ABI. If it compiles, you need:

| Platform | Toolchain |
|---|---|
| Windows | Visual Studio Build Tools (C++ workload) |
| macOS | Xcode Command Line Tools (`xcode-select --install`) |
| Alpine | `apk add --no-cache python3 make g++` |

### `spawn sh ENOENT` during a git-URL install

`npm i -g git+ssh://…` is not a supported path: npm 11 clones the repository into its cache and
runs `prepare` there without installing dependencies, so the build tooling is missing. Install
from the registry or from a packed tarball.

## Verify — the last step of every path

```
npx holmes-kit doctor        # local install
holmes-kit doctor            # global install
```

Expect `10 pass, 1 warn, 0 fail` on a healthy install. The lines that matter most:

- `global prefix` — whether `-g` would work on this machine, and the remedy if not
- `tree-sitter grammars` / `better-sqlite3` — whether the native modules actually load

## What we deliberately do NOT do

| Idea | Why not |
|---|---|
| A `postinstall` script that prints guidance | Triggers npm 11's `allow-scripts` warning and forfeits this package's current property of running no install scripts at all |
| Recommending `npx @holmes-lab/holmes-kit init` with no install | `init` writes wiring with absolute paths; under bare `npx` those point into the npx cache and break when it is pruned |
| Fixing your npm prefix from inside the package | A package rewriting your npm configuration is exactly the supply-chain behaviour this guide warns about |
