# NFS Most Wanted (2005) split-screen on Linux with PartyDeck

🇧🇷 [Leia em português](README.md)

An attempt to run **Need for Speed: Most Wanted (2005, PC, patch 1.3)** in local multiplayer (two instances on the same PC, linked over LAN) on **Linux**, using [PartyDeck](https://github.com/partydeck/partydeck), since [Nucleus Co-op](https://www.splitscreen.me/) only works on Windows.

> **Status: partially working.**
> The LAN lobby works (both instances see each other and join the same room). **The race does not start:** the host stays on "Waiting for players to connect" and player 2 keeps loading. The most likely cause is documented below, but **the problem is not solved yet**.

This repository documents what was discovered, what works, what was ruled out, and where help is welcome.

---

## Test environment

| Item | Value |
|---|---|
| Distro | CachyOS (Arch), KDE Plasma 6 (Wayland) |
| Game | NFS Most Wanted 2005 **Black Edition**, patch **1.3** |
| Launcher | PartyDeck v0.8.7 (tar.gz, SteamOS x86_64) |
| Runtime | GE-Proton via `umu-run` |
| GPU | AMD Radeon RX 5700 XT |

> You need **your own legitimate copy** of the game with the official EA patch 1.3. This repository does not distribute the game or any third-party files.

---

## How the game does multiplayer (what the Nucleus handler teaches)

The PC version has no split-screen. It has **LAN** for 2 to 4 players. Nucleus Co-op works by opening several instances and linking them over LAN. Reading the Nucleus handler (`handler.js`, version 10 from the splitscreen.me hub), the key points are:

1. **Multiple instances:** the *NFSMW Extra Options* mod (`NFSMWExtraOptions.asi`, loaded by an ASI loader `winmm.dll`) with `AllowMultipleInstances = 1`.
2. **Each instance needs its own `server.dll` and `server.cfg`.** The handler makes a real copy of `server.dll` (not a link) and gives each instance a different `PORT`.
3. **Windowed mode** (`WindowedMode=2` in the Extra Options ini) and an adjusted resolution.
4. **Manual in-game flow:**
   - Instance 1: *LAN → Create Server* (name = player name) → *Create Room*.
   - Instance 2: *LAN → Create Server* → search for the room and *Join* (may take more than one attempt).

---

## What works

### Step 1: the game runs on its own on Linux
It works with GE-Proton (via `umu-run`) and a dedicated Wine prefix.

### Step 2: two instances in the **same folder** crash
In PartyDeck, both instances use the same game folder. When the second instance creates the LAN server, the game **closes**. With the Proton log on (`PROTON_LOG=1`) this shows up:

```
wine: Unhandled page fault on read access to 00000050 at address 03E39D56
```

The address is inside `server.dll` (loaded at `03E00000`). The same crash happened when running one instance outside PartyDeck while another was already open in the same folder, so **the cause is not PartyDeck, gamescope or the controllers**.

### Step 3: separate folders fix the lobby
Running each instance from its **own folder**, with its own `server.dll` and `server.cfg` and different ports, fixed the crash.

Create the folders (use `bash`; these commands fail in `fish`). Adjust `SRC` to your game folder:

```bash
SRC="$HOME/Games/Need For Speed Most Wanted Black Edition"
IP=$(hostname -I | awk '{print $1}')
for n in 1 2; do
  D="$HOME/Games/NFSMW-p$n"
  rm -rf "$D"; mkdir -p "$D"
  cp -as "$SRC"/. "$D"/                 # symlinks for the rest of the game
  rm -f "$D/server.dll" "$D/server.cfg"
  cp "$SRC/server.dll" "$D/server.dll"  # REAL copy
  grep -vE '^(PORT|LAN_THROTTLE|ADDR|TRUST_MATCH)=' "$SRC/server.cfg" > "$D/server.cfg"
  p=$((9899+n))                          # 9900 and 9901
  printf 'PORT=%s\nLAN_THROTTLE=6\nADDR=%%%%bind("%s:%s")\nTRUST_MATCH=%%%%bind("%s")\n' "$p" "$IP" "$p" "$IP" >> "$D/server.cfg"
done
```

Each `server.cfg` ends with something like this (instance 2 uses port 9901):

```
PORT=9900
LAN_THROTTLE=6
ADDR=%%bind("192.168.x.x:9900")
TRUST_MATCH=%%bind("192.168.x.x")
```

> If the original `server.cfg` has an active `ADDR=` line (EA's default), comment it out with `#`.

### Step 4: making PartyDeck pick the right folder
The PartyDeck handler points to **one** game folder (`path_gameroot`). The solution was to use a `.bat` file as the executable, which picks `NFSMW-p1` or `NFSMW-p2` depending on launch order, using a lock folder (`md` can only succeed once):

`launch.bat` (in the original game folder):

```bat
@echo off
set "BASE=Z:\home\YOUR_USER\Games"
md "%BASE%\.nfsmw-slot1" 2>nul
if not errorlevel 1 (set "SLOT=1") else (set "SLOT=2")
cd /d "%BASE%\NFSMW-p%SLOT%"
start /wait "" speed.exe
if "%SLOT%"=="1" rd "%BASE%\.nfsmw-slot1"
```

(The file needs CRLF line endings.) In PartyDeck, edit the handler and set **Executable file** to `launch.bat`.

Handler (`handler.json`), relevant fields:

```json
{
  "name": "Need for Speed Most Wanted",
  "version": "1.3",
  "spec_ver": 3,
  "path_gameroot": "/home/YOUR_USER/Games/Need For Speed Most Wanted Black Edition",
  "exec": "launch.bat",
  "use_goldberg": false
}
```

**Result:** both instances open in split-screen and join the same room ("game room").

**Watch out for the lock:** if PartyDeck closes abruptly, the `.nfsmw-slot1` folder may be left behind, and both instances will go to the same folder (and one will crash). Before launching again: `rmdir ~/Games/.nfsmw-slot1`.

---

## The open problem: the race does not start

With both instances in the room, when the race starts the host stays on "Waiting for players to connect" and player 2 keeps loading.

### Evidence collected

**1. UDP port 3658 is contested.** With `PROTON_LOG=1` and `WINEDEBUG=+winsock`, instance 1's log shows:

```
bind socket 0x23c, addr { family AF_INET, address 0.0.0.0, port 3658 }
bind failed, status 0xc0000043
```

The game carries on anyway, sending through a socket with no defined port.

**2. The port changes owner.** With `ss -tulnp`, before the race starts port 3658 belonged to instance 2. After it started, it moved to instance 1, and instance 2 was left without it.

**3. Network traffic (`tcpdump`).** During loading, almost only this shows up, in a loop, on the `lo` interface:

```
IP 192.168.x.x.3658 > 192.168.x.x.3658: UDP, length 21
```

That is, packets leaving 3658 and arriving at 3658 itself. No UDP traffic going from one instance to the other appears. The discovery broadcasts (`255.255.255.255:9999`, 384 bytes) work, and that is how the lobby sees itself.

### Main hypothesis (unconfirmed)
During the race, the game uses **UDP port 3658** and identifies other players by **IP**. Since both instances share the same IP and fight over the same port, the host's packets always land on the same instance, and the other never receives anything.

This is consistent with reports about Wine: the behavior of shared UDP ports differs from Windows (for example, WineHQ bug 51097 describes only the last bound socket receiving packets). It also resembles an old Warcraft III case with two copies on the same machine, where `bind` failed and the fix was to intercept it and enable `SO_REUSEADDR`.

### What has been ruled out
| Hypothesis | Test | Result |
|---|---|---|
| Port 9999 conflict | `ss` and `tcpdump` analysis | It is only discovery broadcast, and it works |
| Firewall (UFW) | rule allowing UDP 9999 on the LAN | No change |
| `demangler.ea.com` server hanging the connection | blocking the host in `/etc/hosts` | No change |
| Mixed-up keyboard/controllers | one controller per instance | Not the cause of the hang |
| Shared `server.dll`/`server.cfg` | separate folders | Fixed the **lobby**, not the race |

---

## Possible paths (help welcome)

1. **Two computers on the same network.** Each has its own IP, so port 3658 does not conflict. It should work, since that is what the game was designed for. (Not yet tested by the author.)
2. **An `.asi` plugin that intercepts `bind`.** The game already loads `.asi` plugins (via the ASI loader). A plugin could make each instance use its own address (for example `127.0.0.1` and `127.0.0.2`) or enable port sharing. Since the game's LAN server also uses addresses, this may require changes in several places. **Untested.**
3. **A virtual network per instance** (network namespaces). It would give each instance its own IP, but it is invasive and may break communication with gamescope. **Not recommended as a first attempt.**

If you know the Most Wanted protocol (port 3658, "demangler"), know how to work with `bind` under Wine/Proton, or have already made this work on Linux, please open an *issue* or *pull request*.

---

## How to reproduce the diagnostics

Wine network log to a file, running outside PartyDeck (one terminal per instance):

```bash
mkdir -p ~/nfslogs/p1 ~/nfslogs/p2

# instance 1
cd ~/Games/NFSMW-p1 && env WINEPREFIX=$HOME/.local/share/partydeck/prefixes/1 \
  PROTONPATH=GE-Proton PROTON_LOG=1 PROTON_LOG_DIR=$HOME/nfslogs/p1 \
  WINEDEBUG=+winsock umu-run speed.exe

# instance 2
cd ~/Games/NFSMW-p2 && env WINEPREFIX=$HOME/.local/share/partydeck/prefixes/2 \
  PROTONPATH=GE-Proton PROTON_LOG=1 PROTON_LOG_DIR=$HOME/nfslogs/p2 \
  WINEDEBUG=+winsock umu-run speed.exe
```

Search for port failures:

```bash
grep -inE "bind|10048|ADDRINUSE|3658" ~/nfslogs/p1/steam-default.log | head
```

Open ports and UDP traffic:

```bash
ss -tulnp | grep -iE 'speed|wine'
sudo tcpdump -ni any 'udp and not port 53 and not port 5353 and not port 1900 and not port 67 and not port 68' -c 60
```

---

## Notes about PartyDeck

- With `disable_mount_gamedirs: true` (enabled in this setup), the `overlay` folders and per-profile saves inside the game folder are **not** applied, according to PartyDeck's release notes. That is why the `launch.bat` was needed.
- PartyDeck `.pd2` handlers are zip files with a `handler.json` and optional `overlay/` and `profile_copy_gamesave/` folders.
- The keyboard and mouse are sent to both instances at the same time. Use **one controller per player**.

---

## Credits

- **PartyDeck**: [partydeck/partydeck](https://github.com/partydeck/partydeck) (author: wunnr).
- **Nucleus Co-op handler** for Most Wanted (used as a reference): maintained by Talos91 on the [Game Handlers Hub](https://hub.splitscreen.me/).
- **NFSMW Extra Options** and **WidescreenFix** (game mods): ThirteenAG and contributors.
- **Nucleus Co-op**: Lucas Assis and the community.

This project is a compatibility study and has no affiliation with EA or the authors above.
