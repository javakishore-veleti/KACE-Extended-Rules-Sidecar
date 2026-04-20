# KACE-Extended-Rules-Sidecar

**JVM rule-engine sidecar for the KACE multi-engine facade. Hosts Drools (v0) + future engines.**

---

## What does KACE mean?

**KACE** is the umbrella brand for this 4-repo project. It is a standalone acronym
(treat it like IKEA or NASA — don't re-expand it in every sentence). Historically
the letters came from **K**ishore **A**pps **C**ommerce **E**ngine, but today KACE
is the name of a 4-service suite built on Shopify by
[**Kishore Veleti**](https://github.com/javakishore-veleti) — Shopify Partner org
*Kishore Applications* (Partner ID `4868609`, org id `214691442`).

**The 4 repos in the KACE suite:**

| Repo | Role |
| --- | --- |
| [KACECommerceEngine](https://github.com/javakishore-veleti/KACECommerceEngine) | TypeScript + Fastify middleware — rule engine facade, rewards, Shopify BFF, hand-rolled SessionStorage. **The brain.** |
| [KACE-PromptKart](https://github.com/javakishore-veleti/KACE-PromptKart) | Hydrogen public storefront selling AI prompt packs. |
| [KACE-StudyDesk](https://github.com/javakishore-veleti/KACE-StudyDesk) | Hydrogen public storefront selling micro-courses. |
| [**KACE-Extended-Rules-Sidecar**](https://github.com/javakishore-veleti/KACE-Extended-Rules-Sidecar) *(this repo)* | Spring Boot + Drools JVM sidecar. |

---

## What is `KACE-Extended-Rules-Sidecar` specifically?

A small **Java / Spring Boot 3** HTTP service that exists because
`KACECommerceEngine` is a TypeScript service and **JVM-native rule engines cannot
run in Node.js**.

To include any JVM rule engine (Drools, Easy Rules, OpenL Tablets, MVEL-based
custom engines, ...) in `KACECommerceEngine`'s multi-engine facade, those engines
have to live in their own JVM process. `KACECommerceEngine` calls this sidecar over
HTTP at `:8081`, and the result slots into the same consensus ranker pipeline as the
TypeScript engines.

### Why "Extended Rules Sidecar" instead of "Drools Sidecar"?

- **Future-proof.** Adding another JVM engine later (e.g. Easy Rules for simple
  annotation-based rules) is "add another adapter under the same hood", not
  "create a new sidecar service".
- **Honest naming.** This sidecar is the KACE project's "any-JVM-rule-engine"
  seam. Pinning the name to Drools would hide that.
- **Cross-language comparison is the point.** The reason Drools lives here at all
  is to give the consensus ranker a JVM voice, not because Drools is sacred. Any
  other JVM engine that answers the same `evaluate` contract would work.

### Properties

- **Single endpoint group:** `POST /evaluate/{engineId}` (e.g. `/evaluate/drools`).
- **Stateless.** No DB, no Redis, no Mongo. Rules + context come in on each
  request; compiled rule artifacts are cached in-memory (`KieBase` LRU).
- **No Shopify calls.** `KACECommerceEngine` owns all Shopify I/O; this sidecar
  only receives the already-normalized rule JSON + evaluation context.
- **Not a hard dependency of `KACECommerceEngine`.** If this service is down, the
  TS engines still run; the consensus ranker just reports one fewer participant.

### Status

⏳ **Pre-scaffold.** Repo has a placeholder README, MIT license, and `.gitignore`.
Actual Spring Boot scaffolding + Drools integration ships in a later phase.

---

## Planned tech stack

| Layer | Choice |
| --- | --- |
| Language | Java 21 (LTS) |
| Build | Gradle 8 (Kotlin DSL) |
| Framework | Spring Boot 3.x (Web, Actuator, Validation) |
| JVM rule engines (v0) | Drools 9.x |
| JVM rule engines (v1+) | Easy Rules, OpenL Tablets |
| Logging | Logback + JSON layout |
| Metrics | Micrometer → Prometheus |
| Tracing | OpenTelemetry Java agent |
| OpenAPI | springdoc-openapi |
| Local run | `./gradlew bootRun` on port `:8081` |
| Container (future) | Eclipse Temurin JRE 21 |

---

## Planned directory layout

```
src/main/java/com/kishoreapps/kace/extrulessidecar/
├── api/                   controllers + advice (EvaluateController, HealthController)
├── service/               service interfaces + impl/
│   ├── EvaluateService.java
│   └── impl/
│       ├── EvaluateServiceImpl.java
│       └── workflows/EvaluateWorkflow/
├── tasks/                 reusable tasks (ValidateRequest, HashRuleSet, TranslateRules, InvokeEngine, BuildResponse)
├── core/                  framework-free — no Spring imports
│   ├── engine/
│   │   ├── RuleEngine.java          (interface — same shape as KACE's TS adapter)
│   │   ├── drools/
│   │   │   ├── DroolsEngine.java
│   │   │   ├── RuleTranslator.java  (KACE JSON → .drl)
│   │   │   ├── ContextMapper.java
│   │   │   └── KieBaseCompiler.java
│   │   └── (future) easyrules/, openl/
│   ├── workflow/          (Workflow, Task, WorkflowContext interfaces)
│   └── domain/            (Rule, Action, Context, ActionAccumulator)
├── dao/                   data access — even if in-memory, keep the seam
│   ├── KieBaseCacheDao.java
│   └── impl/InMemoryKieBaseCacheDao.java
├── dtos/                  request/ + response/
├── constants/
├── utils/
└── config/                AppProperties, OpenApiConfig, EngineRegistryConfig
```

Same `api/service/core/dao/dtos/constants/utils/config + tasks/workflow` layout as
`KACECommerceEngine` — so the two backend services look like mirror images across
languages.

---

## License

MIT — see [`LICENSE`](LICENSE).

---

## Related planning docs (not in this repo)

Full design lives in the parent planning folder (`Shopify_Middleware/`):

- `README_KACE-Extended-Rules-Sidecar.md` — full architecture of this sidecar
- `README_KACECommerceEngine.md` — the middleware that calls this sidecar
- `Development_Plan.xlsx` — phase-by-phase roadmap
- `diagrams/` — architecture + mindmap images
