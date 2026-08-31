---
name: prevent-background-cleanup-eperm-crash
---

### T-1 隔离自然退出自动清理的 EPERM
- depends_on: []
- files: [/Users/bb/Projects/pi-extensions/extensions/background-bash/src/process-manager.ts, /Users/bb/Projects/pi-extensions/extensions/background-bash/tests/background-bash.integration.test.ts, /Users/bb/Projects/pi-extensions/.changeset/quiet-background-cleanup.md]
- refs: [R-1, AC-1]
- parallel: false
- verify: pnpm --dir /Users/bb/Projects/pi-extensions --filter @richardgill/pi-background-bash exec vitest --run tests/background-bash.integration.test.ts && pnpm --dir /Users/bb/Projects/pi-extensions --filter @richardgill/pi-background-bash tsc
- status: done
- step: impl
- agent: eperm-green-impl
- commit: c461632
- note:
