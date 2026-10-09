# ccusagebar

Your Claude plan limits, always visible on the MacBook Pro Touch Bar, or around the
notch on MacBooks that have one instead.

![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)
![Platform: macOS](https://img.shields.io/badge/platform-macOS-lightgrey.svg)
![Language: Swift](https://img.shields.io/badge/language-Swift-orange.svg)

![Touch Bar: ✳ 5h 45% · 23:10, week 12% · Mon 06:00, Fable 15%, then the Control Strip](docs/touchbar-spark.png)

![Menu bar: ✳ 5h 16% left of the notch, wk 14% · F 19% right of it](docs/notch.png)

## Why

Claude Code's `/usage` shows how much of your plan limits you have used, but only
when you ask. ccusagebar keeps the same numbers on screen all the time and can
answer Claude Code's prompts from the notch. You build it locally, so macOS
Gatekeeper never quarantines it, unlike MTMR (whose last release isn't notarized,
so current macOS blocks it) and BetterTouchTool (which is paid).

## Features

- The numbers Claude Code's `/usage` shows: the 5-hour session limit, the weekly
  limit, and any per-model weekly limit (Fable, today).
- Touch Bar line in the app area, with brightness and volume left in the Control
  Strip; on a Mac with a notch, a short label in black "ears" on either side of it.
- A limit turns amber at 75% and red at 90% (or sooner, if the usage API itself
  calls it high). Extra usage shows in the line while it is switched on, and a
  credit grant once you start spending it.
- Details panel: every limit with a bar and time until it resets, extra usage and
  credits, and the Claude Code sessions running on this Mac, with the ones waiting
  on you first. Click a session to bring its terminal forward (Warp, iTerm2,
  Terminal, VS Code: whichever app it runs under).
- Session status in the notch's left ear: the session's name and how long it has
  been working, behind a ✳ that breathes while Claude works and turns orange when
  the session waits on you.
- Optional: answer Claude Code permission prompts, questions and plans from the
  notch.
- Notifications when a limit reaches 80% and 95%, and when its window resets.
- Last known numbers are shown marked `(stale)` when a fetch fails.

![Details: each limit with a bar and reset countdown, credits, and Claude Code sessions working or waiting](docs/details.png)

## Requirements

- A Mac with a Touch Bar or a notch (on a Mac with neither, nothing is shown unless
  you set the mode yourself, see [Configuration](#configuration)).
- Xcode command line tools and Node.
- Claude Code, logged in with a Claude subscription.

## Installation

Clone the repository and run:

```sh
./install.sh      # builds, copies to ~/Applications, adds a login item
```

To build without installing:

```sh
./build.sh        # builds CCUsageBar.app in the repository
```

## Usage

### Touch Bar

The line takes the app area of the Touch Bar; brightness and volume stay in the
Control Strip. The system ✕ can close it; it comes back on the next app switch, or
tap ✦ in the Control Strip. Tap the line to open the details.

It covers the Touch Bar's Esc, so it suits Macs with a physical Esc key (or an
external keyboard).

### Notch

On a MacBook with a notch the short label sits in the ears: `✳ 5h 41%` on the left,
`wk 11% · F 14%` on the right, and an orange `● 1` when a Claude Code session is
waiting on you. Hover to drop the full line with reset times below the notch; click
to grow it into the details. It follows the built-in display, and hides while the
lid is closed. The ears cover a little menu bar next to the notch.

The notch line is newer than the Touch Bar one. If it sits wrong on your Mac,
[open an issue](https://github.com/HikvIneH/ccusagebar/issues/new/choose) with a
screenshot and the model.

### Details

Tap the Touch Bar line, or click the notch, and the details drop from the top of
the screen. The refresh button (↻) refreshes; a click anywhere else puts it away.

### Answering Claude Code from the notch

```sh
./install-hook.sh           # after ./install.sh; ./install-hook.sh remove to undo
```

![Notch panel asking to allow a Bash command, with Terminal, Deny and Allow](docs/prompt.png)

This adds a `PermissionRequest` hook to `~/.claude/settings.json` (the old file is
kept as `settings.json.bak-ccusagebar`). Then, when a session needs you, the notch
opens on it, with a sound:

- a tool to allow: the command, the edit as a diff, the URL; **Allow** or **Deny**
- a question (`AskUserQuestion`): click an option, tick several, or type your own
- a plan (`ExitPlanMode`): rendered; **Approve**, or **Keep planning** with what to change

**Terminal** hands it back to the terminal and brings that forward. While the app
is not running the hook prints nothing and Claude Code asks in the terminal as
before; a prompt left for an hour goes back to the terminal too.

## Configuration

Settings are macOS defaults under `com.hikvineh.ccusagebar`.

**Mode.** By default it shows whichever the Mac has: the Touch Bar line on a Touch
Bar Mac, the notch line on a screen with a notch, and nothing where there is
neither. To pick one yourself:

```sh
defaults write com.hikvineh.ccusagebar mode notch      # or: touchbar
defaults delete com.hikvineh.ccusagebar mode           # back to automatic
```

Then quit and reopen `CCUsageBar`. `mode notch` on a screen without a notch puts
the same line in the middle of the menu bar.

**Alerts.** A notification when a limit reaches 80% and again at 95%, once per
window, and one when that window resets. macOS asks for permission the first time.
To change the thresholds, or turn them off:

```sh
defaults write com.hikvineh.ccusagebar alerts -array 70 90   # other thresholds
defaults write com.hikvineh.ccusagebar alerts -array         # no alerts
```

## How it works

- `ccusage-line.sh` reads Claude Code's OAuth token from the Keychain and calls
  the usage endpoint Claude Code itself uses (`/api/oauth/usage`). That endpoint
  rate-limits hard, so the answer is cached in `~/Library/Caches/ccusagebar.json`,
  fetched at most every 5 minutes, and shown as `(stale)` if a fetch fails. It
  prints two lines for people and a third, JSON, for the app.
- `main.swift` presents a system-modal Touch Bar through the private
  `NSTouchBar presentSystemModalTouchBar:placement:systemTrayItemIdentifier:` and
  anchors it on a Control Strip item (`DFRElementSetControlStripPresenceForIdentifier`),
  the same private calls MTMR and Pock use. It re-presents itself on every app
  switch and refreshes every 5 minutes.
- `notch.swift` is a borderless panel above the menu bar, sized from
  `NSScreen.auxiliaryTopLeftArea` / `auxiliaryTopRightArea`. Public API only.
- `details.swift` is the panel that drops below it; `alerts.swift` posts through
  `UNUserNotificationCenter`.
- `usage.swift` reads the Claude Code sessions from `~/.claude/sessions`, one file
  per running `claude`, every 3 seconds. Local files, no network.
- `bridge.swift`: the hook runs the app's own binary as `CCUsageBar --hook`, which
  passes the request over a Unix socket (`~/Library/Caches/ccusagebar.sock`, this
  user only) to the running app and prints its answer. Esc in the session kills the
  hook, which closes the socket, which takes the card away. `prompt.swift` draws
  the cards.
- It's an `LSUIElement` agent: no Dock icon, no menu bar.

Undocumented endpoint, private APIs and Claude Code's internal session files: no
App Store, and any of them could break.

## Uninstall

1. If you installed the hook, run `./install-hook.sh remove`.
2. Quit `CCUsageBar`, delete `~/Applications/CCUsageBar.app`, and remove it from
   System Settings → General → Login Items.

## Disclaimer

Unofficial: not made by, endorsed by or affiliated with Anthropic. Not related to
the [ccusage](https://github.com/ccusage/ccusage) project either; the "cc" is for
Claude Code.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md): how to build, what to check, commit format.

## License

[MIT](LICENSE)
