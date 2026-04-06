---
name: sonic-architecture
description: Design and implement SONiC switch software features using its containerized, Redis/SAI-driven architecture. Use this skill when adding new SWSS orchagents/mgrds, modeling Redis DB schemas (CONFIG_DB/APPL_DB/STATE_DB/ASIC_DB/COUNTERS_DB), wiring SAI via sairedis/syncd, extending CLI/management frameworks, and validating on the VS platform.
---

# SONiC Architecture Patterns

Build robust SONiC features that integrate cleanly with containers, Redis databases, and the SAI/ASIC pipeline.

Given: a SONiC feature or enhancement target.
Produces: clear data flows, DB schemas, container boundaries, and production‑grade code paths from CONFIG_DB/APPL_DB to SAI/ASIC with tests.

## When to Use This Skill

- Adding or changing L2/L3 features inside `sonic-swss` (orchagent/mgrd/syncd flows)
- Introducing/altering Redis DB tables in CONFIG_DB/APPL_DB/STATE_DB/COUNTERS_DB
- Extending the CLI (`sonic-utilities`) and/or Management Framework (RESTCONF/gNMI)
- Bringing up features on the VS platform and writing unit/E2E tests
- Debugging data-path issues across Redis/SAI/ASIC and container boundaries

## Core Concepts

### Containers & Boundaries

- `swss`: Switch State Service (orchagent, sync daemons: fpmsyncd, neighsyncd, teamsyncd, etc.)
- `syncd`: Vendor SAI + `sairedis` bridge that programs the ASIC
- `bgp`: FRR/Quagga routing stack; `fpmsyncd` publishes FIB to APPL_DB
- `database`: Redis server hosting SONiC’s logical DBs
- Other common containers: `lldp`, `snmp`, `dhcp-relay`, `teamd`, `pmon`, `telemetry`

### Redis Databases (logical)

- `CONFIG_DB`: persistent desired config; source of truth for managers
- `APPL_DB` (a.k.a. APP DB): application‑generated operational intent (routes, neighbors, next‑hops, etc.)
- `STATE_DB`: cross‑module readiness/operational flags
- `ASIC_DB`: SAI/ASIC‑friendly representation consumed by `syncd`
- `COUNTERS_DB`: statistics collected via Flex Counters

### Southbound Pipeline (simplified)

CLI/RESTCONF/gNMI → Config validation (CVL) → `CONFIG_DB` → mgrd/daemon → `APPL_DB` → orchagent (swsscommon Consumer) → SAI via `sairedis` → `syncd` → ASIC

Routing example: FRR `zebra` → `fpmsyncd` → `APPL_DB` → orchagent → SAI/ASIC.

## Repository Map (where code lives)

- `sonic-buildimage`: image build system, container definitions, database config, VS target
- `sonic-swss`: orchagent, *syncd daemons (publishers to APPL_DB), cfgmgrd/mgrd utilities
- `sonic-sairedis`: `syncd` and Redis bridge for SAI; includes VS (virtual switch) libs
- `sonic-utilities`: CLI (`config`, `show`, helpers), utilities_common
- `sonic-mgmt-framework`: RESTCONF/gNMI servers, CVL, translib, YANG tooling
- `sonic-mgmt`: Ansible/pytest testbeds and E2E validation

## Development Workflow

1) Write the HLD
- Document tables in CONFIG_DB/APPL_DB/STATE_DB/COUNTERS_DB and SAI objects.
- Call out warm‑reboot, scale, and multi‑ASIC implications.

2) Model the data
- Define or extend Redis table schemas; keep keys/fields minimal and stable.
- Add/align YANG models (native or OpenConfig) for northbound APIs.

3) Implement managers and orchestrators
- `mgrd` (in `cfgmgr/` or feature daemon): watch `CONFIG_DB`, apply kernel/sysctl, publish to `APPL_DB`.
- `orchagent` (in `orchagent/`): consume `APPL_DB` tables, resolve dependencies via `STATE_DB`, and call SAI.
- Never write `ASIC_DB` directly; use SAI via `sairedis`.

4) Counters/telemetry
- Expose per‑object counters via Flex Counters; publish to `COUNTERS_DB`.

5) CLI & Management Framework
- `sonic-utilities`: add `config` and `show` commands; use `SonicV2Connector`/`swsscommon`.
- Mgmt Framework: update YANG, transformers, and translib handlers for RESTCONF/gNMI.

6) Build & test
- Unit tests in `sonic-swss/tests` (gtest) and module tests for new daemons.
- VS bring‑up for functional tests; E2E with `sonic-mgmt` pytest.

## Minimal Code Patterns

### A tiny orch skeleton (C++)
```cpp
// fooorch.h
#include "orch.h"
#include "dbconnector.h"
#include "consumerstatetable.h"

class FooOrch : public Orch
{
  public:
    FooOrch(swss::DBConnector* appDb, const std::string& table)
    : Orch({ { appDb, table } }) {}

  private:
    void doTask(Consumer& consumer) override;
};
```
```cpp
// fooorch.cpp
#include "fooorch.h"
#include "producerstatetable.h"
#include "logger.h"

void FooOrch::doTask(Consumer& consumer)
{
    auto& queue = consumer.m_toSync;
    while (!queue.empty()) {
        auto it = queue.begin();
        KeyOpFieldsValuesTuple t = it->second;
        const std::string &key = kfvKey(t);
        const std::string &op  = kfvOp(t);
        auto fvs = kfvFieldsValues(t);

        if (op == SET_COMMAND) {
            // TODO: parse fvs, call SAI create/set via appropriate SAI object APIs
        } else if (op == DEL_COMMAND) {
            // TODO: call SAI remove
        }
        queue.erase(it);
    }
}
```
Wire‑up: register `FooOrch` in `orchdaemon.cpp` after creating the `APP_DB` connector.

### Config manager pattern (C++)
```cpp
// barcfgmgrd.cpp
#include "dbconnector.h"
#include "consumerstatetable.h"
#include "producerstatetable.h"

int main() {
    swss::DBConnector cfg("CONFIG_DB", 0);
    swss::DBConnector app("APPL_DB", 0);

    swss::ConsumerStateTable cfgTbl(&cfg, "BAR_TABLE");
    swss::ProducerStateTable appTbl(&app, "BAR_APP_TABLE");

    while (true) {
        swss::Selectable *sel;
        std::set<swss::Selectable*> selectables = { &cfgTbl };
        swss::Select s;
        for (auto *ss : selectables) s.addSelectable(ss);
        s.select(&sel);
        swss::KeyOpFieldsValuesTuple kco;
        if (cfgTbl.pop(kco)) {
            // Translate CONFIG_DB -> APPL_DB
            appTbl.set(kfvKey(kco), kfvFieldsValues(kco));
        }
    }
}
```

## DB Design Guardrails

- Don’t bypass SAI: update hardware only through orchagent→SAI→`syncd`.
- Keep Redis keys stable; favor additive fields for backward compatibility.
- Prefer notifications and incremental updates over container restarts.
- Use `STATE_DB` to guard ordering and readiness between modules.
- Reuse existing tables when possible; avoid near‑duplicate schemas.

## Build & Test Quickstarts

### Build a VS image
- Prereqs: Docker, Debian/Ubuntu build deps per `sonic-buildimage`.
- Steps (from a fresh clone):
  - `git clone https://github.com/sonic-net/sonic-buildimage && cd sonic-buildimage`
  - `make init`  # pulls toolchains/containers
  - `make configure PLATFORM=vs`
  - `make target/sonic-vs.img.gz -j$(nproc)`

### Run swss unit tests
- In `sonic-swss`: `./autogen.sh && ./configure && make -j$(nproc)`
- Then: `make check` or `tests/run` helpers if present.

### Exercise E2E with sonic-mgmt (pytest)
- Clone `sonic-net/sonic-mgmt`, set up testbed inventory, then:
  - `cd tests && pytest -m sanity -vv`  (or select feature‑specific marks)

## Debugging Tips

- Inspect Redis: `redis-cli -n <db_id> keys *` then `hgetall <key>`
- Increase logs: `swssloglevel -l DEBUG` (orchagent); check `/var/log/syslog`
- Verify SAI path: `saiplayer`/`saidump` in `sonic-sairedis` for replay/debug
- Confirm container health: `docker ps`, `systemctl status swss`, `show services`
- Counters: `show interface counters -a`, verify `COUNTERS_DB` updates

## Warm Restart & Scale

- Use warm‑restart infrastructure (`warmrestart/`) to preserve state across restarts.
- Size Redis and Flex Counters thoughtfully; batch SAI calls to reduce control‑plane load.

## Multi‑ASIC & Chassis Notes

- Follow multi‑ASIC guidelines: per‑ASIC namespaces, proper DB instance targeting, and VOQ/scale considerations if applicable.

## Checklist for a New Feature

- [ ] HLD merged and DB schemas reviewed
- [ ] APPL_DB producer(s) or CONFIG_DB→APPL_DB translation implemented
- [ ] Orchagent consumer + SAI calls implemented with error handling
- [ ] Flex Counters wired; `show`/telemetry outputs added
- [ ] CLI/RESTCONF/gNMI paths implemented and YANG validated (CVL)
- [ ] VS image build success and swss unit tests passing
- [ ] sonic-mgmt E2E tests added and green
- [ ] Warm reboot behavior verified; documentation updated

## Related Skills

- `architecture-patterns` — Complementary layering principles for clean module boundaries
- `test-harness` — Use with `sonic-mgmt` to run functional suites


## System Design Core Concepts

The following principles guide robust SONiC feature design across containers, Redis databases, and the SAI/ASIC pipeline.

- Clear owners and boundaries
  - Each daemon owns specific tables and responsibilities; avoid overlapping writes.
  - Northbound (CLI/Mgmt Framework) → CONFIG_DB; Managers translate to APPL_DB; Orchestrators translate APPL_DB → SAI.
- Declarative, level‑driven control loops
  - Treat Redis tables as desired/observed state, not imperative APIs.
  - Make loops idempotent; reconcile to target state on every event and on timer.
- Data model first
  - Design CONFIG_DB/APPL_DB/STATE_DB/COUNTERS_DB keys, fields, and lifecycles before writing code.
  - Keep keys stable and minimal; version schemas when breaking.
- Asynchrony and backpressure
  - Expect eventual consistency; model dependencies explicitly (STATE_DB readiness, reference counts, warm‑start restore).
  - Avoid blocking in orch loops; use retry queues and conservative batching.
- Failure domains and durability
  - Bound blast radius by container; persist only what’s necessary in CONFIG_DB.
  - Design warm/fast reboot paths (reconcile from ASIC/COUNTERS_DB or saved state as applicable).
- Multi‑ASIC and scale
  - Partition responsibilities per ASIC; avoid cross‑ASIC coupling in table design.
  - Validate O(N·logN) or better behaviors under large route/neighbor scales and churn.
- Observability as a contract
  - Emit counters via Flex Counters; expose health with STATE_DB.
  - Provide actionable logs with stable keys and object identifiers.
- Compatibility and upgrades
  - Maintain backward compatibility for DB schemas and CLI; introduce new fields as additive when possible.
  - Include explicit migration steps in HLDs and tests.

### End‑to‑End Flow (typical)

CLI/RESTCONF/gNMI → CVL validation → CONFIG_DB → manager/mgrd → APPL_DB → orchagent (consumers) → sairedis → syncd → SAI → ASIC → COUNTERS_DB/telemetry.

### Design Checklist (system level)

- Define DB schema and ownership (who writes/reads each table?)
- Specify state machine and reconciliation strategy (events, timers, retries)
- Identify dependencies (interfaces, neighbors, ACLs, QoS, etc.) and gate via STATE_DB
- Plan warm/fast reboot behavior and recovery sources
- Model counters/telemetry and alarms; document show commands
- Validate multi‑ASIC behavior and scale limits; include VS/E2E tests

## References

- SONiC main org and docs
  - SONiC GitHub (sonic‑net): https://github.com/sonic-net
  - SONiC Wiki (Architecture, HLDs, guides): https://github.com/sonic-net/SONiC/wiki
- Core repositories
  - sonic‑buildimage (image, containers, YANG models): https://github.com/sonic-net/sonic-buildimage
  - sonic‑swss (orchagent, swsscommon, sync daemons): https://github.com/sonic-net/sonic-swss
  - sonic‑sairedis and syncd: https://github.com/sonic-net/sonic-sairedis
  - sonic‑utilities (CLI): https://github.com/sonic-net/sonic-utilities
  - sonic‑mgmt‑framework (RESTCONF/gNMI, CVL, translib): https://github.com/sonic-net/sonic-mgmt-framework
  - sonic‑mgmt (E2E tests): https://github.com/sonic-net/sonic-mgmt
- Specifications and protocols
  - Switch Abstraction Interface (SAI) spec (OCP): https://github.com/opencomputeproject/SAI
  - FRRouting FPM (kernel route programming): https://docs.frrouting.org/en/latest/fpm.html
  - Redis (data structures/patterns used by SONiC): https://redis.io/docs/latest/
- Helpful deep dives and design docs
  - Warm reboot design: see SONiC Wiki → High Level Designs
  - Flex Counters architecture in sonic‑swss
  - VS platform usage and tests in sonic‑sairedis and sonic‑mgmt


### Local Snapshots (as of 2026-04-06)

- [references/SONiC-Wiki-Home.md](references/SONiC-Wiki-Home.md) — SONiC wiki Home
- [references/SONiC-Architecture.md](references/SONiC-Architecture.md) — SONiC wiki Architecture
- [references/SONiC-Design-Specs.md](references/SONiC-Design-Specs.md) — SONiC wiki Design Specs index
- [references/SONiC-Configuration.md](references/SONiC-Configuration.md) — Configuration guide
- [references/SONiC-Developing-Guide.md](references/SONiC-Developing-Guide.md) — Developing guide
- [references/SONiC-Troubleshooting-Guide.md](references/SONiC-Troubleshooting-Guide.md) — Troubleshooting guide
- [references/SONiC-Warm-Reboot.md](references/SONiC-Warm-Reboot.md) — Warm Reboot HLD (repo doc)

Note: Local copies are convenience snapshots. Always verify against upstream for the latest revisions.
