# Parallel migration handoffs

Each prompt owns exactly one handoff filename. Do not edit or delete another lane's handoff.

Required structure:

```text
STATE: AUDIT_READY | SOURCE_READY | DONE | BLOCKED_ON_USER
OWNED FILES CHANGED:
- absolute or repo-relative path
EVIDENCE:
- concrete inspected/implemented result
INTEGRATION REQUESTS:
- exact shared file/asset/API change deferred to an integrator
TESTS FOR INTEGRATOR:
- exact test/filter/scenario
REMAINING:
- honest unfinished work
```

Do not use a handoff to claim a build, QATS run, Blueprint compile, runtime result, or zero-reference result that this lane was not authorized to execute.

All ten prompts are peer workers launched simultaneously. None is an orchestrator or integrator, none waits for another, and none performs global build/editor/test/runtime actions. Each owns only its unique report file and returns its result directly to the user for later integration by the main Codex.

This folder is the permanent repository archive of those point-in-time worker reports. Paths and runtime ownership claims inside an individual handoff describe the state observed by that worker and may be superseded by the numbered migration documents or the central plan; the canonical current status is maintained one directory above.
