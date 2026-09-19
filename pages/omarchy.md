# Omarchy — working knowledge

Omarchy is an opinionated Arch + Hyprland desktop distribution. These are notes from
actually driving it, not from its documentation. Where the documentation and this page
disagree, this page is describing what happened.

**What this was true against:** Hyprland 0.56.2, installed from an ISO downloaded
2026-09.
**Check before trusting the dated parts:** `hyprctl version`.

If that reports something newer, everything under *Perishable* below is a claim about
the past, not about your machine. The *Durable* section does not depend on the version.

---

## Durable — how Omarchy is built

This part describes the shape of the system. It goes stale only if the distribution is
redesigned.

### Why it suits an AI-heavy workflow

It ships lazy launchers for `claude`, `codex`, `opencode`, `gemini`, `copilot` and `grok`
in `~/.local/bin`, with a single keybind and a token-usage panel. If you use several coding
agents, three or four of them are first-class in this OS rather than things you install and
wire up yourself.

It also has voice typing built in, and hardware profiles — `omarchy-hw-asus-rog` is one of
them, so ROG laptops get sane defaults without hunting for kernel parameters.

### Graphical commands need the Wayland environment exported first

Over SSH, or from any non-graphical context, a graphical command will fail with an opaque
error until you set these:

```bash
export XDG_RUNTIME_DIR=/run/user/$(id -u)
export WAYLAND_DISPLAY=wayland-1
```

This is the single most common cause of "the command does nothing and says nothing". It
is how Wayland works, not a bug — it will not be fixed and will not change.

### Screenshots and synthetic keystrokes

Wayland, so the X11 tools do not apply:

```bash
grim screenshot.png              # capture the screen
wtype -M logo -k space -m logo   # press a key combination (here: Super+Space)
```

Both need the two exports above.

### Turning the screensaver off

```bash
omarchy-toggle-screensaver
```

Worth knowing it exists — the setting is not where you would look for it in a normal
desktop environment.

### Installing: identify the target disk by device type, not by size

It installs to a normal `sd*` or `nvme*` block device. **Identify the target by device
type, not by size** — a 466 GB target and a 477 GB system disk are indistinguishable at a
glance, and getting it wrong destroys the wrong drive.

### Running it in a VM before committing

It runs well under QEMU with virgl GPU acceleration on integrated graphics. 4 GB of memory
and 6 cores boots in roughly 30 seconds.

**Do not force QEMU onto a discrete NVIDIA card** via a per-application graphics
preference. It causes persistent screen flashing. Integrated graphics with virgl is both
faster to set up and stable.

### Service management — the reason this OS is worth the move for agent work

Local AI services that load a model at startup are expensive to leave running and annoying
to start by hand. Linux solves this natively with **socket activation**: systemd holds the
port, starts the service on the first connection, and stops it again after an idle period.

The service is then only resident when something is actually asking it a question, and
nothing has to remember to start it. There is no clean equivalent on Windows — a scheduled
task at logon leaves it resident all day, and no task at all means it is simply missing
when you need it.

If you are weighing this distribution for a machine that runs local model services, that
is the concrete argument.

---

## Perishable — check these before relying on them

Each item says what it was true against and how to check. A bug is a statement about one
release; it gets fixed, and then this section is wrong.

### `hyprctl dispatch` is broken

- **True against:** Hyprland 0.56.2
- **Check:** `hyprctl version`, then try the documented syntax once

It parses its arguments as Lua and rejects the documented syntax. Every example you find
online will fail. **Do not spend time on it.**

Launch applications this way instead:

```bash
setsid <app> &
```

`setsid` works regardless, so the workaround stays correct even after the bug is fixed —
it is only the "do not bother trying" advice that expires.

### The ISO is about 5.8 GB

- **True against:** the ISO downloaded 2026-09
- **Check:** the size listed on the download page

Several write-ups claim roughly 2 GB. They were out of date, and the difference matters
when you are choosing a USB stick. Image sizes move with every release — read the number
rather than trusting this line.

### Dictation / voice typing

- **Checked against:** official Omarchy manual on 2026-09-18.
- **Install/setup:** open the Omarchy menu with `Super + Space`, then go to `Install > AI > Dictation`.
- **Toggle dictation:** `Super + Ctrl + X` starts/stops dictation.
- **Push-to-talk:** hold `F9` while speaking, then release it when done.
- **Status indicator:** while dictation is active, Omarchy shows a dictation indicator in the center indicator cluster of the top bar.

These bindings are documented in the Omarchy hotkeys manual. Re-check if the installed Omarchy/Hyprland version has changed substantially.

### ChatGPT desktop app

- **Verified on:** 2026-09-18 on the current Omarchy/Arch install.
- OpenAI's official Arch install path worked successfully and installed the `chatgpt-bin` package plus the needed graphics dependency.
- After installation, `chatgpt` launched the desktop app successfully from a terminal.
- The installed app opened normally on Omarchy and exposed the desktop ChatGPT/Codex interface.
- Treat this as version-sensitive packaging knowledge: re-check OpenAI's Linux install instructions if the package/install mechanism changes.

