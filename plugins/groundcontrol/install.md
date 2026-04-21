# Groundcontrol Plugin — Install Pin

Groundcontrol is a TypeScript OpenClaw plugin (task board, initiatives, OKRs) that
loads into the gateway at startup. It lives in its **own repository** — this file
only pins the version and documents the install procedure for the Admin-Agent.

## Source

- **Repository:** `<GROUNDCONTROL_REPO_URL>` (TODO: fill in public Git URL)
- **Path within repo:** `openclaw-plugin/`
- **Pinned version / commit:** `main` (change to a specific commit SHA once the
  plugin is stable enough to pin)

> **Note to Admin-Agent:** If this pin is updated in a commit, treat it as a plugin
> upgrade during `fleet update` — re-run the Install Steps below and restart the
> gateway.

## Install Steps

Run on the instance (SSH from Admin-Agent):

1. **Clone or pull the plugin repo:**
   ```bash
   mkdir -p ~/repositories
   if [ ! -d ~/repositories/groundcontrol ]; then
     git clone <GROUNDCONTROL_REPO_URL> ~/repositories/groundcontrol
   else
     git -C ~/repositories/groundcontrol fetch --tags
   fi
   git -C ~/repositories/groundcontrol checkout <PINNED_REF>
   ```

2. **Install dependencies and build:**
   ```bash
   cd ~/repositories/groundcontrol/openclaw-plugin
   npm install
   npm run build
   ```

3. **Register plugin with the gateway.** Symlink (or copy) the built plugin into
   `~/.openclaw/plugins/`:
   ```bash
   mkdir -p ~/.openclaw/plugins
   ln -sfn ~/repositories/groundcontrol/openclaw-plugin ~/.openclaw/plugins/groundcontrol
   ```

4. **Install the GC cron worker** (if not already present). The plugin ships with
   `scripts/gc-worker.sh`. Add to the user crontab:
   ```cron
   */5 * * * * $HOME/repositories/groundcontrol/openclaw-plugin/scripts/gc-worker.sh >> $HOME/.openclaw/gc-worker.log 2>&1
   ```
   (Exact schedule may be overridden by `cron/default-cron.md` — prefer that if it
   defines a GC entry.)

5. **Restart the gateway** so the plugin loads. Use the OpenClaw-native CLI:
   ```bash
   openclaw gateway restart
   ```

## Verification

- `openclaw plugin list` shows `groundcontrol` as loaded.
- `openclaw plugin show groundcontrol` lists the expected tool set (initiatives,
  tasks, docs, OKRs — roughly 20+ tools).
- Tail `~/.openclaw/gc-worker.log` for worker heartbeat.

## Uninstall

1. `rm ~/.openclaw/plugins/groundcontrol`
2. Remove the `gc-worker.sh` entry from crontab.
3. Restart the gateway.
4. Optional: `rm -rf ~/repositories/groundcontrol` if nothing else depends on it.

## Related Documents

- `docs/install.md` — calls this file during fresh install.
- `docs/update.md` — calls this file when the pinned ref changes.
- `cron/default-cron.md` — canonical GC worker schedule (if defined).
