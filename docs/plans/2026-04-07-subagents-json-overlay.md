# `subagents.json` Overlay Config Plan

**Goal:** Let users override built-in agent frontmatter (model, thinking, description, skills, etc.) and disable built-in agents via a new `subagents.json` config file, without mutating any `.md` files on disk.

**Architecture:** A new `subagents-config.ts` module loads and validates `subagents.json` from project (`.pi/subagents.json`) and user (`~/.pi/agent/subagents.json`) scopes, returning a merged in-memory overlay. The overlay is applied inside `agents.ts` immediately after each agent is parsed from its `.md` file, and disabled agent names are filtered out before agents are returned to callers. Because the overlay is applied at the bottom of the loader, every downstream consumer (chain resolution, model warnings, agent-management list/get, runtime execution) sees the merged config without any further changes.

**Tech Stack:** TypeScript (Node `--experimental-strip-types`), `node:fs`/`node:path`/`node:os`, `node:test`/`node:assert/strict` (existing test infrastructure under `test/unit/`).

---

## Design notes (read before executing tasks)

### Resolved precedence (highest wins)
1. Project `subagents.json` overrides
2. User `subagents.json` overrides
3. `.md` frontmatter (built-in, user, or project)

`disabled` is **unioned** across both files (a project file cannot re-enable an agent the user disabled and vice versa — if either says "disabled", it's disabled). This is intentional: disabling is conservative, overriding is precise.

### Test isolation (critical — read before writing any test in Task 2+)
Because the overlay loader reads `~/.pi/agent/subagents.json` from the developer's real `os.homedir()`, **every test that calls `discoverAgents`/`discoverAgentsAll` would silently merge whatever the contributor has on disk**. This pollutes existing tests too (e.g. `test/unit/agent-frontmatter.test.ts` calls `discoverAgents`).

The fix is a single env-var seam in `subagents-config.ts`:
```ts
function userSubagentsPath(): string {
  const override = process.env.PI_SUBAGENTS_HOME;
  const home = override && override.length > 0 ? override : os.homedir();
  return path.join(home, ".pi", "agent", "subagents.json");
}
```
Every test in Tasks 2/3/4 must wrap its body in a `before/after` that sets `process.env.PI_SUBAGENTS_HOME` to a tempdir (which by default contains no `subagents.json`, so the user-scope read becomes a no-op) and restores the previous value after. The optional `opts?: { userFilePath?: string; projectFilePath?: string }` parameter on `loadSubagentsOverlay` from Task 1 is **only** used by Task 1's own pure unit tests; once the overlay is wired into discovery in Task 2, all higher-level tests must use `PI_SUBAGENTS_HOME` instead because they cannot reach the seam.

Task 1 must therefore implement *both* the env-var path and the `opts` seam. They are not redundant: `opts` enables pure-function unit tests with no env mutation, while `PI_SUBAGENTS_HOME` is the only thing that works once `discoverAgents` is the entry point.

### Schema (`subagents.json`)
```jsonc
{
  // Optional: overlay specific frontmatter fields onto agents by name.
  // Only fields listed here are overridden; everything else (including
  // systemPrompt body) is inherited from the .md file.
  "agents": {
    "scout": {
      "model": "anthropic/claude-opus-4-6",
      "thinking": "high"
    },
    "reviewer": {
      "description": "Custom reviewer description",
      "skills": ["code-review", "security-audit"]
    }
  },

  // Optional: agent names to skip loading entirely. They will not appear
  // in `discoverAgents`, will not be invokable, and chains that reference
  // them will produce the same "unknown agent" error as today.
  "disabled": ["delegate", "worker"]
}
```

### Overridable fields (allowlist)
Only these `AgentConfig` fields can be set via `agents.<name>` overrides — anything else is rejected with a warning at load time:

- `description` (string)
- `model` (string)
- `thinking` (string)
- `tools` (string[]) — stored on `AgentConfig.tools` as `string[]` directly; no comma-joining needed (the frontmatter comma-split happens at parse time in `agents.ts:96-99`, but the in-memory representation is already an array)
- `skills` (string[])
- `extensions` (string[])
- `output` (string)
- `defaultReads` (string[])
- `defaultProgress` (boolean)
- `interactive` (boolean)
- `maxSubagentDepth` (number)

**Not overridable** (for safety / scope):
- `name` — would break the lookup keying
- `systemPrompt` — body content; if you want to rewrite the prompt, write a real `.md` file under your user/project agents dir (existing override path)
- `source` / `filePath` — internal metadata
- `mcpDirectTools` — derived from `tools`, follows automatically
- `extraFields` — frontmatter passthrough; out of scope for v1

### What this feature does NOT do (v1 scope guard)
- It does **not** support overriding chains or disabling chains. Chain override is symmetric in spirit but built-in chains do not currently exist (`loadChainsFromDir` is only called for user/project dirs in `agents.ts:264-268`), so the value is low. If we add built-in chains later, extend the schema then.
- It does **not** mutate `.md` files. There is already a `handleUpdate` path in `agent-management.ts:405-450` that does that destructively; this feature is the non-destructive complement.
- It does **not** add a new CLI command. Configuration is purely file-based, matching the existing `settings.json -> skills` pattern.

### Edge cases (must be handled in tests)
1. `agents.<name>` references a name that doesn't exist among any loaded agent (built-in, user, or project, **before disabled-filtering**) → emit a warning, ignore the override (do not crash). "Known" means "appeared on disk", regardless of whether it was subsequently disabled.
2. An agent appears in both `agents` and `disabled` → `disabled` wins, the override is silently dropped (it would never be applied anyway). **No warning** — this is a legitimate "I'm temporarily hiding this but keeping my override around" pattern.
3. Both project and user `subagents.json` define overrides for the same agent → project wins field-by-field (deep-merge per field, not whole-object replace), so the user can set `model` and the project can set `thinking` and both apply.
4. `subagents.json` is missing → no-op, no warning.
5. `subagents.json` exists but is invalid JSON → emit a warning, treat as empty.
6. `agents.<name>` contains an unknown field (e.g. `systemPrompt`, `name`, typo like `modle`) → emit a warning, drop just that field, keep the rest of the override.
7. `disabled` is not an array, or contains non-strings → emit a warning, treat as empty.
8. **Override sets a boolean field to `false`** (e.g. `defaultProgress: false` on an agent whose frontmatter has `defaultProgress: true`) → the `false` value must be applied. This means `applyOverride` must use `if (key in override)` / `if (override.x !== undefined)` semantics, **never** truthy checks like `if (override.defaultProgress)`. See Task 2 for the precise contract.

---

### Task 1: Create `subagents-config.ts` loader module

**Context:**
This task introduces the new config file but does not yet wire it into agent loading. We add a standalone, pure module so it can be unit-tested in isolation against tempdirs without touching the rest of the codebase. The module is responsible for: (a) locating up to two `subagents.json` files (project + user), (b) parsing them safely, (c) validating each entry against the allowlist of overridable fields, (d) merging them into a single overlay with project-precedence-per-field for overrides and union semantics for `disabled`, and (e) collecting human-readable warnings for every problem encountered. No file I/O outside the two known paths. No mutation of any other module.

The module mirrors the existing convention used by `skills.ts:158-185` for `collectSettingsSkillPaths` (read both `.pi/<file>.json` and `~/.pi/agent/<file>.json`, swallow errors per file). Reuse the `CONFIG_DIR = ".pi"` and `AGENT_DIR = path.join(os.homedir(), ".pi", "agent")` constants — duplicate them locally in the new file rather than exporting them from `skills.ts` (the constants are not currently exported and we want the new module to stand alone).

**Files:**
- Create: `subagents-config.ts`
- Test: `test/unit/subagents-config.test.ts`

**What to implement:**

In `subagents-config.ts`:

```ts
export interface AgentOverride {
  description?: string;
  model?: string;
  thinking?: string;
  tools?: string[];
  skills?: string[];
  extensions?: string[];
  output?: string;
  defaultReads?: string[];
  defaultProgress?: boolean;
  interactive?: boolean;
  maxSubagentDepth?: number;
}

export interface SubagentsOverlay {
  agents: Map<string, AgentOverride>;  // merged overrides, by agent name
  disabled: Set<string>;                // union of both scopes
  warnings: string[];                   // load/validation problems
}

/**
 * Load and merge subagents.json from project (.pi/subagents.json) and user
 * (~/.pi/agent/subagents.json) scopes. Project overrides take precedence
 * over user overrides at the per-field level. `disabled` is unioned.
 *
 * Never throws. All errors become entries in `overlay.warnings`.
 */
export function loadSubagentsOverlay(cwd: string): SubagentsOverlay;
```

The set of overridable field names lives in a single internal constant:

```ts
const OVERRIDABLE_FIELDS = new Set<keyof AgentOverride>([
  "description", "model", "thinking", "tools", "skills",
  "extensions", "output", "defaultReads", "defaultProgress",
  "interactive", "maxSubagentDepth",
]);
```

User file path resolution (used by both production and tests):
```ts
function userSubagentsPath(): string {
  const override = process.env.PI_SUBAGENTS_HOME;
  const home = override && override.length > 0 ? override : os.homedir();
  return path.join(home, ".pi", "agent", "subagents.json");
}
```
This env var is the **only** mechanism that lets discovery-level tests (Tasks 2/3/4) avoid reading the developer's real `~/.pi/agent/subagents.json`.

Loading order inside `loadSubagentsOverlay`:
1. Read user file at `userSubagentsPath()` (or `opts.userFilePath` if provided) → produce a partial overlay.
2. Read project file at `<cwd>/.pi/subagents.json` (or `opts.projectFilePath` if provided) → produce a partial overlay.
3. Merge: for `agents`, start with user overrides, then for each project override do a per-field shallow merge into the user override (project field wins). For `disabled`, union both sets.
4. Return the combined `SubagentsOverlay` with all collected warnings concatenated (user warnings first, then project warnings, prefixed with the source path so the user can tell which file is broken).

Per-file parsing helper (internal, not exported):

```ts
function parseOverlayFile(filePath: string): {
  agents: Map<string, AgentOverride>;
  disabled: Set<string>;
  warnings: string[];
};
```

It must:
- Return empty maps/sets and no warnings if `filePath` does not exist.
- Catch `JSON.parse` failures and return one warning: `` `${filePath}: invalid JSON: <message>` ``.
- For each entry under `agents`, validate it is a plain object; for each field in that entry, drop unknown fields with a warning `` `${filePath}: agents.${name}: unknown field '${field}'` ``.
- For array-typed fields (`tools`, `skills`, `extensions`, `defaultReads`), require `Array.isArray(value)` and every element to be a string; otherwise drop the field with a warning.
- For boolean-typed fields (`defaultProgress`, `interactive`), require `typeof value === "boolean"`.
- For numeric `maxSubagentDepth`, require `Number.isInteger(value) && value >= 0`.
- For string fields, require `typeof value === "string"`.
- For `disabled`, require `Array.isArray(value)`; drop non-string entries with a warning, keep the rest.
- If `agents` is present but not a plain object, emit a warning and treat as empty.

**Steps:**
- [ ] Create `test/unit/subagents-config.test.ts` with **a single top-level `describe("subagents-config", () => { ... })` wrapper** containing all test cases. This describe name is mandatory: it is what `--test-name-pattern="subagents-config"` matches against (node:test matches the pattern against the full hierarchical test name, which includes parent describes). Mirror `test/unit/agent-frontmatter.test.ts:1-17` for tempdir setup/teardown.

  Test cases (all use the `opts` seam — no env var, no real I/O outside tempdirs):
  - missing file → empty overlay, no warnings
  - valid project-only file with one agent override and one disabled name → overlay reflects both
  - valid user-only file via `opts.userFilePath` pointing into a tempdir
  - both files with overlapping agent: project field wins, user-only field is preserved
  - both files with disabled lists: union semantics
  - invalid JSON → warning includes the file path, no crash
  - unknown field in `agents.<name>` → warning, other fields kept
  - wrong type for an array field (e.g. `skills: "code-review"`) → warning, field dropped, rest kept
  - `disabled` not an array → warning, treated as empty
  - **boolean override sets `defaultProgress: false`** → overlay's `agents.get(name).defaultProgress === false` (not undefined, not dropped)
  - separate `describe("detectUnknownAgentNames", ...)` block can be added now or in Task 4 — Task 4 adds it, so leave it out here.
- [ ] Run `npm run test:unit -- --test-name-pattern="subagents-config"`
  - Did it fail with "Cannot find module './subagents-config.ts'" or similar? If it passed unexpectedly, stop and investigate why.
- [ ] Implement `subagents-config.ts` per the spec above. Both the env-var path resolution AND the `opts` seam must be implemented now even though only the seam is exercised by Task 1's tests — Tasks 2/3/4 depend on the env var.
- [ ] Run `npm run test:unit -- --test-name-pattern="subagents-config"`
  - Did all tests pass? If not, fix the failures and re-run before continuing.
- [ ] Run `npm run test:unit`
  - Did all tests pass? If not, fix the failures and re-run before continuing. **Note: this is a vacuous green check** — nothing imports the new module yet, so existing tests cannot regress. The check exists only to confirm the new test file itself doesn't break the suite (e.g. via syntax error).
- [ ] Commit with message: `feat(subagents-config): add subagents.json overlay loader`

**Acceptance criteria:**
- [ ] `subagents-config.ts` exports `loadSubagentsOverlay`, `SubagentsOverlay`, and `AgentOverride`.
- [ ] `loadSubagentsOverlay` honors both `opts.userFilePath`/`opts.projectFilePath` and the `PI_SUBAGENTS_HOME` env var (env var only takes effect when `opts.userFilePath` is not provided).
- [ ] All new unit tests pass.
- [ ] `npm run test:unit` is fully green.
- [ ] No other source files were touched in this commit.

---

### Task 2: Apply agent overrides inside the loader

**Context:**
Now that the overlay loader exists and is tested in isolation, wire it into `agents.ts` so that overrides are applied to every agent at discovery time, regardless of scope. We deliberately apply the overlay **after** an agent is parsed from its `.md` file but **before** it is returned, so every downstream consumer (`agent-management.ts` list/get, chain resolution in `settings.ts`, runtime in `subagent-runner.ts`, model warnings) automatically sees the merged config without any additional changes.

The key implementation point: `loadAgentsFromDir` (in `agents.ts:63-166`) is called from three places (`discoverAgents` and `discoverAgentsAll`), each potentially passing the same overlay. To avoid loading the overlay three times per discovery call, **load the overlay once at the top of `discoverAgents`/`discoverAgentsAll` and pass it down to `loadAgentsFromDir` as a new parameter**.

This task does not implement disabled-filtering (that is Task 3) and does not surface warnings to callers (that is Task 4). Keep this task focused on field-level overrides only.

**Files:**
- Modify: `agents.ts`
- Test: `test/unit/subagents-config-overlay.test.ts`

**What to implement:**

1. Import the overlay loader at the top of `agents.ts`:
   ```ts
   import { loadSubagentsOverlay, type SubagentsOverlay, type AgentOverride } from "./subagents-config.ts";
   ```

2. Change the signature of `loadAgentsFromDir` (currently at `agents.ts:63`) to accept an optional overlay:
   ```ts
   function loadAgentsFromDir(dir: string, source: AgentSource, overlay?: SubagentsOverlay): AgentConfig[]
   ```

3. After the existing `agents.push({...})` block (currently `agents.ts:140-162`), apply the override **only if** `overlay?.agents.has(frontmatter.name)`. Implement an internal helper:
   ```ts
   function applyOverride(agent: AgentConfig, override: AgentOverride): AgentConfig
   ```
   that returns a new `AgentConfig` with each **present** field of `override` replacing the corresponding field on `agent`. "Present" means `override.x !== undefined` (or equivalently `"x" in override` since the loader in Task 1 never assigns `undefined`). **Never** use truthy checks like `if (override.defaultProgress)` — that would silently discard a deliberate `false`. Concretely:
   ```ts
   function applyOverride(agent: AgentConfig, o: AgentOverride): AgentConfig {
     const next = { ...agent };
     if (o.description !== undefined) next.description = o.description;
     if (o.model !== undefined) next.model = o.model;
     if (o.thinking !== undefined) next.thinking = o.thinking;
     if (o.tools !== undefined) next.tools = [...o.tools];
     if (o.skills !== undefined) next.skills = [...o.skills];
     if (o.extensions !== undefined) next.extensions = [...o.extensions];
     if (o.output !== undefined) next.output = o.output;
     if (o.defaultReads !== undefined) next.defaultReads = [...o.defaultReads];
     if (o.defaultProgress !== undefined) next.defaultProgress = o.defaultProgress;
     if (o.interactive !== undefined) next.interactive = o.interactive;
     if (o.maxSubagentDepth !== undefined) next.maxSubagentDepth = o.maxSubagentDepth;
     return next;
   }
   ```
   Do not mutate the input. Do not touch `name`, `systemPrompt`, `source`, `filePath`, `mcpDirectTools`, or `extraFields`. For array fields, replace wholesale (no concatenation) and copy via spread to avoid sharing references with the overlay map.

4. Refactor `agents.ts:140-162` so the push uses the result of `applyOverride` if an override exists, otherwise the original config. Cleanest pattern: build the `AgentConfig` first into a `const base`, then `agents.push(overlay?.agents.has(base.name) ? applyOverride(base, overlay.agents.get(base.name)!) : base)`.

5. In `discoverAgents` (currently `agents.ts:229`), load the overlay once at the top: `const overlay = loadSubagentsOverlay(cwd);` and pass it to all four `loadAgentsFromDir` calls (built-in, userOld, userNew, project).

6. In `discoverAgentsAll` (currently `agents.ts:246`), do the same: load overlay once and thread it through every `loadAgentsFromDir` call.

**Steps:**
- [ ] Create `test/unit/subagents-config-overlay.test.ts` mirroring `test/unit/agent-frontmatter.test.ts` for tempdir setup. Wrap **all** test cases in a single top-level `describe("subagents-config-overlay", () => { ... })` so `--test-name-pattern="subagents-config-overlay"` matches.

  Mandatory test isolation harness in this file:
  ```ts
  let tempHome: string;
  let prevHomeEnv: string | undefined;
  beforeEach(() => {
    tempHome = fs.mkdtempSync(path.join(os.tmpdir(), "pi-subagents-overlay-home-"));
    prevHomeEnv = process.env.PI_SUBAGENTS_HOME;
    process.env.PI_SUBAGENTS_HOME = tempHome;
  });
  afterEach(() => {
    if (prevHomeEnv === undefined) delete process.env.PI_SUBAGENTS_HOME;
    else process.env.PI_SUBAGENTS_HOME = prevHomeEnv;
    fs.rmSync(tempHome, { recursive: true, force: true });
  });
  ```
  This guarantees the user-scope read points at an empty tempdir for every test, isolating it from the contributor's real home. Project-scope `subagents.json` files can be written directly into the test's tempdir cwd at `.pi/subagents.json`.

  **Existing tests at risk:** `test/unit/agent-frontmatter.test.ts` already calls `discoverAgents` and does not set `PI_SUBAGENTS_HOME`. As part of this task, **add the same `beforeEach`/`afterEach` env-var harness to `agent-frontmatter.test.ts`** to prevent the contributor's real `~/.pi/agent/subagents.json` from polluting it once the overlay is wired in. Audit `test/unit/agent-scope.test.ts` and `test/unit/agent-selection.test.ts` for the same risk; add the harness wherever `discoverAgents`/`discoverAgentsAll` is called.

  Test cases:
  - An agent with `model: foo` in its `.md` and `model: bar` in `subagents.json` → discovered agent has `model: "bar"`.
  - An agent with no `model` in its `.md` and `model: bar` in `subagents.json` → discovered agent has `model: "bar"`.
  - An agent with `model: foo` in its `.md` and no override → discovered agent has `model: "foo"` (regression check).
  - Override sets `defaultProgress: false` on an agent that had `defaultProgress: true` in frontmatter → discovered agent has `defaultProgress: false`. **This test will catch the truthy-check bug if `applyOverride` is implemented sloppily.**
  - Override sets `skills: ["a", "b"]` → discovered agent has exactly those two skills (replace, not merge).
  - User-scope override (write `subagents.json` to `tempHome/.pi/agent/subagents.json`) → discovered agent reflects it.
  - Project-scope override on the same agent overrides only the field it specifies; user-scope fields not touched by project remain.
- [ ] Run `npm run test:unit -- --test-name-pattern="subagents-config-overlay"`
  - Did it fail with assertion errors (overrides not applied yet)? If it passed unexpectedly, stop and investigate why.
- [ ] Implement the changes in `agents.ts` per the spec above.
- [ ] Run `npm run test:unit -- --test-name-pattern="subagents-config-overlay"`
  - Did all tests pass? If not, fix the failures and re-run before continuing.
- [ ] Run `npm run test:unit`
  - Did all tests pass? In particular, `agent-frontmatter.test.ts`, `agent-selection.test.ts`, and `agent-scope.test.ts` must remain green (with the new env-var harness applied where needed). If not, fix and re-run.
- [ ] Run `npm run test:integration` if it is reasonably fast on this machine; otherwise note as a manual follow-up.
- [ ] Commit with message: `feat(agents): apply subagents.json overrides at load time`

**Acceptance criteria:**
- [ ] All new overlay tests pass.
- [ ] `npm run test:unit` is fully green.
- [ ] Built-in agent files in `agents/*.md` are unchanged on disk.
- [ ] Overlay is loaded exactly once per `discoverAgents`/`discoverAgentsAll` call (verify by code inspection — no redundant `loadSubagentsOverlay` calls inside `loadAgentsFromDir`).

---

### Task 3: Filter disabled agents from discovery

**Context:**
With overrides working, add the second half of the feature: hiding agents listed under `disabled` from every consumer. Filtering happens at the very end of `discoverAgents` and `discoverAgentsAll`, after the merge step in `mergeAgentsForScope`, so that disabled built-ins disappear from `list`, are not invokable, and produce the existing "unknown agent" error if referenced from a chain (no special-casing needed — they simply do not exist in the returned array).

Note that `disabled` filtering must apply to **all sources** (built-in, user, project), not just built-ins, even though the most common use case is hiding built-ins. A user can disable their own agent if they want, and we should not surprise them by silently keeping it.

**Files:**
- Modify: `agents.ts`
- Test: extend `test/unit/subagents-config-overlay.test.ts` (same file as Task 2)

**What to implement:**

1. After the `mergeAgentsForScope` call in `discoverAgents` (currently `agents.ts:241`), filter the merged array:
   ```ts
   const visibleAgents = overlay.disabled.size > 0
     ? agents.filter((a) => !overlay.disabled.has(a.name))
     : agents;
   return { agents: visibleAgents, projectAgentsDir };
   ```

2. In `discoverAgentsAll` (currently `agents.ts:246`), apply the same filter to each of `builtin`, `user`, and `project` arrays before returning. Do not filter `chains` — chain disabling is out of scope for v1 (per design notes above).

3. Conflict handling: if a name appears in both `overlay.agents` and `overlay.disabled`, the agent is filtered out and the override is effectively dead. This is fine; no extra warning is needed at this layer because Task 4 will handle "unknown agent name in overrides" warnings, which subsumes this case (a disabled agent is unknown from the overrides' perspective).

**Steps:**
- [ ] Add new test cases to the existing `describe("subagents-config-overlay", ...)` block in `test/unit/subagents-config-overlay.test.ts` (the env-var harness from Task 2 already covers them):
  - Built-in agent `scout` listed under `disabled` → `discoverAgents(cwd, "both").agents` does not contain `scout`, but other built-ins are still present.
  - User agent listed under `disabled` → not present in discovery results.
  - Agent listed under both `disabled` and `agents.<name>` → not present (disabled wins). Verify that no warning is emitted for this combination (per edge case 2).
  - Empty `disabled` array / missing `disabled` → all agents present (regression).
- [ ] Run `npm run test:unit -- --test-name-pattern="subagents-config-overlay"`
  - Did the new tests fail with "expected agent to be missing"? If they passed unexpectedly, stop and investigate why.
- [ ] Implement filtering in `agents.ts` per the spec above.
- [ ] Run `npm run test:unit -- --test-name-pattern="subagents-config-overlay"`
  - Did all tests pass? If not, fix and re-run.
- [ ] Run `npm run test:unit`
  - Did all tests pass? If not, fix and re-run.
- [ ] Commit with message: `feat(agents): honor disabled list in subagents.json`

**Acceptance criteria:**
- [ ] Disabled built-ins are absent from `discoverAgents` and `discoverAgentsAll` results.
- [ ] Non-disabled agents are unaffected.
- [ ] `npm run test:unit` is fully green.

---

### Task 4: Surface overlay warnings via discovery

**Context:**
The overlay loader collects warnings (invalid JSON, unknown fields, wrong types) and the overlay can also reference agent names that do not exist after loading (typo, removed agent, disabled agent). Right now those warnings are computed but never surfaced. This task threads them through `discoverAgentsAll` so that `agent-management.ts` (which already builds a `warnings: string[]` channel for its handlers — see `agent-management.ts:382, 397, 432`) can display them in `list`/`get` output. We do not pipe them into `discoverAgents` because that function returns a narrower shape and most of its callers only need the agent list.

Why this is its own task: it touches a public-ish API (`discoverAgentsAll`'s return shape) and benefits from being a small, reviewable diff that doesn't get tangled with the loader/filter logic above.

**Files:**
- Modify: `agents.ts`
- Modify: `subagents-config.ts` (add an "unknown agent in overrides" warning helper)
- Test: extend `test/unit/subagents-config-overlay.test.ts`

**What to implement:**

1. Add a small exported helper in `subagents-config.ts`:
   ```ts
   /**
    * Given a list of known agent names (after loading and merging from disk),
    * return warnings for any names referenced in the overlay (in `agents` or
    * `disabled`) that do not exist. Mutates nothing.
    */
   export function detectUnknownAgentNames(
     overlay: SubagentsOverlay,
     knownNames: Iterable<string>,
   ): string[];
   ```
   It returns one warning string per unknown name, formatted like:
   `subagents.json: agents.<name> references unknown agent '<name>'`
   `subagents.json: disabled[] references unknown agent '<name>'`

2. In `discoverAgentsAll` (currently `agents.ts:246`), the implementation order **must** be:
   1. Load overlay (already done at the top of the function from Task 2).
   2. Load builtin/user/project lists from disk via `loadAgentsFromDir` (overrides applied during this call, per Task 2).
   3. **Collect the full set of known names from the unfiltered lists** — this is the input to `detectUnknownAgentNames` and must include disabled names so that disabling a real agent does not produce a "references unknown agent" warning.
   4. Call `detectUnknownAgentNames(overlay, knownNames)` and concatenate with `overlay.warnings` to form `overlayWarnings`.
   5. Apply the disabled-filter from Task 3 to each list (builtin/user/project).
   6. Return.

   Add a new field to the return type:
   ```ts
   export function discoverAgentsAll(cwd: string): {
     builtin: AgentConfig[];
     user: AgentConfig[];
     project: AgentConfig[];
     chains: ChainConfig[];
     userDir: string;
     projectDir: string | null;
     overlayWarnings: string[];  // NEW
   }
   ```

3. Do **not** add `overlayWarnings` to `discoverAgents` (the narrower function); leave its shape unchanged to avoid touching unrelated callers. Existing call sites of `discoverAgentsAll` need a small audit:

**Steps:**
- [ ] **Audit (already done — record in commit message):** running `grep -rn "discoverAgentsAll" --include="*.ts"` from the repo root yields these callers:
  - `agent-management.ts` lines 71, 77, 88, 94, 105, 327, 375, 446, 505 — all use either object destructuring (`const d = discoverAgentsAll(cwd); d.builtin`) or chained access (`.chains.filter(...)`); adding a new field is safe.
  - `slash-commands.ts:248`: `const agentData = { ...discoverAgentsAll(ctx.cwd), cwd: ctx.cwd };` — **this is a behavior change**, not just a type-check issue. The new `overlayWarnings` field will now be passed into `AgentManagerComponent` (constructed at `slash-commands.ts:257`) inside `agentData`. Verify by reading `agent-manager.ts` that the component accepts unknown extra fields without crashing (TypeScript structural typing usually permits this, but a runtime `Object.keys` iteration could regress). If the component does iterate keys, this task must also pluck `overlayWarnings` out of the spread before passing to the component, e.g.:
    ```ts
    const { overlayWarnings, ...rest } = discoverAgentsAll(ctx.cwd);
    const agentData = { ...rest, cwd: ctx.cwd };
    // surface overlayWarnings via ctx.ui.notify(...) if non-empty
    ```
    Pick whichever is appropriate after reading `agent-manager.ts`. Document the choice in the commit message.
  - `path-resolution.test.ts` (line 6): imports `discoverAgentsAll` — must be checked for the same env-var harness gap as the unit tests in Task 2. Add the harness if it touches user-scope discovery.
  - `agents.ts:246`: the definition itself — modified by this task.
- [ ] Add new test cases to the `describe("subagents-config-overlay", ...)` block in `test/unit/subagents-config-overlay.test.ts`:
  - `agents.foo` references a name that does not exist anywhere → `discoverAgentsAll(cwd).overlayWarnings` contains a string mentioning `foo`.
  - `disabled: ["nonexistent"]` → `overlayWarnings` contains a string mentioning `nonexistent`.
  - `disabled: ["scout"]` (where `scout` is a real built-in) → `overlayWarnings` does NOT contain a string mentioning `scout` (the name is known).
  - `agents.scout` override + `disabled: ["scout"]` → `scout` is filtered out **and** no "unknown agent" warning is emitted (verifies edge case 2 + the order-of-operations fix above).
  - Invalid JSON in the project file → `overlayWarnings` contains a string mentioning the file path.
- [ ] Add a `describe("detectUnknownAgentNames", ...)` block in `test/unit/subagents-config.test.ts` for the pure-function unit test (no fs).
- [ ] Run `npm run test:unit -- --test-name-pattern="subagents-config"`
  - Did the new tests fail? If they passed unexpectedly, stop and investigate why.
- [ ] Implement `detectUnknownAgentNames` in `subagents-config.ts`.
- [ ] Update `discoverAgentsAll` in `agents.ts` to compute and return `overlayWarnings`.
- [ ] Run `npm run test:unit`
  - Did all tests pass? Pay particular attention to any test that destructures `discoverAgentsAll`'s return value — it should still work because TypeScript struct widening is permissive on returns. If not, fix and re-run.
- [ ] Commit with message: `feat(agents): surface subagents.json overlay warnings via discoverAgentsAll`

**Acceptance criteria:**
- [ ] `discoverAgentsAll` returns an `overlayWarnings: string[]` field.
- [ ] Unknown names in `agents` and `disabled` produce warnings; known names do not.
- [ ] All callers of `discoverAgentsAll` still type-check and run.
- [ ] `npm run test:unit` is fully green.

---

### Task 5: Document `subagents.json` in README and add a sample file

**Context:**
The feature is functionally complete after Tasks 1–4. The final task makes it discoverable: a new section in `README.md` next to the existing skills/settings.json discussion (around `README.md:396-399`), plus a commented sample file the user can copy. Without docs, no one will find the new config.

**Files:**
- Modify: `README.md`
- Create: `docs/examples/subagents.json.sample`

**What to implement:**

1. In `README.md`, locate the section that documents project/user settings (search for the line `User settings:` near line 399). Immediately after that section, add a new top-level section:

   ````markdown
   ## Configuring agents with `subagents.json`

   You can override built-in agent fields and disable agents from loading
   without editing the bundled `.md` files. pi-subagents reads two files:

   - **Project**: `.pi/subagents.json` (relative to your working directory)
   - **User**: `~/.pi/agent/subagents.json`

   Project values take precedence over user values, field-by-field.
   The `disabled` lists from both files are unioned.

   ### Example

   ```jsonc
   {
     "agents": {
       "scout": {
         "model": "anthropic/claude-opus-4-6",
         "thinking": "high"
       },
       "reviewer": {
         "description": "Custom reviewer for the auth subsystem",
         "skills": ["code-review", "security-audit"]
       }
     },
     "disabled": ["delegate", "worker"]
   }
   ```

   ### Overridable fields

   `description`, `model`, `thinking`, `tools`, `skills`, `extensions`,
   `output`, `defaultReads`, `defaultProgress`, `interactive`,
   `maxSubagentDepth`.

   The agent's `name`, `systemPrompt` (body), and internal metadata
   cannot be overridden. To replace the system prompt, write a real
   agent file under `~/.agents/` or `<project>/.agents/` (existing
   override path).

   ### Disabling agents

   Names in `disabled` are skipped entirely — they will not appear in
   `/agents list`, are not invokable from chains or directly, and produce
   the standard "unknown agent" error if referenced.

   Validation problems (unknown fields, invalid JSON, references to
   non-existent agents) are surfaced as warnings via `discoverAgentsAll`,
   visible in agent-management output.
   ````

2. Create `docs/examples/subagents.example.jsonc` with the same example block above, plus inline `// ...` JSONC comments documenting each field. Users can copy it into place and edit. The `.jsonc` extension is intentional so editors apply JSON-with-comments syntax highlighting. This file is intentionally outside the `files` array in `package.json` — it ships only via the GitHub repo, not the npm package, to avoid bloating the published tarball.

**Steps:**
- [ ] Read the current `README.md` section around line 396 to confirm the exact heading style and surrounding context.
- [ ] Edit `README.md` to insert the new section verbatim from the spec above, adjusting heading level if needed to match neighbouring sections.
- [ ] Create `docs/examples/subagents.example.jsonc` with the JSONC example.
- [ ] Run `npm run test:unit` one more time as a sanity check — no code changes, but it confirms the docs commit doesn't break anything.
- [ ] Commit with message: `docs(subagents-config): document subagents.json overlay`

**Acceptance criteria:**
- [ ] `README.md` has a new "Configuring agents with `subagents.json`" section.
- [ ] `docs/examples/subagents.json.sample` exists and is valid JSONC.
- [ ] No code changes in this commit.
- [ ] `npm run test:unit` is fully green.

---

## Out of scope / future work

- **Chain overrides and chain disabling.** Built-in chains do not currently exist (only user/project chains are loaded — see `agents.ts:264-268`). When/if built-in chains are added, extend the schema with `chains.<name>` and `chainsDisabled` fields following the same pattern.
- **Known sharp edge: disabled agents referenced by existing chains.** If a user disables an agent (e.g. `worker`) that is referenced by a chain step, the chain will produce the existing "unknown agent" error at runtime when executed. This is **intentional** for v1: it surfaces the inconsistency loudly rather than silently skipping steps. Do not add a "fix" for this without an explicit ask — the right resolution is for the user to either re-enable the agent or remove the chain reference.
- **`extraFields` overrides.** Frontmatter passthrough fields are kept in `agent.extraFields`. Overriding them via `subagents.json` is straightforward (add a 12th allowlist entry) but unmotivated until someone asks for it.
- **Per-environment overlays** (e.g. `subagents.dev.json`). Out of scope; users can use shell aliases or symlinks for now.
- **CLI command** to manage `subagents.json` interactively (analogous to `agent-management.ts`). Out of scope for v1; the file is small enough to edit by hand and the existing agent-management `update` action serves a different (destructive) purpose.
