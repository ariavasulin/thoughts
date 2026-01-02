---
date: 2026-01-01T22:03:48-08:00
researcher: ariasulin
git_commit: N/A
branch: N/A
repository: N/A (system troubleshooting)
topic: "CodeLayer-Pro Daemon Startup Troubleshooting"
tags: [troubleshooting, codelayer, humanlayer, daemon, homebrew]
status: in_progress
last_updated: 2026-01-01
last_updated_by: ariasulin
type: troubleshooting
---

# Handoff: CodeLayer-Pro "Auto-Detection Failed" Error Troubleshooting

## Task(s)
**Status: In Progress**

User reported persistent "Auto-Detection Failed - Could not auto-detect Claude path" error in CodeLayer-Pro despite Claude binary being correctly installed at `/opt/homebrew/bin/claude`.

Attempted version downgrades:
- 0.20.0 → 0.19.0 (did not resolve)
- 0.19.0 → 0.18.1-pro (did not resolve)

## Critical References
- CodeLayer-Pro logs: `/Users/ariasulin/Library/Logs/dev.humanlayer.wui.pro/CodeLayer-Pro.log`
- HumanLayer config: `~/.config/humanlayer/humanlayer.json`
- Daemon files: `~/.humanlayer/` (contains `daemon-pro.db`, `daemon-pro.sock`, `codelayer.json`)

## Recent changes
No code changes - this is system troubleshooting. Version downgrades performed via:
```
cd $(brew --repo humanlayer/humanlayer) && git checkout <commit> -- Casks/codelayer-pro.rb
brew reinstall --cask codelayer-pro
```

## Learnings

### Root Cause Analysis
The "Auto-Detection Failed" error in the UI is likely a **symptom**, not the root cause. Log analysis reveals:

1. **Primary failure**: The `cld` daemon crashes immediately with `"Unable to locate HLD Port"` error
2. **Cascade effect**: This causes "Failed to fetch Claude config" and "Failed to auto-detect Claude" errors in the webview

### Key Log Evidence (`CodeLayer-Pro.log`)
```
[ERROR] [Daemon] production: error: Unable to locate HLD Port
[ERROR] [Tauri] Daemon stdout closed immediately (0 bytes read)
[ERROR] [Tauri] Failed to auto-start daemon: Daemon stdout closed before reporting port
[ERROR] [Console] Failed to auto-detect Claude: {}
```

### Important Finding
When `hld-pro` is run **manually from terminal**, it works correctly:
- Successfully auto-detects Claude at `/opt/homebrew/bin/claude`
- Creates socket at `~/.humanlayer/daemon-pro.sock`
- Starts HTTP server on port 7780

This suggests the issue is either:
1. A race condition where `cld` starts before `hld` is ready
2. An environment/PATH difference when launched from the GUI vs terminal

### Relevant Binaries
- Claude: `/opt/homebrew/bin/claude` → symlink to `/opt/homebrew/Caskroom/claude-code/2.0.71/claude`
- hld-pro: `/opt/homebrew/bin/hld-pro` → `/Applications/CodeLayer-Pro.app/Contents/Resources/bin/hld`
- cld-pro: `/opt/homebrew/bin/cld-pro` → `/Applications/CodeLayer-Pro.app/Contents/Resources/bin/cld`

## Artifacts
- This handoff document

## Action Items & Next Steps

1. **Verify the hypothesis**: Start `hld-pro` manually before opening CodeLayer-Pro.app to see if the UI works when hld is already running
   ```bash
   pkill -9 -f hld; pkill -9 -f cld; rm -f ~/.humanlayer/daemon-pro.sock
   hld-pro &
   # Then open CodeLayer-Pro.app
   ```

2. **If that works**: The issue is confirmed as a daemon startup race condition - report to HumanLayer

3. **If that doesn't work**: The issue may be GUI-specific environment handling - check if there's a way to manually configure Claude path in:
   - `~/.config/humanlayer/humanlayer.json`
   - `~/.humanlayer/codelayer.json`

4. **Consider reporting**: Open issue at https://github.com/humanlayer/humanlayer or contact HumanLayer support with log excerpts

## Other Notes

### Useful Commands
- Check daemon status: `ps aux | grep -E "[h]ld|[c]ld"`
- View logs: `tail -f "/Users/ariasulin/Library/Logs/dev.humanlayer.wui.pro/CodeLayer-Pro.log"`
- Run hld in debug mode: `hld-pro -debug`
- Show config: `codelayer-pro config show`

### Version Downgrade Process
```bash
cd $(brew --repo humanlayer/humanlayer)
git log --oneline -10 -- Casks/codelayer-pro.rb  # Find version commits
git checkout <commit> -- Casks/codelayer-pro.rb
brew reinstall --cask codelayer-pro
git checkout HEAD -- Casks/codelayer-pro.rb  # Reset tap
```

### Installed Version History
- 0.20.0 (current in tap)
- 0.19.0
- 0.18.1-pro ← currently installed
- 0.18.0-pro
