# my-tweaks – personal branch notes

This branch (`my-tweaks` on `yurkovych/claude-overlay`) adds opt-in settings and fixes on top of
[shengyanlin/claude-overlay](https://github.com/shengyanlin/claude-overlay). The goal: a quick,
throwaway chat window – hotkey in, ask, Ctrl+N, hotkey out.

Every new setting defaults to the **original** behaviour, so the branch behaves like stock until
`config.json` turns things on. This file lists what changed and the update routine;
`my-config.json` next to it is the backup of my config.

## My config.json

The app reads its config from `%LOCALAPPDATA%\claude-overlay\config.json` – outside the repo,
so a copy of mine is kept in **`my-config.json`** here (the app never reads that copy).
If the real one is ever lost: copy `my-config.json` there, rename it to `config.json`,
restart the overlay. When the real config changes, update the copy too.

**Dictionary mode** (in `USER_INSTRUCTIONS`): start a message with `t:` or `translate:` and a word
or phrase in English or Ukrainian, e.g. `t: gank` or `t: перемога`, to get a short dictionary entry.

## Settings this branch adds

| Setting | Default (stock) | What it does |
|---|---|---|
| `SHOW_INTRO` | `true` | `false` = open to an empty chat: no greeting, no "ring on ✻" note |
| `START_POSITION` | `"top-right"` | `"bottom-center"` = open centred just above the taskbar, every launch |
| `RESUME_OFFER` | `true` | now settable in config.json; `false` = no "Resume last conversation" |
| `NEW_CHAT_KEY` | `""` | in-window new-conversation shortcut, e.g. `"ctrl+n"`; matches the physical key, so it works on any keyboard layout (Windows) |
| `NO_LOCAL_TOOLS` | `false` | `true` = no file, shell or subagent tools; answers from knowledge + web search only |
| `USER_INSTRUCTIONS` | `""` | your text appended to the system prompt (Claude and Codex) |
| `LOCAL_CONTEXT` | `false` | `true` = today's date, weekday and time zone in the prompt at each new conversation |
| `INPUT_MAX_LINES` | `2` | input box grows with its text up to this many lines, then scrolls |
| `CHAT_FONT_SIZE` | `15` | conversation + input text size in px (interface untouched); Ctrl +/- still zooms everything |
| `REPLY_HEADER` | `true` | `false` = no "✦ Claude" label above replies |
| `THINKING_LABEL` | `"keep"` | `"transient"` = the "✻ thinking" block disappears once the answer starts |
| `PARAGRAPH_GAP` | `1.0` | blank-line height between paragraphs/turns as a fraction of a text line (0.2-1.0) |
| `COPY_BUTTON` | `"row"` | `"hover"` = floating Copy over the pointed-at reply; `"none"` = no button |
| `JUMP_BUTTON` | `true` | `false` = never show the "↓ Latest" / "↓ New output" pill |

Launch flag: **`--hidden`** starts the app invisible and hidden, for the Startup folder (no flash).

## Fixes (always on)

Input-box selection is stock Tk on purpose (a selected line break paints to the right edge);
earlier workarounds for that were removed as more trouble than they were worth.

- Fonts are copied at their exact pixel size – user bubbles and tables match the chat text
- Right-hand corner resize grips shrunk so they don't bite into the ✕ button / corner
- Version label hides whole instead of clipping; ◢ grip aligned with the input box
- A gap before your next message even without a Copy row under the reply
- The chat stays pinned to the newest text when the input box grows or the window resizes
- Ctrl+A / C / V / X / Z / Y work on any keyboard layout (Windows; matched by physical key)
- Undo / redo (Ctrl+Z / Ctrl+Y) in the input box – Tk has it off by default
- Hand cursor over the send button

Known quirk (stock): after Task Manager / UAC / the Start menu had focus, the global hotkey
can stop responding (the key hook misses key releases). Press Ctrl+Shift+Win+C once to reset it.
Win+C is reserved by Windows, so a proper system hotkey can't be registered for it.

## Startup shortcut (Windows)

PowerShell, once:

```powershell
$s = (New-Object -ComObject WScript.Shell).CreateShortcut("$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\Claude Overlay (hidden).lnk")
$s.TargetPath = "C:\Users\Jonny\AppData\Local\Python\pythoncore-3.14-64\pythonw.exe"
$s.Arguments = '"C:\Users\Jonny\DEV\claude-overlay\claude_overlay.py" --hidden'
$s.WorkingDirectory = "C:\Users\Jonny\DEV\claude-overlay"
$s.IconLocation = "C:\Users\Jonny\DEV\claude-overlay\claude_overlay_2.ico"
$s.Save()
```

## Updating from upstream

Remotes: `upstream` = shengyanlin (updates come from here), `origin` = my fork.
The in-app update button doesn't work on this branch (`update.cmd` only runs on `main`).

**1. Preview first** – what would the update bring?

```
git fetch upstream
git log --oneline main..upstream/main
git diff --stat main upstream/main
```

If it touches `claude_overlay.py`, `config.py` or `worker.py`, review the diff against these
tweaks before updating.

**2. Update** – double-click `my-update.cmd`: switches to `main`, runs the official `update.cmd`,
switches back, rebases `my-tweaks` on top and pushes it to the fork. Stops with instructions on a
conflict.

`my-update.cmd` is deliberately **not** tracked (it must survive the switch to `main` while
running; it's listed in `.git/info/exclude`). If it's lost, recreate it from this:

```bat
@echo off
setlocal
cd /d "%~dp0"
echo ============================================================
echo   my-update: stock update on main, then re-apply my-tweaks
echo ============================================================
echo.

git diff --quiet HEAD
if errorlevel 1 (
  echo [X] You have uncommitted changes. Commit or stash them first.
  pause & exit /b 1
)

git switch main
if errorlevel 1 ( pause & exit /b 1 )

rem The official updater: pulls upstream/main (fast-forward only), refreshes packages,
rem checks the app loads. It pauses at the end - press a key to continue here.
cmd /c call "%~dp0update.cmd"
if errorlevel 1 (
  echo.
  echo [X] The stock update failed - you are still on main.
  echo     Fix what it said, then run:  git switch my-tweaks
  pause & exit /b 1
)

git switch my-tweaks
if errorlevel 1 ( pause & exit /b 1 )

echo.
echo Re-applying my-tweaks on top of the new main...
git rebase main
if errorlevel 1 (
  echo.
  echo [X] Rebase conflict: the update changed the same lines as my-tweaks.
  echo     Either fix the files and run:  git add -A  then  git rebase --continue
  echo     or give up for now with:       git rebase --abort
  pause & exit /b 1
)

git push --force-with-lease origin my-tweaks
echo.
echo ============================================================
echo   [OK] Updated, tweaks re-applied, fork backed up.
echo   Restart the overlay for the changes to take effect.
echo ============================================================
pause
```
