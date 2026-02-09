# 🔍 clawdscan — Clawdbot Session Health Analyzer

**Diagnose bloat, find zombies, reclaim disk space.**

A zero-dependency CLI tool that analyzes Clawdbot's session JSONL files and provides actionable diagnostics for session health management.

## The Problem

Clawdbot sessions grow unbounded. Every message, tool call, compaction, and model switch is appended to JSONL files that never get cleaned up. Over time:

- **Sessions bloat** — a single session can reach 16MB / 5,000+ messages
- **Zombie sessions** — created but never used, cluttering the filesystem
- **Stale sessions** — weeks old, never to be revisited, eating disk
- **No visibility** — you can't see which sessions are healthy vs. problematic

Relates to: [clawdbot/clawdbot#1808](https://github.com/clawdbot/clawdbot/issues/1808) — Session bloat with moderate usage

## Installation

```bash
# It's a single Python file — just copy or symlink
cp clawdscan.py /usr/local/bin/clawdscan
chmod +x /usr/local/bin/clawdscan

# Or run directly
python3 clawdscan.py <command>
```

**Requirements:** Python 3.9+, zero dependencies.

## Usage

### Full Health Scan
```bash
clawdscan scan
```

Scans all agents and sessions, classifies health, reports issues, and gives recommendations:
- 🔴 **Critical** — mega-bloated (>5MB) or message overflow (>900 msgs)
- 🟡 **Warning** — bloated (>1MB), stale (>7 days), message-heavy (>300 msgs)
- 👻 **Zombie** — ≤2 messages, created >48h ago
- 📦 **Compacted** — sessions that have been compacted 3+ times
- ✅ **Healthy** — no issues detected

### Top Sessions
```bash
clawdscan top -n 20              # Top 20 by size
clawdscan top --sort messages    # Top by message count
clawdscan top --agent main       # Filter to one agent
```

### Inspect a Session
```bash
clawdscan inspect e6934c05       # First 8+ chars of session ID
```

Shows detailed breakdown: messages, timeline, models used, tool usage histogram, compaction count, custom events.

### Tool Usage
```bash
clawdscan tools                  # Aggregate tool usage across all sessions
clawdscan tools --agent main     # Per-agent
```

### Model Usage
```bash
clawdscan models                 # Which models are used how often
```

### Disk Usage
```bash
clawdscan disk                   # Full breakdown with percentile stats
```

### Cleanup
```bash
clawdscan clean --zombies                    # Preview zombie cleanup
clawdscan clean --zombies --execute          # Actually move zombie sessions
clawdscan clean --stale-days 28              # Preview stale session cleanup
clawdscan clean --stale-days 28 --execute    # Execute stale cleanup
clawdscan clean --min-size 5M               # Preview large session cleanup
```

**Safety:** All cleanup is dry-run by default. Add `--execute` to actually move files. Files are moved to `~/.clawdbot/archived-sessions/<timestamp>/`, never deleted.

## Example Output

```
🔍 clawdscan v0.1.0 — Session Health Report
   Scanning: /Users/yajat/.clawdbot
   Time: 2026-02-09 12:33:45 UTC

📁 Agent: main
   Sessions: 1140 active, 85 deleted
   Disk: 133.5 MB
   ⚠️  132 sessions with issues (showing top 10):

   1645637f-2c1       15.7 MB   5314 msgs      3d ago  🔴 mega-bloat 🔴 msg-overflow 📦 over-compacted
                 compactions: 26, duration: 3d 23h, models: claude-opus-4-5
   ...

═══ Summary ═══════════════════════════════════════════════
  Total sessions: 1754
  Total disk:     148.6 MB
  Issues found:   417
  🔴 Critical:    9 sessions
  🟡 Warning:     404 sessions
  👻 Zombies:     30 sessions

  Cleanup potential:
    Zombie cleanup:  60.5 KB (30 sessions)
    Stale cleanup:   46.1 MB (393 sessions)
```

## JSON Export

```bash
clawdscan scan --json report.json --verbose
```

Exports full scan data as JSON for integration with monitoring, dashboards, or CI pipelines.

## How It Integrates

- **Clawdbot cron job** — schedule daily scans and alert on bloat:
  ```
  clawdscan scan --json /tmp/clawdscan.json && cat /tmp/clawdscan.json | jq '.critical_count'
  ```
- **ClowdControl** — feed session health metrics into the Mission Control dashboard
- **Heartbeat checks** — add to HEARTBEAT.md for periodic monitoring
- **Skill potential** — could become a Clawdbot skill on ClawdHub

## Architecture

```
clawdscan.py (single file, ~500 lines)
├── Session JSONL parser (handles all entry types)
├── Health classifier (size/age/msg/zombie thresholds)
├── 7 commands: scan, top, inspect, tools, models, disk, clean
├── Color output (respects NO_COLOR env var)
└── Safe cleanup (archive, never delete)
```

## License

MIT

## Author

Built by Chhotu (Agent Forge, Feb 9 2026) — solving a real pain point we hit daily.
