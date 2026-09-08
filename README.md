# boxhop
BoxHop is a Linux remote-asset console with ssh, ftp and rdp

# BoxHop 0.5.1

BoxHop is a Linux remote-asset console with an UPPY-style dark navy/blue interface.


## 0.5.1 — visual polish and release optimisation pass

- New BoxHop application icon, installed at standard hicolor sizes from 16px through 512px.
- The native development runner exposes the bundled icon through `XDG_DATA_DIRS`, so the correct window icon is visible without a system install.
- Refined dark UI: icon-labelled navigation/actions, clearer status pill, improved asset rows with host/IP subtitles, denser file rows with file/folder icons, stronger hierarchy and consistent spacing/borders.
- FTP downloads now stream directly to disk rather than buffering the entire remote file in RAM.
- FTP downloads use a temporary `.boxhop-part` file and atomically rename it into place only after the transfer completes.
- PASV fallback uses the FTP control connection peer address plus the advertised data port, improving NAT compatibility and avoiding third-host FTP bounce behaviour.
- Asset configuration writes are now transactional: JSON is written/fsynced to a temporary mode-0600 file and atomically renamed into place.
- GTK 4.10+ confirmation dialogs now use `AlertDialog` instead of the deprecated `Dialog` API.
- Release builds now enable size-oriented optimisation, full LTO, single codegen unit, symbol stripping and abort-on-panic. The panic hook still records crash context before abort.
- Added `scripts/size-report.sh` and `RELEASE_CHECKLIST.md`.

### Language decision

BoxHop remains Rust. Rewriting it in C/C++ would not materially improve network/GUI latency, would increase memory-safety risk, and would cost substantial development time. Rust is also a good fit for replacing remaining helper processes with in-process protocol/crypto implementations later. The main size cost in the final Flatpak is the GUI/runtime and bundled protocol helpers, not the Rust executable itself.

### Size target

The target for the final x86_64 BoxHop application payload is roughly **15–25 MiB compressed** and **25–45 MiB installed**, excluding the shared Flatpak platform. This remains an estimate until the final OpenSSH/FreeRDP/credential-vault packaging is complete. Use `scripts/size-report.sh` for actual measurements.


## 0.4.4

- Fixed stale Asset form values being ignored by live actions. FTP/SFTP refresh, folder navigation, upload/download, SSH terminal, RDP, Wake, Check and Shutdown now use the values currently visible in the selected Asset editor.
- This specifically fixes the case where Test FTP succeeds from the editor but the Files tab says `FTP username is not configured` because Save asset had not yet been pressed.
- `Save asset` still controls persistence to disk and encrypted credential storage; unsaved edits are used for the current session but are not silently persisted.
- Existing 0.4.3 assets, vault data and logs remain compatible.

## 0.4.3

- Adds persistent **running logs** and **crash logs** under `~/.local/state/boxhop/logs/` with UTC ISO-8601 timestamps, PID and severity.
- Adds a **Logs** tab inside BoxHop with live views of `boxhop.log` and `crash.log`; the page refreshes automatically once per second while visible.
- Runtime logging records application actions, connection results and errors without logging saved passwords, vault passwords or the vault key.
- Rust panics are captured to `crash.log` with a backtrace plus the recent runtime-log tail for context.
- Every running BoxHop process creates a per-PID session marker. If a process is killed, aborts or otherwise exits without normal cleanup, the stale marker is recorded in `crash.log` on the next launch.
- `boxhop.log` rotates at 5 MiB and `crash.log` rotates at 2 MiB to avoid unbounded growth; the previous file is retained as `.1`.
- Removes the unnecessary mutable TCP stream warning.
- Existing 0.4.2 assets and encrypted credential vault remain compatible.

### Log files

```text
~/.local/state/boxhop/logs/
├── boxhop.log
├── boxhop.log.1
├── crash.log
├── crash.log.1
└── session-<pid>.running
```

The session marker exists only while that BoxHop process is active. It is deleted during a clean shutdown. A stale marker is evidence that the previous process did not reach normal shutdown.

## 0.4.2

- SSH/FTP/RDP test buttons and the top-level Check action now run in background workers so network/authentication tests cannot block GTK.
- SSH tests have a 12-second hard timeout and suppress the harmless first-use `Permanently added ... to known hosts` warning from failure text.
- A failed public-key/password authentication now returns control to the UI instead of triggering a desktop “Not Responding” prompt.

- Fixes the FTP Files-tab freeze by moving FTP/SFTP refresh, remote navigation, upload and download work off the GTK UI thread.
- The BoxHop window remains responsive while a slow FTP server is being queried.
- Remote lists now show `Refreshing…`, `Opening…`, `(empty directory)` or `Remote listing failed` instead of leaving a blank pane with no explanation.
- FTP Refresh reports the server-reported working directory and the number of visible entries.
- FTP login-directory handling no longer silently forces `/` when the path is blank or `.`.
- MLSD parsing is more tolerant of fact-name case, and an empty/unparseable MLSD result is verified with LIST before BoxHop decides the directory is empty.
- FTP still runs in-process in Rust: no host `ftp`, `curl` or extra FTP client dependency.
- Existing encrypted credentials, assets and 0.4.0 configuration remain compatible.

## FTP diagnosis

A green **Test FTP** result proves that the control connection and login work. It does not prove that the FTP account can see files in every directory.

In the Files tab:

1. Select **FTP**.
2. Use `.` for the FTP account's login directory, or enter a specific FTP-visible path.
3. Press **Refresh**.
4. BoxHop reports the actual server working directory and item count.

If BoxHop reports `0 items`, the directory is genuinely empty from that FTP account's point of view, or the account is chrooted/restricted from seeing anything there. This is different from a failed listing, which is shown as an error.

## Credential vault layout

The native development build stores BoxHop configuration below the standard per-user config directory, normally:

```text
~/.config/boxhop/
├── assets.json
└── credentials/
    ├── vault-key.gpg
    ├── asset-<uuid>.gpg
    └── .gnupg/
```

`vault-key.gpg` contains only the random BoxHop vault key encrypted by the user's master password. Each `asset-<uuid>.gpg` contains that asset's SSH, FTP and RDP password fields encrypted by the random vault key.

## Current limitations

- Plain FTP is implemented. FTPS/TLS is not enabled yet.
- SFTP password injection into the dual-pane browser is still pending; SFTP uses OpenSSH key/agent authentication.
- Interactive SSH password injection is still pending; SSH keys/agent or normal terminal prompting remain available.
- Native development currently discovers OpenSSH, FreeRDP and GnuPG from `PATH`. These are explicit development dependencies and are not allowed to remain hidden host dependencies in the final Flatpak.
- The current Flatpak manifest is still a development manifest and its Rust sources are not yet fully vendored/pinned for an offline final build. The manifest now targets GNOME runtime 50 rather than the EOL GNOME 48 branch.

## Arch native development

```bash
sudo pacman -S --needed \
  base-devel \
  rust \
  gtk4 \
  vte4 \
  openssh \
  freerdp \
  gnupg
```

Then:

```bash
./run-from-source.sh
```

Do not run BoxHop through `sudo`.

## Final Flatpak dependency policy

The final distributable must not assume host copies of OpenSSH, FreeRDP, GnuPG, curl, FTP tools or other command-line utilities. Anything BoxHop executes at runtime must be bundled into the Flatpak or replaced by an in-process implementation. The FTP backend already meets that requirement because it is implemented inside BoxHop itself. See `DEPENDENCIES.md`.
