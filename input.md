# Korean (Hangul) Input on Fedora 45 KDE Plasma: Windows-Style Setup

Fcitx 5 + fcitx5-hangul, started by KDE and tuned to behave like the Windows 10 Korean IME.

> Checked in November 2026 against the versions Fedora 45 ships (Plasma 6.8, Qt 6.11.2, fcitx5 5.1.23, fcitx5-hangul 5.1.11) and verified on a Fedora 45 KDE PC with the Step 4 checks.

---

## What you get

- **Right Alt is 한/영.** It switches the moment you press it, so fast `a`, 한/영, `a` gives `aㅁ`, and browsers don't flash their Alt menu. A dedicated 한/영 key also works.
- **Right Ctrl stays Ctrl.** Hanja conversion is on **F9**, in Hangul mode only. A dedicated 한자 key (on Korean 103/106-key keyboards) also works, in Hangul mode only.
- **No popup when switching.** Tapping Left Shift and pressing Ctrl+Space don't switch modes.
- **Typing like Windows:** the syllable you're typing shows inline in apps that support it, syllables are committed one at a time, and the layout is 두벌식.
- **One 한/영 state for all windows.** New windows keep your current mode, and each login starts in English.
- **Password fields type English.** Fcitx turns Korean off in fields that apps mark as password fields. Terminal password prompts, such as `sudo` in Konsole, aren't marked, so switch to English before typing a password there.
- **Caps Lock works normally in English mode.** In Hangul mode it's ignored: Hangul keeps typing, as on Windows, and Shift still gives ㅃ, ㅉ, ㄸ, ㄲ, ㅆ, ㅒ, ㅖ.
- **Works in native Wayland apps and in most X11 apps**, using Fcitx's recommended setup. A few X11 apps need a per-app setting; see [Only if a specific app needs it](#only-if-a-specific-app-needs-it).

## Before you start

- **Close System Settings and Fcitx's settings window.** A page left open can write its old values back if you click Apply.
- **Use Konsole as your normal user.** Never use a root shell (`sudo -i`, `su`). Of the four steps, only Step 1 uses `sudo`.
- **If this PC was set up before,** you can optionally back up Fcitx's settings first: `cp -a ~/.config/fcitx5 ~/fcitx5-backup`

## Step 1: Install

```bash
sudo dnf install -y fcitx5 fcitx5-hangul fcitx5-gtk fcitx5-qt fcitx5-configtool
```

Don't install these two:
- `kcm-fcitx5`: it couldn't be installed on Fedora 45 at the time of writing, and it's optional anyway (see [Notes](#notes)).
- `fcitx5-autostart`: it sets environment variables that Fcitx advises against on KDE, and starts Fcitx a second way alongside KDE.

## Step 2: Configure

Paste the whole block, from the opening `(` to the closing `)`.

- It must end with **Done**.
- If it prints **STOPPED**, nothing was changed. Fix what it says and paste it again.
- If it prints **WARNING** lines, fix them before rebooting.
- If it stops with an error and no **Done**, a command failed partway. Fix the error and paste the whole block again; running it again is safe.

```bash
(
set -e

# --- Safety checks: stop before changing anything if something is wrong ---
if [ "$(id -u)" -eq 0 ]; then echo "STOPPED: run this as your normal user, not with sudo."; exit 1; fi
if ! rpm -q fcitx5 fcitx5-hangul fcitx5-gtk fcitx5-qt fcitx5-configtool >/dev/null; then
  echo "STOPPED: install the packages first (step 1)."; exit 1; fi
if ! command -v kwriteconfig6 >/dev/null; then echo "STOPPED: kwriteconfig6 not found (KDE Plasma 6 required)."; exit 1; fi

# --- Stop your own Fcitx and wait until it has saved and exited ---
pkill -u "$(id -u)" -x fcitx5 2>/dev/null || true
for i in $(seq 50); do pgrep -u "$(id -u)" -x fcitx5 >/dev/null || break; sleep 0.1; done
if pgrep -u "$(id -u)" -x fcitx5 >/dev/null; then echo "STOPPED: Fcitx did not exit; nothing was changed."; exit 1; fi

# Let KDE start Fcitx 5 (instead of Fedora's on-screen keyboard)
kwriteconfig6 --file kwinrc --group Wayland --key InputMethod /usr/share/applications/org.fcitx.Fcitx5.desktop

# Keyboard: US layout; Right Alt becomes 한/영; Right Ctrl stays Ctrl
kwriteconfig6 --file kxkbrc --group Layout --key Use true
kwriteconfig6 --file kxkbrc --group Layout --key LayoutList us
kwriteconfig6 --file kxkbrc --group Layout --key VariantList ""
kwriteconfig6 --file kxkbrc --group Layout --key ResetOldOptions true
kwriteconfig6 --file kxkbrc --group Layout --key Options korean:ralt_hangul

# Korean input for X11 apps (Fcitx's recommended setup)
mkdir -p ~/.config/environment.d ~/.config/gtk-3.0 ~/.config/gtk-4.0
cat > ~/.config/environment.d/fcitx5.conf <<'EOF'
XMODIFIERS=@im=fcitx
QT_IM_MODULES="wayland;fcitx;ibus"
EOF
kwriteconfig6 --file ~/.config/gtk-3.0/settings.ini --group Settings --key gtk-im-module fcitx
kwriteconfig6 --file ~/.config/gtk-4.0/settings.ini --group Settings --key gtk-im-module fcitx
touch ~/.gtkrc-2.0
sed -i '/^gtk-im-module/d' ~/.gtkrc-2.0
echo 'gtk-im-module="fcitx"' >> ~/.gtkrc-2.0

# Fcitx input methods: English (US) + Hangul
mkdir -p ~/.config/fcitx5/conf
cat > ~/.config/fcitx5/profile <<'EOF'
[Groups/0]
Name=Default
Default Layout=us
DefaultIM=hangul

[Groups/0/Items/0]
Name=keyboard-us
Layout=

[Groups/0/Items/1]
Name=hangul
Layout=

[GroupOrder]
0=Default
EOF

# Fcitx behavior
cat > ~/.config/fcitx5/config <<'EOF'
[Hotkey]
AltTriggerKeys=
ActivateKeys=

[Hotkey/TriggerKeys]
0=Hangul

[Behavior]
ActiveByDefault=False
ShareInputState=All
ShowInputMethodInformation=False
PreeditEnabledByDefault=True
AllowInputMethodForPassword=False
EOF

# Hangul engine: 두벌식, syllable-by-syllable commit, Hanja keys (Hangul mode only)
cat > ~/.config/fcitx5/conf/hangul.conf <<'EOF'
Keyboard=Dubeolsik
WordCommit=False

[HanjaModeToggleKey]
0=Hangul_Hanja
1=F9
EOF

# --- Warn about leftovers from other guides that would override these settings ---
for f in ~/.bash_profile ~/.bashrc ~/.profile ~/.config/plasma-workspace/env/*.sh /etc/profile.d/*.sh /etc/environment; do
  if [ -f "$f" ] && grep -qsE '^[^#]*(GTK_IM_MODULE|QT_IM_MODULE=|SDL_IM_MODULE|XMODIFIERS)' "$f"; then
    echo "WARNING: $f sets input-method variables. Remove those lines before rebooting."
  fi
done
if [ -f /etc/xdg/autostart/org.fcitx.Fcitx5.desktop ]; then
  echo "WARNING: fcitx5-autostart is installed. Remove it: sudo dnf remove fcitx5-autostart"
fi

echo "Done. Close System Settings if it's open, then reboot: systemctl reboot"
)
```

## Step 3: Reboot

```bash
systemctl reboot
```

## Step 4: Verify

```bash
kreadconfig6 --file kwinrc --group Wayland --key InputMethod
kreadconfig6 --file kxkbrc --group Layout --key Options
fcitx5-remote --check && echo "Fcitx is running"
fcitx5-remote -n
env | grep -E 'XMODIFIERS|IM_MODULE'
```

Expected output:

```text
/usr/share/applications/org.fcitx.Fcitx5.desktop
korean:ralt_hangul
1
Fcitx is running
keyboard-us
QT_IM_MODULES=wayland;fcitx;ibus
XMODIFIERS=@im=fcitx
```

- **The `1` line:** the current mode. `1` is English, `2` is Hangul, and `0` means no text box has focus. `keyboard-us` changes to `hangul` while you're in Hangul mode.
- **No `GTK_IM_MODULE` or `QT_IM_MODULE` lines** should appear. The `QT_IM_MODULES` and `XMODIFIERS` lines can appear in either order.
- **If the `fcitx5-remote --check` line prints nothing,** Fcitx isn't running. See [Troubleshooting](#troubleshooting).

Typing tests:

- [ ] `a`, 한/영, `a` gives `aㅁ`, even when typed fast, with no Alt-menu flash in the browser
- [ ] In Hangul mode, `한` then **F9** shows Hanja choices
- [ ] In English mode, F9 does nothing to the input method
- [ ] Right Ctrl + C copies selected text in a text editor such as Kate (in Konsole, Ctrl+C stops a command instead)
- [ ] X11 test: `QT_QPA_PLATFORM=xcb konsole` opens Konsole as an X11 app, and Korean types there too

---

## What each setting does

| Setting | Effect |
|---|---|
| `kwinrc` `InputMethod` = Fcitx 5 | KDE starts Fcitx natively, replacing Fedora's default on-screen keyboard |
| `kxkbrc` US layout + `korean:ralt_hangul` | Right Alt sends a real 한/영 key: it switches on key-down and never triggers the Alt menu. Right Ctrl is untouched. |
| `XMODIFIERS=@im=fcitx` | Older X11 apps that use XIM, such as Java apps and Wine programs, can use Fcitx |
| `QT_IM_MODULES="wayland;fcitx;ibus"` | Qt 6 apps running under X11 fall back to Fcitx. Native Wayland Qt apps still try `wayland` first. |
| `gtk-im-module=fcitx` (GTK 2/3/4 settings files) | GTK apps running under X11 use Fcitx. Native Wayland GTK apps keep using Wayland's own input method, as Fcitx documents. |
| `TriggerKeys` = `Hangul` | Only 한/영 switches modes. Ctrl+Space stays free (code autocomplete, etc.). |
| `AltTriggerKeys` empty | Tapping Left Shift doesn't switch modes |
| `ActivateKeys` empty | In English mode, the 한자 key doesn't switch you to Hangul |
| `ActiveByDefault=False` | Each login starts in English |
| `ShareInputState=All` | One 한/영 state for all windows; new windows keep your current mode |
| `ShowInputMethodInformation=False` | No popup when switching |
| `PreeditEnabledByDefault=True` | The syllable you're typing shows inline, in apps that support it |
| `AllowInputMethodForPassword=False` | No Korean in fields that apps mark as password fields |
| `Keyboard=Dubeolsik` | 두벌식 layout |
| `WordCommit=False` | Syllables are committed one at a time |
| `HanjaModeToggleKey` = 한자 key, F9 | Hanja conversion keys, in Hangul mode only |

> Numbered lists in Fcitx's config files (`0=`, `1=`, …) must start at `0` with no gaps. Fcitx reads from `0=` upward and stops at the first missing number, so entries after a gap are ignored and a list without `0=` is empty.

---

## Only if a specific app needs it

**A Chrome or Electron app ignores Korean** (usually an older app):

1. Right-click it in the app menu → **Edit Application**.
2. In **Command-line arguments**, add `--enable-wayland-ime --wayland-text-input-version=3`.
3. If it still misbehaves, change it to `--wayland-text-input-version=1`. For old Electron apps, also add `--ozone-platform=wayland`.

**An X11 app ignores Korean:** in that app's **Edit Application → Environment variables**, set only for that app:
- GTK apps: `GTK_IM_MODULE=fcitx`
- Qt 5 apps: `QT_IM_MODULE=fcitx`
- SDL2 games: `SDL_IM_MODULE=fcitx`

Use `xim` in place of `fcitx` only as a last resort.

**Flatpak apps:** use `flatpak override --user --env=NAME=value <app-id>`. KDE's documented fix for Steam:

```bash
flatpak override --user --env=XMODIFIERS=@im=fcitx --env=GTK_IM_MODULE=xim \
  --env=QT_IM_MODULE=xim --env=QT_IM_MODULES="wayland;fcitx;ibus" com.valvesoftware.Steam
```

## Optional changes

After the initial setup, Fcitx is running and overwrites its own config files, so make Fcitx changes in the GUI: **tray icon → Configure**.

- **Make the printed 한자 on Right Ctrl work, as Windows does.** Right Ctrl then stops working as Ctrl. Go to System Settings → Keyboard → Key Bindings. With **Configure keyboard options** on (the setup already turned it on), open Korean Hangul/Hanja keys and enable **Make right Ctrl a Hanja key**, keeping **Make right Alt a Hangul key** enabled.
- **F9-only Hanja:** in Fcitx's settings (tray icon → Configure), open the settings of the Hangul input method and remove the 한자 entry from **Hanja Mode Toggle Key**.
- **Have each app remember its own 한/영 state:** go to tray icon → Configure → Global Options → **Share Input State** and choose **Program**.

---

## Keep it working

- **Change Fcitx settings through tray icon → Configure** (or run `fcitx5-configtool`), not by editing files while Fcitx runs.
- **Never click "Restart" in the Fcitx tray menu.** Log out and back in instead.
- **Don't install `fcitx5-autostart`**, and don't set `GTK_IM_MODULE`, `QT_IM_MODULE` or `SDL_IM_MODULE` globally.
- **Don't run `ibus start` or `ibus restart`.** On KDE Wayland they switch KDE's input-method setting to IBus.
- **Use Left Alt for Alt shortcuts,** since Right Alt is 한/영.
- **After a major Fedora upgrade,** run Step 4 again.

## Known differences from Windows 10

These remain with this setup. The F9 one comes from using F9 as the Hanja key; the rest come from how fcitx5-hangul works or from app and platform limitations.

- **F9:** in Hangul mode, Fcitx takes F9, so apps only receive it in English mode. Windows' Korean IME doesn't use F9 for Hanja; it uses the 한자 key or Right Ctrl.
- **Clicking elsewhere mid-syllable:** Windows normally keeps the half-typed syllable. Here it depends on the app: some apps commit it, others lose it.
- **Hanja conversion of text you already typed or selected** works only in apps that share their text with the input method. Terminals and older X11 apps usually don't.
- **Older X11 apps that use XIM** may show the syllable you're typing in a small separate box instead of inline.
- **Chrome popups** (bookmark-name box, tab-group-name box) may not take Korean on Wayland. This was reported as a Chromium bug in October 2026 on a different Wayland compositor, so it may not affect KDE.
- **Mode indicator:** Fcitx shows its tray icon instead of Windows' 가/A.

## Troubleshooting

- **`fcitx5-remote --check` prints nothing:**
  1. Make sure System Settings → Keyboard → Virtual Keyboard shows **Fcitx 5**.
  2. Reboot.
  3. If it still prints nothing, run `fcitx5-diagnose > ~/fcitx5-diagnose.txt` and read the report.
- **`env` shows `GTK_IM_MODULE` or `QT_IM_MODULE`:** remove the leftovers that set them, then reboot. Common places: `~/.bash_profile`, `~/.config/plasma-workspace/env/`, and the `fcitx5-autostart` package.
- **한/영 is flaky, or the Alt menu still flashes:**
  1. Install `wev` (`sudo dnf install wev`) and run `wev -f wl_keyboard:key`.
  2. Press 한/영. It must show `sym: Hangul`, not `sym: Alt_R`.
- **Korean works in some apps but not others:** see [Only if a specific app needs it](#only-if-a-specific-app-needs-it).

## Undo

Run as your normal user:

```bash
pkill -u "$(id -u)" -x fcitx5
kwriteconfig6 --file kwinrc --group Wayland --key InputMethod --delete
kwriteconfig6 --file kxkbrc --group Layout --key Options --delete
kwriteconfig6 --file kxkbrc --group Layout --key ResetOldOptions --delete
kwriteconfig6 --file ~/.config/gtk-3.0/settings.ini --group Settings --key gtk-im-module --delete
kwriteconfig6 --file ~/.config/gtk-4.0/settings.ini --group Settings --key gtk-im-module --delete
sed -i '/^gtk-im-module/d' ~/.gtkrc-2.0
rm -f ~/.config/environment.d/fcitx5.conf
rm -rf ~/.config/fcitx5
systemctl reboot
```

After the reboot, KDE goes back to Fedora's default (the on-screen keyboard). The keyboard layout stays set to US. To also remove the packages:

```bash
sudo dnf remove fcitx5 fcitx5-hangul fcitx5-gtk fcitx5-qt fcitx5-configtool
```

## Notes

- **`kcm-fcitx5`** adds a Fcitx page to System Settings and is optional. On Fedora 45 it couldn't be installed at the time of writing: it was built against `libPlasma.so.7`, but Plasma 6.8 ships `libPlasma.so.8`. To check whether it's been fixed:

  ```bash
  dnf repoquery --requires kcm-fcitx5 | grep -i plasma
  ```

  Once that shows `libPlasma.so.8`, `sudo dnf install kcm-fcitx5` will work.

- **Sources:**
  - [Fcitx: Using Fcitx 5 on Wayland](https://fcitx-im.org/wiki/Using_Fcitx_5_on_Wayland)
  - [Fcitx: Setup Fcitx 5](https://fcitx-im.org/wiki/Setup_Fcitx_5)
  - [KDE: Type in Chinese, Japanese, Korean, or Vietnamese](https://linux.kde.org/docs/cjkv-input/)
  - [Arch Wiki: Fcitx5](https://wiki.archlinux.org/title/Fcitx5)
  - [Fedora: Input methods are not handled by KDE initial setup](https://discussion.fedoraproject.org/t/input-methods-are-not-handled-by-kde-initial-setup/183548)
  - [Microsoft: Korean IME](https://learn.microsoft.com/en-us/globalization/input/korean-ime)
