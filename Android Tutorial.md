# Nighty on Android: running nighty-linux-headless in Termux

*Tested on a Samsung Galaxy S23 · Termux · Debian 13 (proot-distro) · Nighty 2.6.1 · October 2026. Built on [glowxx/nighty-linux-headless](https://github.com/glowxx/nighty-linux-headless).*

## Read this first

- This is an unofficial community guide, not affiliated with Nighty or the project’s author. Nighty is third-party software, and automating a Discord user account may break Discord’s Terms of Service. Use it at your own risk.
- Your Nighty license key, Discord account token, bot token and `.env` file are secrets. Never paste them into issues, comments or screenshots.
- It works, but with trade-offs: each boot takes about 4–5 minutes, the battery drains quickly, and the bot can lag. See *Limitations* at the end.

## Why this setup

The project is built for Linux servers: Docker, or a bare-metal install with Wine (and Box64 on ARM). Plain Termux can’t run either, because it uses Android’s own C library instead of glibc and has no systemd, no sudo/root and no Docker.

The workaround is to run a real Debian inside Termux with proot-distro, then use the project’s bare-metal installer inside it:

`Android → Termux → Debian (proot) → Box64 → Wine (x86-64) → repacked Nighty → the project’s web panel`

## What you need

- An ARM64 Android phone with several GB of free storage, and a charger. (Mine had 1.3–2 GB of RAM free while Nighty ran.)
- Termux from F-Droid or GitHub, not the Play Store.
- Your own `Nighty.exe` (the x86-64 Windows build) and license key.
- A Discord application with a bot, from the Discord Developer Portal.
- Wi-Fi, and a second device (PC or tablet) for the web panel. That lets Termux stay in the foreground.

Terminal tips: paste commands one at a time. A prompt of `~ $` means plain Termux; `root@localhost:~#` means you’re inside Debian.

## Step 1: Termux and storage access

In plain Termux:

```
pkg install -y proot-distro
termux-setup-storage
```

The `-y` skips the confirmation prompt (if the install aborts at the prompt, a later Y goes to the shell and gives ‘No command Y found’). `termux-setup-storage` asks for permission to read your files; tap Allow. It lets Debian read your Download folder as `/sdcard/Download`.

## Step 2: Install and enter Debian

```
proot-distro install debian
proot-distro login debian
```

Your prompt changes to `root@localhost:~#`. Don’t run `proot-distro login` again from inside Debian (it refuses with ‘attempted to run proot-distro in a proot session’). Type `exit` to go back to Termux.

## Step 3: Install packages inside Debian

```
apt update && apt install -y git curl sudo nano python3 xz-utils libdbus-1-3 libfontconfig1 libfreetype6 libasound2t64 libpulse0
hash -r
which python3
```

`which python3` must print `/usr/bin/python3`. This matters: Debian in proot has no Python by default, and Termux’s tools sit at the end of the PATH. Without Debian’s own Python, the installer’s library check can pick up Termux’s Python (its path starts with `/data/data/com.termux/`), which can’t load Debian’s libraries. The installer then keeps reporting that Wine/X11 libraries are missing even though they’re installed.

The last five packages remove Box64 errors such as `Error initializing native libdbus-1.so.3` and `Error initializing native libfontconfig.so.1`.

## Step 4: Get the project and your Nighty.exe

```
git clone https://github.com/glowxx/nighty-linux-headless.git
cd nighty-linux-headless
cp '/sdcard/Download/Nighty.exe' .
ls -lh Nighty.exe
```

Adjust the path to wherever your file is. Quote paths that contain spaces or parentheses, for example `'/sdcard/Download/Nighty (1)/Nighty.exe'`, and keep the space and the dot at the end of the `cp` line. `ls` should show roughly 100 MB. If `cp` says ‘No such file’, run `termux-setup-storage` again in plain Termux.

## Step 5: Run the installer

```
bash scripts/install.sh
```

What it does: installs `uv`, builds Box64, downloads a static x86-64 Wine, creates `.env`, and repacks `Nighty.exe` into a headless `Nighty_stub.exe` (using Python 3.8 through uv).

What to expect:

- When I ran it, the Box64 APT repository key returned a 404, so it fell back to compiling Box64 from source. That step prints nothing for a long time on a phone (I saw 10+ minutes). To check that it’s alive, open a second Termux session (swipe in from the left edge, tap New session) and run `ps -ef | grep -E '[c]c1|[c]make|[m]ake' | head -5`. Lines mentioning cc1, make or cmake mean it’s compiling.
- Keep the phone plugged in with the screen on, and tap Acquire wakelock in Termux’s notification.
- It’s finished when you see `Setup complete`. Rerunning is safe, because it skips what’s already done.
- Yellow warnings about `iproute2` are expected. Step 11 deals with them.

## Step 6: Configure `.env`

```
nano .env
```

Set `WEBUI_USERNAME` and `WEBUI_PASSWORD`. These are only the login for the project’s web panel (port 8088). They are not your Discord or Nighty credentials. The panel listens on your whole network, so choose a real password (letters and numbers are safest).

In nano on a phone: tap CTRL in the extra-keys row, then F to search. Save with CTRL+X, then Y, then Enter. ALT then U undoes a mistake.

Then apply the tuning that made the difference on this hardware:

```
sed -i 's/^BOOT_TIMEOUT=.*/BOOT_TIMEOUT=3600/' .env
sed -i '/^NIGHTY_BOX64_PROFILE=/d;/^WEBUI_BOOT_TIMEOUT=/d' .env
echo '' >> .env
echo 'NIGHTY_BOX64_PROFILE=balanced' >> .env
echo 'WEBUI_BOOT_TIMEOUT=3600' >> .env
grep -n -E 'BOOT_TIMEOUT|PROFILE' .env
```

Why:

- The project’s watchdog restarts the backend if Nighty’s control port (8765) doesn’t answer within 300 seconds. A phone is slower, and a restart also cleans up Nighty’s extracted files, so a slow boot has to start over. One hour is generous.
- My `.env` contained `BOOT_TIMEOUT=300`, while the README names `WEBUI_BOOT_TIMEOUT`, so I set both.
- Box64 profile `balanced` is much faster than the `safe` profile my run defaulted to, and the README recommends it for ARM.

## Step 7: Create a launcher with the crash workarounds

```
cat > ~/start.sh <<'EOF'
#!/bin/bash
export GODEBUG=asyncpreemptoff=1 BOX64_AVX=0
cd ~/nighty-linux-headless && bash scripts/run.sh once
EOF
```

Once Nighty’s own code was starting, the backend crashed by itself about every four minutes. `diagnostics/backend.log` showed Go runtime errors (`panic: runtime error: invalid memory address or nil pointer dereference` and `fatal error: fault`). The README notes that Nighty’s bundled Go TLS client can crash under Box64. Setting `GODEBUG=asyncpreemptoff=1` (disables Go’s asynchronous preemption) and `BOX64_AVX=0` (hides AVX instructions from the emulated program) made the backend stay up. I applied both together, so I can’t say which one matters. They must be set in the same shell that starts `run.sh`, which is why they live in a launcher.

## Step 8: Keep Android from killing Termux

- Plug the phone in. Emulation drains the battery quickly.
- Pull down the Termux notification and tap Acquire wakelock.
- Settings → Apps → Termux → Battery → Unrestricted.
- Keep the screen on, and open the web panel from another device if you can.
- If Termux ever dies with `signal 9`, Android’s phantom-process limit may be the cause. Developer options (where available) has a ‘Disable child process restrictions’ switch, or you can disable the monitor through adb. I didn’t need it.

## Step 9: First start

```
bash ~/start.sh
```

The output ends with lines like `Box64 profile: balanced`, `Web UI bridge up` and `launching backend`. Then wait. The first boot, which also builds Nighty’s Wine prefix (about 400 MB), took several minutes.

- Find the phone’s IP in Settings → Connections → Wi-Fi, then open `http://PHONE-IP:8088/` from another device on the same Wi-Fi. `localhost` only works on the phone itself.
- The page loading doesn’t prove the backend is ready, because the bridge starts first.
- Healthy signs: no repeating `backend exited` lines, and Nighty’s CPU time climbing. In a second session run `ps -eo pid,time,comm --sort=-time | head -5` twice, 10 seconds apart, and compare the `Nighty_stub.exe` line.

If it loops or times out, see Troubleshooting below.

## Step 10: The setup wizard

On first open, the panel runs a four-step wizard:

1. **Activate:** your Nighty license key. Without it the bot signs in but no commands work.
2. **Sign in:** your Discord account token.
3. **Connect your bot:** the bot token (Developer Portal → your app → Bot → Reset Token). The privileged intents Presence, Server Members and Message Content must be enabled and saved.
4. **Authorize:** open Authorize on Discord, approve the bot on your account, then confirm.

The license and Authorize steps each restart the backend on purpose. The page estimates about 3 minutes; on a phone allow 4–5 each time, and don’t touch anything while it boots. If it lands back on the Authorize step after a restart, repeat the step.

## Step 11: Stop lyric and Spotify lookups from stalling the bot (optional, recommended)

Nighty’s presence features poll `lrclib.net` and `api.spotify.com`. On emulated hosts, calls that hang can stall the whole bot, so the installer blackholes those hosts in `/etc/hosts`. Its method also needs an unreachable route set with `ip route`, which proot can’t do (hence the iproute2 warning). A workable substitute is to point the hosts at the loopback address, so connections are refused instantly:

```
cp /etc/hosts /root/hosts.bak
grep -v -E 'lrclib|api.spotify' /root/hosts.bak > /tmp/h
echo '127.0.0.1 lrclib.net' >> /tmp/h
echo '127.0.0.1 api.lrclib.net' >> /tmp/h
echo '127.0.0.1 api.spotify.com' >> /tmp/h
cat /tmp/h > /etc/hosts
grep -n -E 'localhost|lrclib|spotify' /etc/hosts
```

This takes effect immediately. In `diagnostics/nighty.log` the Spotify errors should now read `Connection refused` and appear quickly instead of timing out. It only works if nothing on the phone listens on port 443. If the timeouts return after a restart, run it again.

## Step 12: Discord Developer Portal checks (this one cost me hours)

If slash commands appear in Discord’s `/` picker but every one answers ‘The application didn’t respond’ almost instantly, check this first:

**Developer Portal → your application → General Information → Interactions Endpoint URL must be empty.**

When a URL is set there, Discord sends interactions to that web address instead of through the bot’s connection, so Nighty never sees a command. Clear it, save, and retry. In my case this was the whole problem.

Other checks if commands are missing or not working:

- Make sure the license was activated (wizard step 1). Without it, Nighty aborts before registering its commands.
- Fully close and reopen Discord, which caches the command list.
- Settings → Authorized Apps should list your app. Per the project’s README, a user-installable app is authorized with an `integration_type=1` link with the `applications.commands` scope, and the wizard’s Authorize link should handle that. Also check Developer Portal → Installation → User Install.
- Confirm in the log that commands were registered:

```
grep -E 'Syncing|Commands synced' diagnostics/nighty.log | tail -n 3
```

## Daily use

- Start: `proot-distro login debian`, then `bash ~/start.sh`.
- Stop: CTRL+C, then `exit` to leave Debian. Exiting also kills any leftover background processes.
- If you see `Nighty is already running (runner PID …)` after a CTRL+C, a leftover supervisor is still alive. Run `exit`, log in again, or `kill` that PID. Changes to `.env` only apply after a fresh start.
- There is no autostart (no systemd in proot). After a reboot, or if Termux closes, start it again and allow about 5 minutes.
- Logs are in `diagnostics/`: `backend.log` (the backend process) and `nighty.log` (Nighty’s Discord and command engine). Timestamps are UTC.
- Nighty’s data (license, tokens) lives inside its Wine prefix at `/root/.local/share/nighty/prefix/drive_c/users/root/AppData/Roaming/Nighty Selfbot`.

## Troubleshooting

- **pkg install aborted at the prompt, then ‘No command Y found’:** use `pkg install -y`.
- **‘No such file or directory’ for `scripts/run.sh`:** wrong folder. Run `cd ~/nighty-linux-headless` first (the launcher does this for you).
- **Installer says Wine/X11 libraries are still missing:** see Step 3. Install Debian’s Python and confirm `which python3`.
- **Box64 prints `Error initializing native …`:** install the libraries from Step 3.
- **`backend boot timed out after 300s (stub never answered on :8765)`, repeating:** the watchdog is too impatient. Apply Step 6 and restart fully.
- **`Nighty is already running (runner PID …)` but nothing is happening:** see Daily use. Exit and log in again.
- **Backend exits every \~4 minutes with a Go `panic` or `fatal error: fault`:** use the launcher from Step 7.
- **The wizard goes back to a loading screen:** expected after the license and Authorize steps. Wait 4–5 minutes.
- **Bot online but no commands:** license missing (Step 10, item 1), Discord cache, or the checks in Step 12.
- **Commands fail almost instantly:** Interactions Endpoint URL (Step 12).
- **Spotify timeouts every 30 seconds in `nighty.log`:** Step 11.

### Backend exits after 3 seconds with a kernel32 error

The log says `wine: could not load kernel32.dll, status c0000135`. First test that Wine itself works, using a throwaway prefix:

```
WINEPREFIX=/root/wptest WINEDEBUG=err+all timeout 120 box64 /root/.local/share/nighty/wine/bin/wine64 cmd /c 'echo parent ok & cmd /c echo child ok' 2>&1 | grep -E 'parent ok|child ok|kernel32|import_dll|not found|No such'
```

If both ‘parent ok’ and ‘child ok’ print, Wine works and Nighty’s own prefix is probably damaged (mine likely was, after Termux crashed during first-time setup). Delete it and start again; it gets rebuilt. Only do this before finishing the wizard, because the prefix holds your license and tokens:

```
rm -rf /root/wptest /root/.local/share/nighty/prefix
```

### Reset Nighty’s own data but keep the installation

Move its data folder aside (a backup, not a delete), then run the wizard again:

```
mv '/root/.local/share/nighty/prefix/drive_c/users/root/AppData/Roaming/Nighty Selfbot' /root/nighty-data-backup
```

## Limitations

- At idle, Wine, proot and Nighty together used about 1.2 CPU cores of 8 in my test. Roughly half of that was proot’s own tracing overhead.
- `Can't keep up, websocket is N s behind` warnings appeared in `nighty.log` from time to time, up to about 30 seconds, so a slash command can occasionally time out. Retry it.
- Each restart takes 4–5 minutes. The phone runs slightly warmer than usual and loses battery fast, so keep it plugged in.
- I didn’t test Docker, autostart, or any phone other than the S23.
- For something dependable and quick, an x86-64 VPS or PC using the project’s Docker method is the better home. A phone works, but it is a hobbyist setup.

## Notes for the maintainers

Optional ideas that would have saved me time:

- The installer’s Python check could prefer `/usr/bin/python3`, or warn when the Python it finds lives under `/data/data/com.termux/`.
- Consider adding `libdbus-1-3` and `libfontconfig1` to the preflight library list, since Box64 complains without them.
- Where `ip route` isn’t possible (proot), a documented fallback to 127.0.0.1 for the blackholed hosts could replace the bare warning, with the caveat from issue #15 about local port 443 listeners.
- The README mentions `WEBUI_BOOT_TIMEOUT`, but my generated `.env` had `BOOT_TIMEOUT=300`. A single documented name would help.
- A short README note for proot users: after CTRL+C, a leftover supervisor can block the next start with `Nighty is already running`.

## Cheat sheet

All commands in order (details above):

```
# plain Termux
pkg install -y proot-distro
termux-setup-storage
proot-distro install debian
proot-distro login debian

# inside Debian
apt update && apt install -y git curl sudo nano python3 xz-utils libdbus-1-3 libfontconfig1 libfreetype6 libasound2t64 libpulse0
hash -r; which python3
git clone https://github.com/glowxx/nighty-linux-headless.git
cd nighty-linux-headless
cp '/sdcard/Download/Nighty.exe' .
bash scripts/install.sh
nano .env      # set WEBUI_USERNAME and WEBUI_PASSWORD, then apply Step 6 and Step 7
bash ~/start.sh
```

Then open `http://PHONE-IP:8088/`, finish the wizard, and check Step 12 if commands misbehave.

## Credits

[glowxx/nighty-linux-headless](https://github.com/glowxx/nighty-linux-headless), Box64, Wine, proot-distro and Termux.
