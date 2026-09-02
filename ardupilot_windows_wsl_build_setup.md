# Building Skybrush ArduPilot on Windows (via WSL)

## Can this be built on Windows?

**No.** ArduPilot flight-controller firmware has no supported native-Windows toolchain.
The build runs on **Linux**, which on Windows means **WSL2** (Windows Subsystem for
Linux). Everything below happens inside a WSL Ubuntu install. Editing can still be done
from Windows (VS Code), but `./waf` must run in WSL.

> There is a Cygwin-based path that runs `waf` natively on Windows, but ArduPilot
> treats it as a developer-only option — it is slower, breaks on long file paths, and
> is not actively maintained. **WSL is the recommended route for everything**, board
> firmware and SITL alike.

**Upstream references:**

- [ArduPilot — Building the Code](https://ardupilot.org/dev/docs/building-the-code.html)
  — Linux/macOS build with `waf` directly; Windows users are directed to WSL/WSL2 and
  then the Linux instructions.
- [ArduPilot — Setting up the Build Environment on Windows 11 using WSL](https://ardupilot.org/dev/docs/building-setup-windows11.html)
  — the official Windows-side setup (enable WSL2, install Ubuntu, toolchain, VS Code,
  USB passthrough). The guide below is our condensed version of this, pinned to the
  Skybrush fork and the `MatekH743-bdshot` board.
- [ArduPilot — Linux/Ubuntu build environment](https://ardupilot.org/dev/docs/building-setup-linux.html)
  — what runs *inside* WSL once Ubuntu is installed.

---

## Overview of what we set up

| Piece | Value |
|---|---|
| OS layer | WSL2 + Ubuntu 24.04 |
| Repo | `git@github.com:skybrush-io/ardupilot.git` |
| Branch | `CMCopter-4.6` |
| Checkout location | `~/ardupilot` **inside WSL** (not `/mnt/c/...`) |
| Target board | `MatekH743-bdshot` (STM32H743) |
| Python | 3.12 in a venv at `~/venv-ardupilot` |
| ARM toolchain | `gcc-arm-none-eabi-10-2020-q4-major` in `~/toolchains/` (**not** the apt package) |

---

## Step 1 — Install WSL2 + Ubuntu (Windows)

Open **PowerShell as Administrator**:

```powershell
wsl --install -d Ubuntu
```

Reboot if asked. On first launch Ubuntu asks for a UNIX username and password
(this password is what `sudo` will prompt for). Confirm it worked:

```powershell
wsl --list --verbose      # STATE should be Running/Stopped, VERSION 2
```

From here on, "run in WSL" means: open a terminal and type `wsl`, or use the
**Ubuntu** app, or the integrated terminal of a VS Code window connected to WSL.

---

## Step 2 — Base packages (WSL)

```bash
sudo apt update
sudo apt install -y git build-essential ccache python3 python3-venv python3-pip \
                    rsync wget
```

Do **not** install `gcc-arm-none-eabi` from apt — on Ubuntu 24.04 that package is
missing `libstdc++_nano.a` and the firmware link step fails with
`cannot find -lstdc++_nano`. We install ArduPilot's own toolchain in Step 5.

---

## Step 3 — Clone the repo (WSL)

Clone **into the Linux filesystem** (`~`), never into `/mnt/c/...`:
`/mnt/c` builds are slow and Windows line-endings (CRLF) break `waf`.

```bash
cd ~
git clone --branch CMCopter-4.6 git@github.com:skybrush-io/ardupilot.git
cd ardupilot
git submodule update --init --recursive        # ~2 GB, several minutes
```

If you don't have SSH set up yet, clone over HTTPS instead and switch to SSH later
(see Step 7):

```bash
git clone --branch CMCopter-4.6 https://github.com/skybrush-io/ardupilot.git
```

---

## Step 4 — Python environment (WSL)

ArduPilot's build scripts need specific Python packages. Keep them in a venv:

```bash
python3 -m venv ~/venv-ardupilot
source ~/venv-ardupilot/bin/activate
pip install --upgrade pip wheel
pip install empy==3.3.4 pymavlink MAVProxy pexpect dronecan future intelhex lxml "numpy<2"
```

`empy` **must** be `3.3.4` — newer versions break `waf`.

---

## Step 5 — ARM toolchain (WSL)

Download ArduPilot's known-good GCC 10.2.1 for ARM (no sudo needed, lives in your home):

```bash
mkdir -p ~/toolchains && cd ~/toolchains
wget https://firmware.ardupilot.org/Tools/STM32-tools/gcc-arm-none-eabi-10-2020-q4-major-x86_64-linux.tar.bz2
tar xjf gcc-arm-none-eabi-10-2020-q4-major-x86_64-linux.tar.bz2
rm gcc-arm-none-eabi-10-2020-q4-major-x86_64-linux.tar.bz2
```

---

## Step 6 — Make it automatic (WSL)

Add these two lines to the **top** of `~/.bashrc` so every new terminal has the
toolchain and the venv ready:

```bash
export PATH=$HOME/toolchains/gcc-arm-none-eabi-10-2020-q4-major/bin:$PATH
source ~/venv-ardupilot/bin/activate
```

Open a fresh terminal and check:

```bash
which arm-none-eabi-gcc      # -> ~/toolchains/.../bin/arm-none-eabi-gcc
arm-none-eabi-gcc --version  # -> 10.2.1
python -c "import em; print(em.__version__)"   # -> 3.3.4
```

---

## Step 7 — git identity + push access (WSL, optional until you push)

First, the commit identity (needed either way):

```bash
git config --global user.name  "YOUR_GITHUB_USERNAME"
git config --global user.email "your.work@email"
```

Then pick **one** auth method.

---

### Option A — HTTPS + token via GitHub CLI (simplest, recommended for WSL / teams)

```bash
sudo apt install -y gh          # if not already installed
gh auth login
#   -> GitHub.com  ->  HTTPS  ->  authenticate: "Login with a web browser"
#   -> copy the one-time code, press Enter, paste it in the browser, approve
gh auth setup-git               # makes git use the token automatically
```

Keep the repo remote on HTTPS (`https://github.com/<you>/ardupilot.git`). No key
files, nothing to add on the GitHub site. Each teammate just runs `gh auth login`
on their own machine.

---

### Option B — SSH key (more setup, then set-and-forget)

```bash
# SSH key for this WSL environment (separate from any Windows key)
ssh-keygen -t ed25519 -C "your.work@email"      # Enter x3
cat ~/.ssh/id_ed25519.pub                        # copy this line
```

Add that public key at **github.com → Settings → SSH and GPG keys → New SSH key**,
then point the repo at SSH and test:

```bash
cd ~/ardupilot
git remote set-url origin git@github.com:<you>/ardupilot.git
ssh -T git@github.com        # "Hi <username>! You've successfully authenticated"
```

Each teammate needs their own key added to their own GitHub account — keys are never
shared.

---

### Notes (both options)

- **Commit author** = `git config user.name/email` (what shows on the commit).
- **Push auth** = the account behind your token / SSH key (`gh auth status` or
  `ssh -T git@github.com` tells you which).
- Being added as a collaborator grants *permission*; each person still authenticates
  with their own token or key.
- VS Code's bottom-left account picture is unrelated to git — ignore it for pushing.

---

## Step 8 — Build

```bash
cd ~/ardupilot
./waf configure --board MatekH743-bdshot
./waf copter
```

First build ~5–15 min; incremental rebuilds after an edit ~30 s (ccache).

Output lands in:

```
~/ardupilot/build/MatekH743-bdshot/bin/
```

| File | Use |
|---|---|
| `arducopter.apj` | Upload via Mission Planner / QGroundControl ("Load custom firmware") |
| `arducopter_with_bl.hex` | Full image incl. bootloader — ST-Link / DFU recovery |
| `arducopter.bin` | Raw image (contents of the `.apj`) |
| `arducopter.abin` | Firmware update over a telemetry radio |
| `arducopter` (no ext.) | ELF w/ debug symbols — `gdb` only, not flashable |

Common extras:

```bash
./waf list_boards            # all board names
./waf configure --board <X>  # switch board
./waf clean                  # wipe build/
./waf --upload copter        # flash directly (needs board USB visible in WSL via usbipd)
```

---

## Working with your own fork

### Remote + branch layout

After forking `skybrush-io/ardupilot` on GitHub to `<you>/ardupilot`,
set the clone up with three remotes:

```bash
cd ~/ardupilot
git remote rename origin skybrush                                   # the build base
git remote add origin    git@github.com:<you>/ardupilot.git         # your fork (you push here)
git remote add ardupilot https://github.com/ArduPilot/ardupilot.git # mainline, reference only
git fetch --all
```

Branches:

| Branch | Purpose | Rule |
|---|---|---|
| `CMCopter-4.6` | clean mirror of `skybrush/CMCopter-4.6` | **never commit here** — it just tracks upstream |
| `an/CMCopter-4.6` | your integration branch — all your hardware changes stack here | this is what you **build and flash** |
| `cm-copter-4.6-<topic>` | one short-lived branch per change | branched off `an/CMCopter-4.6`, deleted after merge |

```bash
git checkout -B CMCopter-4.6 skybrush/CMCopter-4.6
git branch --set-upstream-to=skybrush/CMCopter-4.6 CMCopter-4.6
git checkout -b an/CMCopter-4.6 CMCopter-4.6
git push -u origin an/CMCopter-4.6
```

---

### Making a change and committing it (targeting 4.6)

Replace `<topic>` below with a short name for the change, e.g. `ws2811-tweaks`,
`h743-imu-orientation`, `disable-flowhold`. The branch is then
`cm-copter-4.6-<topic>`.

```bash
# 1. start from the integration branch, up to date
git checkout an/CMCopter-4.6
git pull

# 2. branch for this specific change
git checkout -b cm-copter-4.6-<topic>

# 3. edit files, build, test
./waf copter

# 4. review, then stage + commit (only the files you changed)
git status -sb
git diff
git add <path/to/changed/file> ...
git commit -m "hw: <what changed and why>"
#   - keep one concern per commit
#   - the name/email on the commit come from `git config user.name/email`

# 5. push the branch to your fork
git push -u origin cm-copter-4.6-<topic>
```

---

### Opening a tracking PR (inside your own fork)

Go to `https://github.com/<you>/ardupilot/compare` (or click the yellow
"Compare & pull request" banner). GitHub defaults the base to the *upstream* repo —
**you must change both left dropdowns**:

1. Click **`base repository: skybrush-io/ardupilot`** → choose **`<you>/ardupilot`**
2. Click **`base: master`** → choose **`an/CMCopter-4.6`**
3. Leave the right side: `<you>/ardupilot` / `cm-copter-4.6-<topic>`

Done right it shows just your commit(s) and a handful of changed files. If it shows
hundreds of commits / files, the base is still wrong.

Set a title, **Create pull request**, then **Merge**. Your change is now on
`an/CMCopter-4.6`. Afterwards, get back onto the integration branch:

```bash
git checkout an/CMCopter-4.6
git pull
```

The `cm-copter-4.6-<topic>` branch and its merged PR stay on GitHub as the record
of that change.

---

### Updating from the original (pulling in upstream changes)

"Upstream" here = `skybrush-io/ardupilot` (the repo you forked). To pull its newer
`CMCopter-4.6` commits into your work:

```bash
git fetch skybrush

# refresh the clean mirror
git checkout CMCopter-4.6
git merge --ff-only skybrush/CMCopter-4.6

# replay your hardware commits on top of the new upstream state
git checkout an/CMCopter-4.6
git rebase skybrush/CMCopter-4.6
#   -> fix any conflicts, `git rebase --continue`, then:
./waf copter        # rebuild + test
git push --force-with-lease origin an/CMCopter-4.6
```

`rebase` keeps your changes as a small, visible set of commits on top of upstream —
easiest to carry forward. (Use `git merge skybrush/CMCopter-4.6` instead if several
people share `an/CMCopter-4.6` and force-pushing is a problem.)

To also refresh your fork's copy of `master` / other branches, use the **Sync fork**
button on the fork's GitHub page, or:

```bash
git fetch skybrush
git push origin skybrush/CMCopter-4.6:CMCopter-4.6
```

To see mainline ArduPilot changes for reference: `git fetch ardupilot` then compare
against `ardupilot/master`.

---

### Pulling in changes from true mainline ArduPilot

Only Skybrush normally does this (it's a big merge). If you must:
`git fetch ardupilot`, then merge the relevant `ardupilot/Copter-4.6` into a test
branch and expect conflicts. Prefer letting `skybrush-io/ardupilot` do the upstream
tracking and just following *their* `CMCopter-4.6`.

---

## Editing from Windows

Open the WSL checkout in VS Code with the **WSL** extension:

```bash
cd ~/ardupilot && code .
```

Bottom-left shows a green **`WSL: Ubuntu`** badge. The integrated terminal is a Linux
shell in the project — run `./waf` there.

From File Explorer the same files are at
`\\wsl.localhost\Ubuntu\home\<user>\ardupilot`, but always launch via `code .` from
the WSL terminal so VS Code attaches to Linux (needed for IntelliSense and builds).

If you also keep a checkout on the Windows side, set
`git config --global core.autocrlf input` in Windows git so it doesn't rewrite files
to CRLF.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `/usr/bin/env: 'python3\r'` | Repo has CRLF endings (cloned on Windows / on `/mnt/c`). Re-clone inside `~` in WSL. |
| `ModuleNotFoundError: No module named 'imp'` | Old branch's `waf` + Python ≥ 3.12. Use `CMCopter-4.6` (its `waf` is 3.12-safe), or install Python 3.11. |
| `cannot find -lstdc++_nano` at link | Using the apt `gcc-arm-none-eabi`. Install ArduPilot's toolchain (Step 5) and prepend to PATH (Step 6). |
| `No module named 'empy'` / build config fails | venv not active, or `empy` not `3.3.4`. `source ~/venv-ardupilot/bin/activate` and reinstall. |
| `Permission to skybrush-io/ardupilot.git denied` | Your GitHub account isn't a collaborator on that repo — ask the org to add you, or push to a fork. |
| Very little free flash reported | Expected — this is the Skybrush fork (drone-show, flockctrl, extra fences) on a 2 MB H743; the second flash bank isn't used for code. |
