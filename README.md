# Mocha Powerline

A Claude Code status line in the Catppuccin Mocha palette — and, unlike most status lines,
it shows how much of your Claude.ai subscription you have already burned through.

<img src="docs/statusline.png" width="676" alt="Mocha Powerline status line: model, directory, git branch, context window, 5h and 7d subscription limits, clock">

*Above: 18% of the context window in use, 7% of the 5-hour session limit and 34% of the weekly
one. Once a window passes 80%, a `↻` countdown to its reset appears beside the number.*

| Segment | What it shows |
|---|---|
| model | Display name of the current model, followed by the reasoning effort level when Claude Code reports one — `low`, `medium`, `high`, `xhigh`, `max`. The value is printed as it arrives, so new levels appear on their own |
| directory | Name of the working directory |
| git | Branch, plus `✗` when the working tree is dirty. Hidden outside a repository |
| python | `python3` version, only in Python projects |
| context | Percentage of the context window in use. The background turns amber at `CTX_WARN` and red at `CTX_DANGER` |
| **account** | **Name of the active `claude-acc` slot — `work`, `personal` — at the head of the limits segment. Hidden when `claude-acc` is not in use. See [Account slot](#account-slot)** |
| **limits** | **Subscription usage: `5h` is the session window, `7d` the weekly one. The background follows the worse of the two — amber at `LIMIT_WARN`, red at `LIMIT_DANGER` — and past `LIMIT_COUNTDOWN` the time until reset appears (`↻2h`)** |
| clock | Hours and minutes |

Segments disappear when they have nothing to say, and the neighbours re-join seamlessly.

---

## Prerequisites

Four things, only the first of which is strictly required:

| | Needed for | Required? |
|---|---|---|
| **`jq`** | Parsing the JSON that Claude Code pipes to the status line | Yes — nothing renders without it |
| **A Nerd Font** | The powerline separators and the segment glyphs | Yes, or you get tofu boxes |
| **A truecolor terminal** | The 24-bit Catppuccin palette | Yes, or the colors are wrong |
| **A Claude.ai subscription** | The limits segment only | No — everything else works on API billing |

`git` and `python3` are optional: their segments simply stay hidden when the tools or the
project markers are absent.

The script is POSIX `sh` with no bashisms — verified to produce byte-identical output under
`dash`, `bash`, and `zsh`.

### macOS

```sh
brew install jq
brew install --cask font-jetbrains-mono-nerd-font
```

Then point your terminal at the font:

- **iTerm2** — Settings → Profiles → Text → Font → *JetBrainsMono Nerd Font*
- **Ghostty** — add `font-family = "JetBrainsMono Nerd Font"` to `~/.config/ghostty/config`
- **WezTerm** — `config.font = wezterm.font("JetBrainsMono Nerd Font")` in `~/.wezterm.lua`
- **Kitty** — `font_family JetBrainsMono Nerd Font` in `~/.config/kitty/kitty.conf`
- **Terminal.app** — works, but needs macOS 14+ for truecolor; the other four are a better bet
- **VS Code integrated terminal** — set `"terminal.integrated.fontFamily": "JetBrainsMono Nerd Font"`

### Linux

Install `jq` with your package manager:

```sh
sudo apt install jq        # Debian, Ubuntu
sudo dnf install jq        # Fedora, RHEL
sudo pacman -S jq          # Arch
sudo zypper install jq     # openSUSE
apk add jq                 # Alpine
```

For the font, Arch has a package:

```sh
sudo pacman -S ttf-jetbrains-mono-nerd
```

Everywhere else, install it by hand:

```sh
mkdir -p ~/.local/share/fonts
curl -fLo /tmp/JetBrainsMono.zip \
  https://github.com/ryanoasis/nerd-fonts/releases/latest/download/JetBrainsMono.zip
unzip -o /tmp/JetBrainsMono.zip -d ~/.local/share/fonts/JetBrainsMonoNerdFont
fc-cache -f
fc-list | grep -i "JetBrainsMono Nerd" | head -3   # should print something
```

Then select *JetBrainsMono Nerd Font* in your terminal's profile settings.

### Windows

**WSL is the recommended path.** Install a distro, follow the Linux instructions inside it,
and run Claude Code from the WSL shell. The font, however, is installed on the *Windows* side —
WSL terminals render with the host's font. Download
[JetBrainsMono.zip](https://github.com/ryanoasis/nerd-fonts/releases/latest/download/JetBrainsMono.zip),
select all `.ttf` files, right-click → **Install for all users**, then set it in
Windows Terminal: Settings → your profile → Appearance → Font face → *JetBrainsMono Nerd Font*.

Native Windows works too, but needs a POSIX shell — `sh.exe` from
[Git for Windows](https://gitforwindows.org/) — on `PATH`. Install the dependencies with
your package manager of choice:

```powershell
scoop install jq                      # or: choco install jq
scoop bucket add nerd-fonts
scoop install JetBrainsMono-NF
```

This is the least-tested combination of the three; if the status line misbehaves on native
Windows, WSL will save you the debugging.

### Check your terminal before installing

```sh
printf '\033[38;2;203;166;247m truecolor works \033[0m\n'
printf '     󰾆  \n'
```

The first line should be **mauve** — not white, and not a stray escape sequence. The second
should show, in order: a solid triangle, a bolt, a folder, a branch, the Python logo, a gauge,
a bar chart, and a clock. Those are exactly the glyphs the status line uses, so empty rectangles
here mean empty rectangles there. If the colors fail, switch terminals; if the glyphs fail, the
font is installed but not selected in your terminal profile.

---

## Install

```sh
claude plugin marketplace add Madmadmax/claude-statusline
claude plugin install mocha-powerline@mocha-powerline
```

A `SessionStart` hook wires the status line into `~/.claude/settings.json` when your next
session starts. To skip the wait, run `/mocha-powerline:enable` right away.

If `settings.json` already points at a status line of your own, the plugin leaves it alone and
says so. Remove the `statusLine` key first if you want to switch over.

### Without the plugin

Copy `statusline.sh` anywhere you like and register it yourself:

```sh
mkdir -p ~/.claude
curl -fLo ~/.claude/statusline-command.sh \
  https://raw.githubusercontent.com/Madmadmax/claude-statusline/main/statusline.sh

[ -f ~/.claude/settings.json ] || echo '{}' > ~/.claude/settings.json
jq '.statusLine = {"type":"command","command":"sh \"$HOME/.claude/statusline-command.sh\""}' \
  ~/.claude/settings.json > ~/.claude/settings.json.new \
  && mv ~/.claude/settings.json.new ~/.claude/settings.json
```

### Verify it without launching Claude

Feed the script the same JSON shape Claude Code sends. You should get a colored line back:

```sh
echo '{"cwd":"'"$PWD"'","model":{"display_name":"Opus 5"},
"context_window":{"used_percentage":37.2},
"rate_limits":{"five_hour":{"used_percentage":12.5,"resets_at":1786000000},
"seven_day":{"used_percentage":63.1,"resets_at":1786400000}}}' \
  | sh ~/.claude/statusline-command.sh
```

No restart is needed either way — the status line is rebuilt on every frame.

## Uninstall

```
/mocha-powerline:disable                # remove the status line, keep the plugin
claude plugin disable mocha-powerline   # turn the plugin off entirely
```

`/mocha-powerline:disable` only deletes the `statusLine` key; the `SessionStart` hook will put
it back next session unless you also disable the plugin.

## Account slot

`claude-acc` is a small helper that keeps several pre-authorized Claude accounts side by side
and switches between them without a browser: one slot per account, and `~/.claude/auth/.current`
names the slot in use. The line reads that file and prints the name at the head of the limits
segment, so whose `5h` and `7d` windows are on screen is never a guess.

Two details:

- With no such file — `claude-acc` not installed, or no slot chosen yet — nothing is shown and
  the segment behaves exactly as before.
- A bare `/login` moves the live account without touching the slot, which would leave the name
  lying. The slot's stored email is compared against `oauthAccount.emailAddress` in
  `.claude.json`, and a `✗` is appended when the two disagree.

`CLAUDE_CONFIG_DIR` is honoured for both files when it is set.

## Troubleshooting

| Symptom | Cause |
|---|---|
| Nothing appears at all | `jq` is missing, or `statusLine` never made it into `settings.json` — run `/mocha-powerline:enable` and read what it says |
| Boxes instead of separators and icons | The terminal is not using a Nerd Font, or the font was installed but not selected in the profile |
| Colors are flat or wrong | No truecolor. Check with the `printf` test above; `echo $COLORTERM` should print `truecolor` or `24bit` |
| The limits segment is missing | Expected before the first API response of a session, on API-key billing, and on plans without subscription limits. With `claude-acc` in use the segment still shows the slot name |
| The git segment is missing | You are not inside a git repository, or `git` is not on `PATH` |
| The plugin refuses to wire itself | Another status line is already configured — that is deliberate. Delete the `statusLine` key and re-run `/mocha-powerline:enable` |

## Customizing

Everything lives in `statusline.sh`:

- **Color thresholds** — `CTX_WARN` / `CTX_DANGER` and `LIMIT_WARN` / `LIMIT_DANGER` at the top
  of the file.
- **Reset-time threshold** — `LIMIT_COUNTDOWN`, independent of the colour thresholds. Set it to
  `0` to always show the time until reset.
- **Alarm colors** — `warn_bg` and `danger_bg`. They deliberately sit outside the Catppuccin
  palette: the bar is pastel throughout, so an alarm in that range blends into its neighbours
  instead of warning. For the same reason they need no Latte counterparts.
- **Remove a segment** — delete its `if` block near the end of the file. Neighbours re-join on
  their own; the separator color comes from `last_bg`.
- **Light palette (Catppuccin Latte)** — replace the color block, which is plain `R;G;B`:
  `crust='76;79;105'`, `red='210;15;57'`, `peach='254;100;11'`, `yellow='223;142;29'`,
  `green='64;160;43'`, `sapphire='32;159;181'`, `mauve='136;57;239'`, `lavender='114;135;253'`.

## How it works

There is deliberately **no** `statusLine` field in the plugin manifest — Claude Code does not
recognize one, and `claude plugin validate` says as much: *"Unknown field 'statusLine'. Claude
Code ignores it at load time."* A status line can only be set through the `statusLine` key in
`settings.json`, so the plugin does that itself:

- `hooks/wire-statusline.sh` runs on `SessionStart` and writes an **absolute** path to
  `statusline.sh` into `~/.claude/settings.json` — `${CLAUDE_PLUGIN_ROOT}` is expanded in hook
  and command definitions, but *not* inside `settings.json`, so the path is resolved at wiring
  time.
- A status line that is not ours is never overwritten.
- Writes are idempotent: if everything is already in place, the file is not touched.
- On broken JSON or a missing `jq`, the hook exits 0 without changes, so it can never break
  session startup.
- Only `.statusLine` is edited; every other key in `settings.json` is preserved.

## Where the data comes from

Claude Code pipes a JSON object to the status-line command on stdin. Subscription usage lives at:

```json
"rate_limits": {
  "five_hour":  { "used_percentage": 12.5, "resets_at": 1786000000 },
  "seven_day":  { "used_percentage": 63.1, "resets_at": 1786400000 }
}
```

`used_percentage` is 0–100; `resets_at` is Unix epoch seconds. The block is present only for
subscribers, and only after the session's first API response. Run `/statusline` inside Claude
Code to see the full field schema.

## License

MIT
