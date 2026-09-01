# OpenClaw 8.1 Gateway Won't Start After Upgrade: No Reinstall Needed, Two Commands Fix It

*[中文版 / Chinese version](./README.md)*

> A real rescue log — an existing multi-agent setup refused to start after upgrading from `2026.7.x` to `2026.8.1`.
> Environment: Windows 10/11 · Node.js 24.x · OpenClaw `2026.7.1-2` → `2026.8.1`
> Outcome: **zero reinstalls, zero agents reconfigured, zero data lost**

---

## TL;DR

If your gateway won't start after upgrading to 8.1, **your config is probably fine**. 8.1 requires you to run a state migration first.

```bash
openclaw doctor --fix     # migrate legacy state (this is the actual fix)
openclaw gateway run      # then start
```

→ **Full procedure (with backup and verification): [Section 4](#4-full-repair-procedure)** · One-shot checklist: [Appendix](#appendix-one-shot-triage-checklist)

If you're about to "wipe it and reconfigure every agent from scratch" — **stop and read this first**.

There is also a **silent side effect that raises no error**: 8.1 changed how agent workspace paths resolve, so some agents get quietly moved to an empty directory, which looks like "all my memory is gone" (nothing is actually lost). After you get it starting again, please check [Pitfall 6](#pitfall-6-agent-workspace-silently-redirected-the-sneakiest-one).

---

## 1. Symptoms

A setup that had been running fine for months. Upgrade to 8.1, and the gateway fails to start outright. The console tells you nothing useful, the dashboard won't open, every bot goes offline. `openclaw.json` is syntactically perfect and you didn't change a single character.

This is exactly where it's easy to go down the wrong road, because a very reasonable-sounding hypothesis shows up:

> "Maybe I have too many agents and the config got too complex, so it can't start?"

Then you start editing `openclaw.json` over and over, deleting agents, deleting channels — making it worse each time, until you're heading toward "nuke it and reconfigure everything."

**That hypothesis is wrong.** The number of agents has **nothing** to do with this failure. As you'll see, the real cause narrows down to one specific file.

Why call this out explicitly: "it broke because the config got too complex" is the most seductive wrong hypothesis in startup failures — it can't be directly disproven, and the "solution" it implies (start over) conveniently destroys the evidence. The price is losing all your config and session history for nothing.

---

## 2. The turning point: the real error was on disk the whole time

The first step should never be editing config — it should be **finding the actual error**. When OpenClaw fails to start it writes a structured crash log:

```
~/.openclaw/logs/stability/openclaw-stability-<timestamp>-<pid>-gateway.startup_failed.json
```

On Windows PowerShell, find the most recent ones:

```powershell
Get-ChildItem "$env:USERPROFILE\.openclaw\logs\stability" -Recurse |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 5 LastWriteTime, FullName
```

Open it, and the answer is right there:

```json
{
  "reason": "gateway.startup_failed",
  "error": {
    "message": "Legacy session store requires migration: C:\\Users\\<you>\\.openclaw\\agents\\main\\sessions\\sessions.json. Run \"openclaw doctor --fix\" against the same state/config before starting OpenClaw."
  }
}
```

**The program literally wrote the solution into the error message.**

> **Lesson 1**: When `openclaw.json` is syntactically valid but the gateway won't start, don't edit config on a hunch.
> Read `logs/stability/*.json` first, then run `openclaw doctor`. Editing config is the last step, not the first.

---

## 3. Root cause: 8.1 moved state from JSON to SQLite

8.1 is an **architectural upgrade of the state storage layer**. Session records, provider catalogs, workspace state and more were consolidated from scattered JSON files into SQLite:

```
~/.openclaw/state/openclaw.sqlite
```

To prevent data corruption, 8.1's policy is to **refuse to start when it detects legacy formats** and require you to run the migration explicitly. This is a safety mechanism, not a bug.

So:

- The symptom looks like "the config is broken"
- The reality is "the state wasn't migrated"
- Which is why **no amount of config editing will ever fix it**

### Why existing users are guaranteed to hit this

Worth spelling out, because it determines whether this applies to you:

- **Fresh installs** never hit it — they start out in SQLite format, no legacy artifacts.
- **Existing users** hit this wall **without exception** as long as old-format `sessions.json` / `catalog.json` files are still on disk.

In other words, it has nothing to do with how much you configured or how many agents you run — only with how early you installed. The earlier you installed and the longer you've used it, the more legacy files you have.

Also worth noting: the crash logs on this machine show a `gateway.startup_failed` from an **earlier major-version upgrade too**. That means "major upgrade blocks startup" is OpenClaw's **migration design pattern**, not a one-off 8.1 accident — next time you see this class of symptom, `doctor --fix` should still be your first move.

### Key detail: there are two gates, in series

After fixing the first error, the gateway gets to `ready` — and then **crashes on a second error**:

```
PreparedModelRuntimeOwnerNotPublishedError:
  prepared model catalog owner was not published for the requested config
  (C:\Users\<you>\.openclaw\agents\<agent>\agent)
```

That's the provider catalog also being unmigrated, located at:

```
~/.openclaw/agents/<agent>/agent/plugins/<provider>/catalog.json
```

One per model provider you've configured. **If all you see is "it won't start," it's easy to burn hours flailing at the first gate.**

The good news: the same `openclaw doctor --fix` clears both gates at once.

---

## 4. Full repair procedure

### Step 0: Back up (do not skip this)

**Back up before you touch anything.** The migration rewrites state; you need a way out if it goes wrong.

```powershell
$ts = Get-Date -Format "yyyyMMdd-HHmmss"
$bk = "D:\openclaw-backup-$ts"
New-Item -ItemType Directory -Force -Path $bk | Out-Null

# config file
Copy-Item "$env:USERPROFILE\.openclaw\openclaw.json" "$bk\openclaw.json" -Force

# all agent state (includes session history, can be large)
robocopy "$env:USERPROFILE\.openclaw\agents" "$bk\agents" /E /NFL /NDL /NJH /NJS /R:1 /W:1

# check backup size
"{0:N1} MB" -f ((Get-ChildItem $bk -Recurse -File | Measure-Object -Sum Length).Sum / 1MB)
```

Session history lives inside `agents/`, so **backing up only `openclaw.json` is not enough**. In practice this directory can reach hundreds of MB.

> ⚠️ **`~/.openclaw` contains plaintext credentials. Never push a backup of it to a public repo.** See section 6.

### Step 1: Diagnose first, see what will change

```bash
openclaw doctor
```

Without `--fix`, `doctor` is a **read-only preview** listing every pending migration. Read it before deciding.

### Step 2: Run the migration

```bash
openclaw doctor --fix
```

What this did in practice (single observed run; your environment may differ):

| Action | Notes |
| --- | --- |
| Migrate legacy session store | **the first gate blocking startup** |
| Migrate all provider catalogs into SQLite | **the second gate** |
| Normalize duplicate session keys | history preserved |
| Clear stale model routing pins | prevents sessions being pinned to old providers |
| Migrate workspace state and config audit log | — |

### Step 3: Start and verify

```bash
openclaw gateway run
```

Success means `[gateway] ready` **and no subsequent crash**:

```
[gateway] ready
[telegram] [<account>] starting provider (@YourBot)
[discord] Discord bot probe resolved @YourBot
```

### Step 4: Confirm it actually works end to end

`ready` only means the process came up. Verify for real:

```powershell
# dashboard (note: if you use a proxy, local requests must bypass it)
$env:HTTP_PROXY=""; $env:HTTPS_PROXY=""
(Invoke-WebRequest "http://127.0.0.1:18789/" -UseBasicParsing).StatusCode   # expect 200

openclaw status         # agents / channel accounts OK?
openclaw models list    # models resolvable?
```

You're truly back only when the log shows a real model call returning 200:

```
[model-fetch] response provider=<provider> model=<model> status=200
```

### Step 5: Confirm the legacy artifacts are actually gone (optional but recommended)

After migrating, just count them — harder evidence than reading logs:

```powershell
# expect: both zero
(Get-ChildItem "$env:USERPROFILE\.openclaw\agents" -Recurse -Filter "sessions.json" -EA SilentlyContinue).Count
(Get-ChildItem "$env:USERPROFILE\.openclaw\agents" -Recurse -Filter "catalog.json"  -EA SilentlyContinue).Count

# the SQLite state store should exist with a sane size
Get-ChildItem "$env:USERPROFILE\.openclaw\state" -File | Select-Object Name, Length
```

Both counts at zero + a multi-MB `openclaw.sqlite` = the migration really completed, rather than being temporarily dodged.

---

## 5. Companion pitfalls

### Pitfall 1: Plugin version drift (must fix)

The core is on 8.1 but plugins are still on 7.x:

```
Plugin version drift: N active official plugins not on gateway 2026.8.1
```

This is also the source of that pile of `requires capability consent` warnings. Update them one by one:

```bash
openclaw plugins update <plugin-name>
```

Afterwards, every version column in `openclaw plugins list` should read `2026.8.1` with no leftover old version numbers.

**Behind a restrictive network you'll need a proxy.** OpenClaw pulls plugins from the npm registry:

```powershell
$env:HTTP_PROXY  = "http://127.0.0.1:7890"
$env:HTTPS_PROXY = "http://127.0.0.1:7890"
$env:NO_PROXY    = "localhost,127.0.0.1,.local"    # the local gateway MUST bypass the proxy
openclaw plugins update <plugin-name>
```

Verify the proxy actually works before you start:

```powershell
(Invoke-WebRequest "https://registry.npmjs.org/-/ping" -Proxy "http://127.0.0.1:7890" -UseBasicParsing).StatusCode
```

Don't skip the `NO_PROXY` line — otherwise your later dashboard check against `127.0.0.1:18789` gets funneled through the proxy and fails for no reason.

### Pitfall 2: `GatewayLockError` — the running gateway holds the state lock

While updating plugins you may see:

```
Skipped plugin doctor state migrations because exclusive state ownership is unavailable:
GatewayLockError: failed to acquire gateway state ownership
```

Meaning: **the gateway is running and holding the state lock exclusively**, so the plugin's state migrations were skipped. The plugin itself updated fine, but its migration didn't finish.

> **Correct order: stop the gateway → update plugins → start again.**

### Pitfall 3: Some plugins require explicit capability consent

```bash
openclaw plugins enable <plugin-name> --accept-capabilities
openclaw plugins update <plugin-name> --accept-capabilities
```

Look at what it's actually asking for before you sign:

```bash
openclaw plugins inspect <plugin-name>
```

Some plugins have an empty capability list (`Capability mode: plain`) and this is pure formality. **Still, make `inspect` before `--accept-capabilities` a habit. Don't blind-sign.**

### Pitfall 4: Deleting an agent directory makes the gateway refuse to start

An agent you already removed from config may still have its directory on disk. Delete that directory and the gateway **refuses to start**:

```
OpenClaw startup migrations did not complete cleanly; refusing to report the gateway ready.
- Skipped missing registered agent database ...\<agent>\agent\openclaw-agent.sqlite
```

Because the SQLite registry still has that record. After deleting the directory you **must** follow up:

```bash
openclaw doctor --fix     # clean up stale registry entries
openclaw gateway run
```

> Memorize the order: **verify backup → delete directory → `doctor --fix` → start**.

### Pitfall 5 (Windows only): `gateway restart` errors out

If you try a hot restart during the repair, on Windows you hit:

```
Gateway restart failed: TypeError [ERR_UNKNOWN_SIGNAL]: Unknown signal: SIGUSR1
```

`openclaw gateway restart` implements hot reload via POSIX signals (`SIGUSR1`), and **Windows has no such signal**, so the command is unusable there.

Workaround:

```powershell
# stop the gateway process manually, then start again
openclaw gateway run
```

Good news: **most config changes don't need a restart at all.** After editing `openclaw.json`, read it back with `openclaw config get <path>`; if the new value is already visible, the gateway reads config live and no restart is needed.

### Pitfall 6: Agent workspace silently redirected (the sneakiest one)

**This one throws no error, never crashes, and `doctor` won't mention it** — but it will make you think all your data is gone.

Symptom: gateway fixed, everything nominal, but you open one agent and its workspace is empty except for a brand-new set of `BOOTSTRAP.md` / `AGENTS.md` / `SOUL.md` / `USER.md`. All the memory files, project directories and notes you painstakingly accumulated are nowhere to be found.

**Root cause**: 8.1 changed the semantics of `agents.defaults.workspace`.

| Version | Meaning of `agents.defaults.workspace` |
| --- | --- |
| Before 8.1 | Agents without an explicit `workspace` **use this directory directly** |
| 8.1 onward | This directory is treated as a **parent**; each agent lands in `<parent>\<agentId>` |

So if your config looks like this:

```json
{
  "agents": {
    "defaults": { "workspace": "C:\\Users\\<you>\\.openclaw\\workspace" },
    "entries": {
      "main": { "name": "Main_Manager" }
    }
  }
}
```

after the upgrade `main` no longer looks at `...\workspace\`, but at a freshly created `...\workspace\main\` — an empty directory that OpenClaw helpfully populates with a fresh set of bootstrap files, which looks exactly like "my memory was wiped."

**Key point**: agents that *do* have an explicit `workspace` are **completely unaffected**. So the classic presentation is "6 agents are fine, only 1 is broken," which is very easy to misdiagnose as that one agent being misconfigured.

**Diagnosis**:

```powershell
# is the old directory still there, and still complete?
Get-ChildItem "$env:USERPROFILE\.openclaw\workspace" -Force | Select-Object Name, LastWriteTime

# did the new empty directory appear just now (timestamp ≈ your upgrade)?
Get-ChildItem "$env:USERPROFILE\.openclaw\workspace\<agentId>" -Force
```

If the old directory is intact and the new one is timestamped at your upgrade — **nothing was lost, the agent is just standing in the wrong room.**

**Fix**: give the affected agent an explicit `workspace` pointing back at the original directory.

```bash
openclaw config set agents.entries.main.workspace "C:\Users\<you>\.openclaw\workspace"
openclaw config get agents.entries.main.workspace    # read back to confirm
```

Then restart the gateway (Windows: see pitfall 5) and confirm the agent can read its original `MEMORY.md` and friends.

> **Lesson**: after upgrading, don't only verify "is the gateway ready" — also spot-check **how each agent's workspace path resolves**. Path-semantics changes don't raise errors; they quietly move you into a different room.

---

## 6. Security: plaintext credentials (please read this one)

`doctor` reports the following, and it deserves its own section:

```
WARNING: openclaw.json contains plaintext secret-bearing config fields.
  Paths: gateway.auth.token, channels.<channel>.accounts.*.botToken, ...
```

**The problem**: bot tokens, API keys and the gateway token are stored **in plaintext** in `openclaw.json`. Anything that can read that file can take them — including:

- Your own agents (which typically have file-read tools)
- The backup you casually zipped up (the one from step 0)
- The config snippet you pasted into a chat window asking for help

The risk is higher for multi-agent users: a bot token is full impersonation of your bot, send and receive.

**What to do**: 8.1 supports migrating these into SecretRefs (config keeps only a reference; the real value lives elsewhere):

```bash
openclaw secrets configure
openclaw secrets audit --check
```

Three minimum requirements, worth doing right now:

1. `~/.openclaw` must **never** go into Git (or any public repo)
2. When sharing config for help, tokens and keys **must** be redacted
3. If you suspect a leak, **rotate immediately** in the provider's console (BotFather, etc.)

---

## 7. Final result

| Item | Status |
| --- | --- |
| Gateway | `ready`, stable |
| Dashboard | HTTP 200 |
| Channel accounts | all OK |
| Agents and session history | fully preserved, nothing lost |
| Models | all resolvable |
| Plugin drift | cleared (all `2026.8.1`) |
| Legacy `sessions.json` / `catalog.json` | both 0 |
| Real model call | returns 200 |

**Reinstalls: 0. Agents reconfigured: 0. Data lost: 0.**

---

## 8. Six lessons for anyone hitting the same wall

1. **Read the logs before editing config.** The `error.message` in `~/.openclaw/logs/stability/*.json` usually **states the solution outright**. Valid config that won't start is almost never a config problem.

2. **Major upgrade = run the migration first.** On startup failure, your first move should be `openclaw doctor --fix`, not a reinstall. It's the official migration tool, not folk magic.

3. **A backup is the precondition for acting, not an option.** And back up the whole `agents/` directory (session history lives there), not just `openclaw.json`.

4. **Be suspicious of "it broke because the config got too complex."** It sounds reasonable and is hard to refute, but it's usually a placeholder for not having found the real cause — and its implied "fix" is enormously expensive. Real causes narrow down to a specific file and a specific error line. **Any diagnosis you can't back with the verbatim error text — including your own intuition — is still just a guess.**

5. **`ready` does not mean usable.** Verify end to end: dashboard 200, `status` all green, a real model call returning 200 in the logs, legacy file counts at zero.

6. **The changes that don't error are the dangerous ones.** Migration failures crash loudly, but **path/semantics changes are silent** (pitfall 6 is exactly that). After upgrading, check where each agent's workspace actually resolves, not just the gateway's status.

---

## Appendix: one-shot triage checklist

```powershell
# 1. read the real error (the single most important step)
Get-ChildItem "$env:USERPROFILE\.openclaw\logs\stability" -Recurse |
  Sort-Object LastWriteTime -Descending | Select-Object -First 3 FullName

# 2. back up (including session history)
$bk = "D:\openclaw-backup-$(Get-Date -f yyyyMMdd-HHmmss)"
New-Item -ItemType Directory -Force $bk | Out-Null
Copy-Item "$env:USERPROFILE\.openclaw\openclaw.json" $bk -Force
robocopy "$env:USERPROFILE\.openclaw\agents" "$bk\agents" /E /NFL /NDL /NJH /NJS

# 3. read-only diagnosis
openclaw doctor

# 4. run the migration
openclaw doctor --fix

# 5. start
openclaw gateway run

# 6. verify (local requests bypass the proxy)
$env:HTTP_PROXY=""; $env:HTTPS_PROXY=""
(Invoke-WebRequest "http://127.0.0.1:18789/" -UseBasicParsing).StatusCode
openclaw status

# 7. confirm legacy artifacts are gone (both should be 0)
(Get-ChildItem "$env:USERPROFILE\.openclaw\agents" -Recurse -Filter "sessions.json" -EA SilentlyContinue).Count
(Get-ChildItem "$env:USERPROFILE\.openclaw\agents" -Recurse -Filter "catalog.json"  -EA SilentlyContinue).Count

# 8. spot-check that each agent's workspace still points where it used to (pitfall 6, silent failure)
Get-ChildItem "$env:USERPROFILE\.openclaw" -Directory -Force |
  Where-Object Name -like "workspace*" | Select-Object Name, LastWriteTime
```

---

## Version info

| Item | Value |
| --- | --- |
| OpenClaw | `2026.7.1-2` → `2026.8.1` |
| Node.js | 24.x |
| OS | Windows (10.0.26200) |
| Date | 2026-09-01 |

---

## Notes

Every command and error message here comes from a real repair session; paths, accounts and tokens have been replaced with placeholders.
Items marked as observed in sections 4 and 5 were re-checked in the same environment; the exact number of migration entries varies by environment and should not be taken as universal.

If this saved you a reinstall, pass it on to the next person about to upgrade.
