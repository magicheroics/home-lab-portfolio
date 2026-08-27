# Xubuntu: Missing Window Controls (No Titlebar / Close-Minimize-Maximize Buttons)

**System:** Xubuntu 24.04 LTS — HP Desktop (Pentium Dual-Core E5500, 4GB RAM)
**Role:** Home server (Pi-hole DNS, SSH/SFTP)
**Date encountered:** August 2026

## Symptom

After booting into the desktop, the Xfce panel/taskbar loaded normally (clock, notification icons, username all visible), but application windows had no titlebar decorations — no close, minimize, or maximize buttons. Applications themselves (e.g. terminal) still launched and functioned correctly; only the window chrome was missing. Screen also showed diagonal streaking/tearing artifacts on first inspection, which turned out to be a red herring — the underlying window manager failure was the real issue.

## Diagnosis

Checked the session error log for clues:

```bash
cat ~/.xsession-errors | tail -50
```

The relevant line appeared right at session startup:

```
discover_other_daemon: 1
xfsettingsd: No window manager registered on screen 0.
xfce4-panel: No window manager registered on screen 0. To start the panel without this check, run with --disable-wm-check.
No window manager registered on screen 0. To start the xfdesktop without this check, run with --disable-wm-check.
```

This confirmed that **xfwm4 (the Xfce window manager) failed to launch during session startup**. Every other component of the desktop session (panel, settings daemon, desktop manager) started as expected and logged the same complaint — none of them could register with a window manager because xfwm4 was never running. The rest of the log contained unrelated noise (Firefox snap `ibus` sandbox warnings, a duplicate polkit authentication agent) that did not contribute to the issue.

Without xfwm4, windows still open and run (they're just X11 client applications), but they lack decorations, since drawing titlebars and window controls is the window manager's job.

## Fix

### Immediate fix (same session, no reboot needed)

Restart the window manager manually from a terminal:

```bash
xfwm4 --replace &
```

This re-registers xfwm4 on the display and immediately restores titlebars and window controls on all open and future windows.

*(If run remotely over SSH rather than from a local terminal, prefix with `DISPLAY=:0` so it targets the active graphical session: `DISPLAY=:0 xfwm4 --replace &`)*

### If xfwm4 is missing entirely

```bash
sudo apt install --reinstall xfwm4
```

then re-run the `--replace` command above.

## Preventing recurrence

1. **Clear a potentially corrupt session cache**, a common cause of xfwm4 silently failing to register on login:
   ```bash
   mv ~/.cache/sessions ~/.cache/sessions.bak
   ```
   Log out and back in afterward.

2. **Confirm the session is using the correct Xfce session profile:**
   ```bash
   xfconf-query -c xfce4-session -p /general/SessionName
   ```
   Expected output: `xubuntu` or `Default`.

3. **Check Application Autostart settings:**
   Settings → Session and Startup → Application Autostart — confirm xfwm4 is not disabled there.

## Outcome

Running `xfwm4 --replace &` restored window controls immediately. Session cache was cleared and autostart/session settings verified as above to prevent the window manager from failing to launch on future logins.

## Key takeaway

Missing window decorations with an otherwise-functional desktop panel is a strong signal to check whether the window manager process itself is running, rather than assuming a graphics driver or display fault. `~/.xsession-errors` is the first place to look — it logs exactly which desktop component failed and why, right from session startup.
