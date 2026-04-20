# KACE-Extended-Rules-Sidecar

**JVM rule-engine sidecar for the KACE multi-engine facade.**

A small Java 21 + Spring Boot 3 HTTP service that hosts JVM-native rule engines (Drools in v0; Easy Rules / OpenL Tablets later). Invoked by `KACECommerceEngine` over HTTP. Its output slots into the same consensus ranker as the in-process TypeScript engines, so the ranker gets both a Node-side and JVM-side vote.

---

## Placeholders used in this README

| Placeholder | Description | Example |
| --- | --- | --- |
| `<GITHUB_HANDLE>` | GitHub account the repos live under | `your-github-handle` |
| `<KACE_ENGINE_URL>` | URL of the KACECommerceEngine caller | `http://localhost:8080` in dev |
| `<SIDECAR_URL>` | URL of this sidecar service | `http://localhost:8081` in dev |

---

## What does KACE mean?

**KACE** is the umbrella brand for this 4-repo project. Treat it as a standalone acronym (like IKEA or NASA) — don't re-expand it in every sentence. Historically the letters came from **K**ishore **A**pps **C**ommerce **E**ngine; today it's the suite's brand name.

**The 4 repos:**

| Repo | Role |
| --- | --- |
| [KACECommerceEngine](https://github.com/<GITHUB_HANDLE>/KACECommerceEngine) | TS + Fastify middleware. Rule facade, rewards, Shopify BFF, hand-rolled SessionStorage. **The brain.** |
| [KACE-PromptKart](https://github.com/<GITHUB_HANDLE>/KACE-PromptKart) | Hydrogen public storefront — AI prompt packs. |
| [KACE-StudyDesk](https://github.com/<GITHUB_HANDLE>/KACE-StudyDesk) | Hydrogen public storefront — micro-courses. |
| [**KACE-Extended-Rules-Sidecar**](https://github.com/<GITHUB_HANDLE>/KACE-Extended-Rules-Sidecar) *(this repo)* | Java + Spring Boot + Drools JVM sidecar. |

---

## Architecture

### Why a separate sidecar?

KACECommerceEngine runs on Node.js. **JVM-native rule engines cannot run in Node.** To include any JVM engine (Drools, Easy Rules, OpenL Tablets, …) in the multi-engine rule facade, it must live in its own JVM process. This sidecar is that process.

### Where this sidecar sits

```
 ┌───────────────────────────────────────────────────────┐
 │  KACECommerceEngine  (Fastify, TS, <KACE_ENGINE_URL>) │
 │                                                       │
 │  ┌─────────────────────────────────────────────────┐  │
 │  │ RuleEngineFacade (Mode-A consensus ranker)      │  │
 │  └──┬──────┬──────────┬──────────────────┬─────────┘  │
 │     │      │          │                  │            │
 │     ▼      ▼          ▼                  ▼            │
 │  json-   nools-    rete-next-     ExtendedRulesSidecar│
 │  rules   adapter   adapter        Adapter (HTTP)      │
 └────────────────────────────────────┬──────────────────┘
                                      │ POST /evaluate/drools
                                      │ POST /evaluate/easy-rules (future)
                                      │ POST /evaluate/openl (future)
                                      ▼
 ┌───────────────────────────────────────────────────────┐
 │  KACE-Extended-Rules-Sidecar (Spring Boot, <SIDECAR_URL>)
 │                                                       │
 │  api/controller/    EvaluateController                │
 │  service/ + impl/   EvaluateServiceImpl               │
 │    └── workflows/EvaluateWorkflow/                    │
 │        tasks/       CompileRuleSetTask (workflow-specific) │
 │  tasks/             reusable: ValidateRequestTask,     │
 │                     HashRuleSetTask, TranslateRulesTask,│
 │                     InvokeEngineTask, BuildResponseTask│
 │                                                       │
 │  core/engine/                                         │
 │    drools/         RuleTranslator, DroolsEngine, KieBaseCompiler
 │    easyrules/      (future)                           │
 │    openl/          (future)                           │
 │  dao/              KieBaseCacheDao + in-memory impl    │
 └───────────────────────────────────────────────────────┘
                                      │
                                      ▼
                                (no external deps; no DB, no Shopify)
```

### Properties

- **Stateless.** No database, no Redis, no Mongo. Rules + context arrive on every request.
- **In-memory compiled-artifact cache.** Compiling a rule set into a Drools `KieBase` takes ~50-200 ms. We cache compiled `KieBase` instances keyed by `ruleSetHash` (SHA-256 of the sorted rule JSON sent by KACE). Default LRU capacity 32.
- **No Shopify calls.** KACECommerceEngine owns all Shopify I/O. This sidecar only receives already-normalized rule JSON + evaluation context.
- **Not a hard dependency of KACE.** If the sidecar is down, the TypeScript rule engines in KACE still run; the consensus ranker simply reports one fewer participant.
- **Stateless `KieSession` per request.** No shared Drools session across requests — horizontally scalable without sticky sessions.

---

## The `POST /evaluate/{engineId}` contract

One endpoint per engine id. Request/response shape is **identical across engines** — the KACE ranker can treat every result uniformly.

### Request

```jsonc
POST /evaluate/drools
Content-Type: application/json

{
  "engineRequestId": "req_2c1b8a...",   // correlation id from KACE
  "tenant":          "PROMPTKART",      // PROMPTKART | STUDYDESK
  "trigger":         "cart.updated",
  "ruleSetHash":     "sha256:...",      // enables compiled-artifact cache hit
  "rules":   [ { id, priority, conditions, actions }, ... ],
  "context": { "cart": {...}, "customer": {...}, "products": [...] }
}
```

### Response

```jsonc
200 OK
{
  "engineRequestId": "req_2c1b8a...",
  "engine":          "drools",
  "engineVersion":   "9.x.x",
  "actions": [
    { "type": "APPLY_DISCOUNT",
      "params": {...},
      "firedByRuleId": "..." }
  ],
  "meta": {
    "rulesCount":       17,
    "rulesFired":       1,
    "durationMs":       8,
    "artifactCacheHit": true,
    "session":          "stateless"
  }
}
```

### Error semantics

- `400` — malformed request or bad rule translation. KACE logs + excludes this engine from the consensus.
- `404 /evaluate/{unknownEngineId}` — KACE sent a wrong id; logged loudly.
- `5xx` — internal JVM error. KACE treats the same as `400`: skip this engine, rank the remaining engines' outputs.

**The ranker MUST NOT block on this sidecar.** KACE enforces a per-engine timeout (default 500 ms); slow or erroring engines are dropped from the consensus.

---

## Rule translation (Drools path)

KACE's normalized rule JSON → Drools `.drl` via `core/engine/drools/RuleTranslator.java`.

### Example

Input (normalized KACE JSON):
```json
{ "id":"bulk-buyer-15pct", "priority":10,
  "conditions":{"all":[
    {"fact":"customer.purchaseCount30d","operator":">=","value":3},
    {"fact":"cart.total","operator":">=","value":50}
  ]},
  "actions":[{"type":"APPLY_DISCOUNT","params":{"percent":15,"label":"Bulk Buyer"}}] }
```

Output `.drl`:
```drl
rule "bulk-buyer-15pct"
  salience 10
when
  $cust : Customer( purchaseCount30d >= 3 )
  $cart : Cart( total >= 50 )
then
  actions.add(new Action("APPLY_DISCOUNT",
              Map.of("percent", 15, "label", "Bulk Buyer"),
              "bulk-buyer-15pct"));
end
```

Translator supports:

- Condition trees: `all` / `any` / `not`.
- Operators: `==`, `!=`, `<`, `<=`, `>`, `>=`, `in`, `contains`.
- Fact-path dereferencing: `customer.purchaseCount30d` → `Customer.purchaseCount30d`.
- Custom operators register via `core/engine/drools/OperatorRegistry.java`.

For v1 engines (Easy Rules / OpenL), each gets its own translator in `core/engine/<engine>/`.

---

## Compiled-artifact cache (`KieBaseCacheDao`)

- **Cache key:** `ruleSetHash` (sent by KACE — SHA-256 of the sorted rule JSON).
- **Cache value:** compiled `KieBase` + last-use timestamp.
- **Policy:** LRU, bounded capacity (default 32; configurable via `AppProperties`).
- **Invalidation:** none. KACE sends the full rule set + hash on every request; rule changes → hash changes → cache miss → recompile → evict old.

Other future JVM engines use the same `KieBaseCacheDao` interface for their compiled artifacts, keeping the pattern consistent across engines.

---

## Consensus contract with KACE (Mode A ranker)

KACE's `RuleEngineFacade` runs all enabled engines in parallel on the same `(rules, context)`. The consensus ranker (Mode A) then:

1. Awaits all engines up to the per-engine timeout (default 500 ms).
2. For each successful engine, canonicalizes the `actions` (stable sort by `type`, stable-stringified `params`).
3. Computes `engineActionHash = sha256(canonical(actions))`.
4. Groups engines by hash.
5. If all engines are in one group → `consensus=true`; return that action set.
6. Else → `consensus=false`; return the action set from the **largest group**. Tie-break by configured engine priority (default: `json-rules-engine > nools > rete-next > drools`).
7. Log the disagreement matrix — engine-by-engine action diffs — whenever `consensus=false`.

This sidecar's output is one of N inputs to the ranker. Accurate matters more than fast; be fast *too* when possible.

---

## Planned directory layout

```
src/main/java/com/kishoreapps/kace/extrulessidecar/
├── KaceExtRulesSidecarApplication.java          Spring Boot entry
│
├── api/
│   ├── controller/
│   │   ├── EvaluateController.java              POST /evaluate/{engineId}
│   │   ├── HealthController.java
│   │   └── AdminController.java                 /admin/engines (list enabled)
│   └── advice/GlobalExceptionHandler.java
│
├── service/                                      interfaces + impl/
│   ├── EvaluateService.java
│   └── impl/
│       ├── EvaluateServiceImpl.java              orchestrates EvaluateWorkflow
│       └── workflows/
│           ├── Workflow.java                     (interface)
│           └── EvaluateWorkflow/
│               ├── EvaluateWorkflow.java
│               └── tasks/CompileRuleSetTask.java  workflow-specific
│
├── tasks/                                        reusable tasks
│   ├── Task.java
│   ├── ValidateRequestTask.java
│   ├── HashRuleSetTask.java
│   ├── TranslateRulesTask.java                   dispatches to selected engine's translator
│   ├── InvokeEngineTask.java                     runs RuleEngine.evaluate()
│   └── BuildResponseTask.java
│
├── core/                                         framework-free, no Spring imports
│   ├── engine/
│   │   ├── RuleEngine.java                       interface (same shape as KACE's TS adapter)
│   │   ├── drools/
│   │   │   ├── DroolsEngine.java                 RuleEngine impl
│   │   │   ├── RuleTranslator.java               normalized JSON rule → .drl
│   │   │   ├── ContextMapper.java                JSON context → Drools facts
│   │   │   └── KieBaseCompiler.java
│   │   └── (future) easyrules/, openl/
│   ├── workflow/                                 Workflow, Task, WorkflowContext (framework-free)
│   └── domain/                                   Rule, Action, Context, ActionAccumulator
│
├── dao/                                          data access — keep the seam even if in-memory
│   ├── KieBaseCacheDao.java
│   └── impl/InMemoryKieBaseCacheDao.java
│
├── dtos/
│   ├── request/  EvaluateRequest, RuleDto, ContextDto
│   └── response/ EvaluateResponse, ActionDto, MetaDto
│
├── constants/                                    EngineIds, ErrorCodes, CacheConstants
├── utils/                                        JsonUtils, HashUtils
└── config/                                       AppProperties, ObjectMapperConfig, OpenApiConfig, EngineRegistryConfig
```

Mirror image of `KACECommerceEngine`'s `api/service/core/dao/dtos/constants/utils/config + tasks/workflow` layout, so the two backend services read identically across languages.

---

## Tech stack

| Layer | Choice |
| --- | --- |
| Language | Java 21 (LTS) |
| Build | Gradle 8 (Kotlin DSL) |
| Framework | Spring Boot 3.x (Web, Actuator, Validation) |
| Rule engine (v0) | Drools 9.x |
| Rule engines (v1+) | Easy Rules, OpenL Tablets |
| Logging | Logback + structured JSON layout |
| Metrics | Micrometer → Prometheus |
| Tracing | OpenTelemetry Java agent |
| OpenAPI | springdoc-openapi |
| Local run | `./gradlew bootRun` |
| Container (future) | Eclipse Temurin JRE 21 |
| Port | `8081` |

---

## Status

⏳ **Pre-scaffold.** Repo currently holds this README, an MIT license, and a `.gitignore`. Spring Boot scaffolding + Drools integration ships in a later phase.

---

## License

MIT — see [`LICENSE`](LICENSE).

---

## Related planning docs

Full design docs for this sidecar (and the full KACE suite) live in the parent planning folder — architecture, per-service READMEs, diagrams, development-plan spreadsheet.
