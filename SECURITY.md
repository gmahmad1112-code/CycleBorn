Title: Trap Policy and Lifecycle Overrides

Status: Draft
Owners: VAutoTraps (policy), VLifecycle (integration toggle)
Log Prefixes: [TrapManager], [TrapLifecyclePolicy]

1. Goals
- Define how armed traps influence lifecycle snapshot behavior during arena enter/exit.
- Provide a clear policy hook that VLifecycle can query/execute if enabled by configuration.

2. Interfaces
public interface ITrapLifecyclePolicy {
  // Called prior to normal lifecycle handling
  void OnBeforeLifecycleEnter(LifecycleContext ctx);
  void OnBeforeLifecycleExit(LifecycleContext ctx);

  // Decisions
  bool AllowSnapshotOnEnter(LifecycleContext ctx); // default true
  bool AllowRestoreOnExit(LifecycleContext ctx);   // default true
  bool ForceBuffClearOnExit(LifecycleContext ctx); // default false
}

3. Default Behavior
- If IntegrationAllowTrapOverrides == false (VLifecycle config), VLifecycle ignores trap policy.
- If true, VLifecycle queries the registered ITrapLifecyclePolicy instance per enter/exit.

4. Policy Rules (initial)
- Entering a zone with an ARMED trap owned by another player and the character is within trap radius:
  - AllowSnapshotOnEnter = false (optional per config VAutoTraps.allowLifecycleOverrides)
  - Rationale: keep live state without snapshotting to avoid restore exploits during hostile trap engagement.
- On Exit from such a state:
  - ForceBuffClearOnExit = true to remove bind-walk / trap-specific penalties if policy dictates cleanup.
  - AllowRestoreOnExit remains true unless config specifies persisting effects beyond the arena.

5. Data Dependencies
- TrapPolicyResolver must answer queries:
  - HasArmedTrapInRange(characterId, zoneId, position): (bool, ownerUserId?)
  - GetTrapById(trapId)
  - Enumerate active traps in zone

6. Logging
- "[TrapLifecyclePolicy] Enter override: snapshotOnEnter={bool} char={characterId} zone={zoneId} reason=armedTrapInRange"
- "[TrapLifecyclePolicy] Exit override: forceBuffClear={bool} char={characterId} zone={zoneId}"

7. Acceptance Criteria
- With overrides enabled and a hostile armed trap in range, Enter does not create a snapshot.
- On Exit after hostile trap interaction, bind-walk debuff is cleared if ForceBuffClearOnExit=true.
- With overrides disabled, lifecycle proceeds unaffected by traps.

8. Open Items
- Consider time-bounded suppression (e.g., only first Enter within N seconds after arming).
- Provide per-zone or per-trap opt-out flags if needed.
