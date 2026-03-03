# RK3588 AI Box V1 Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build an RK3588-based B/S AI box that supports 20-channel IPC analysis, main-stream preview relay, adaptive frame sampling, GAT1400 bidirectional integration, GB28181 access/cascade, layered UI, RBAC, and tenant/project isolation.

**Architecture:** Use a single-edge multi-service architecture. Ingest once per stream and fan out to preview relay and analysis pipeline. Keep preview path no-transcode. Centralize event model and protocol adapters. Enforce tenant/project isolation across API, storage, and protocol integration.

**Tech Stack:** RK3588 MPP/RGA/RKNN, Node.js or Go microservices, PostgreSQL, WebSocket, SIP/GB28181 stack, GAT1400 adapter, browser-based cockpit.

---

### Task 1: Repository Skeleton and Config Baseline

**Files:**
- Create: `docs/architecture/rk3588-ai-box-overview.md`
- Create: `configs/default/system.yaml`
- Create: `configs/default/storage.yaml`
- Create: `configs/default/scheduler.yaml`

**Step 1: Create architecture baseline document**

Write service boundaries, data flow, and ownership matrix.

**Step 2: Add storage policy config**

```yaml
media:
  high_watermark_percent: 85
  low_watermark_percent: 70
  cleanup_strategy: oldest_first
```

**Step 3: Add adaptive scheduler default config**

```yaml
scheduler:
  tick_seconds: 2
  fps:
    min: 2
    normal: 8
    max: 12
  boost:
    on_alarm_fps: 12
    duration_seconds: 20
```

**Step 4: Validate configuration schema**

Run: `node scripts/validate-config.js`  
Expected: `All config files valid`

---

### Task 2: Ingest Hub and Stream Fan-out

**Files:**
- Create: `src/ingest-hub-service/index.ts`
- Create: `src/ingest-hub-service/stream-registry.ts`
- Create: `src/ingest-hub-service/fanout.ts`
- Test: `tests/ingest-hub/stream-fanout.test.ts`

**Step 1: Write failing test for single-ingest multi-consumer**

```ts
it("fans out one source stream to preview and analysis consumers", async () => {
  const reg = new StreamRegistry();
  reg.attach("camera-001", "rtsp://demo");
  const out = reg.getConsumers("camera-001");
  expect(out).toEqual(["preview", "analysis"]);
});
```

**Step 2: Run test to verify it fails**

Run: `npm test -- tests/ingest-hub/stream-fanout.test.ts`  
Expected: FAIL with missing implementation.

**Step 3: Implement minimal stream registry and fan-out logic**

Ensure one upstream pull per camera and shared downstream subscriptions.

**Step 4: Run test to verify it passes**

Run: `npm test -- tests/ingest-hub/stream-fanout.test.ts`  
Expected: PASS

---

### Task 3: Preview Relay Without Transcode

**Files:**
- Create: `src/preview-relay-service/index.ts`
- Create: `src/preview-relay-service/transmux.ts`
- Create: `src/preview-relay-service/capability.ts`
- Test: `tests/preview-relay/no-transcode.test.ts`

**Step 1: Write failing no-transcode contract test**

```ts
it("does not call encoder in preview pipeline", async () => {
  const pipeline = buildPreviewPipeline();
  expect(pipeline.usesEncoder).toBe(false);
});
```

**Step 2: Run test to verify failure**

Run: `npm test -- tests/preview-relay/no-transcode.test.ts`  
Expected: FAIL

**Step 3: Implement remux/transmux only path**

Allow RTP/PS to browser-compatible container conversion only.

**Step 4: Add H.265 capability check gate**

If client capability is false, return compatibility error with remediation hint.

**Step 5: Run tests**

Run: `npm test -- tests/preview-relay/no-transcode.test.ts`  
Expected: PASS

---

### Task 4: Adaptive Analysis Scheduler

**Files:**
- Create: `src/analysis-pipeline-service/scheduler.ts`
- Create: `src/analysis-pipeline-service/metrics.ts`
- Create: `src/analysis-pipeline-service/priorities.ts`
- Test: `tests/analysis/scheduler.test.ts`

**Step 1: Write failing tests for fps downgrade/upgrade**

Cover normal, medium pressure, high pressure, and alarm boost.

**Step 2: Run failing tests**

Run: `npm test -- tests/analysis/scheduler.test.ts`  
Expected: FAIL

**Step 3: Implement scoring-based scheduler**

Inputs: npu util, queue depth, p95 latency, temperature, camera priority, alarm density.  
Output: per-camera `analyze_fps`.

**Step 4: Verify behavior on synthetic metrics**

Run: `npm run test:scheduler-fixtures`  
Expected: camera fps map matches policy table.

---

### Task 5: Unified Event Model and Media Recorder

**Files:**
- Create: `src/event-engine-service/event-model.ts`
- Create: `src/media-recorder-service/clip-writer.ts`
- Create: `src/media-recorder-service/snapshot-writer.ts`
- Test: `tests/events/unified-model.test.ts`

**Step 1: Define unified event schema**

Required fields: source_type, event_id, tenant_id, project_id, camera_id, target, bbox, score, start_ts, end_ts, snapshot_url, clip_url.

**Step 2: Add clip duration config**

Default clip duration is 20 seconds, with API-update support.

**Step 3: Implement recorder with storage watermark awareness**

Before writing clip, check current media usage and trigger cleanup when reaching 85%.

**Step 4: Run event and recorder tests**

Run: `npm test -- tests/events/unified-model.test.ts`  
Expected: PASS

---

### Task 6: Storage Watermark Cleanup and Audit

**Files:**
- Create: `src/storage-service/watermark-cleaner.ts`
- Create: `src/storage-service/media-index.ts`
- Create: `src/audit-service/audit-log.ts`
- Test: `tests/storage/watermark-cleaner.test.ts`

**Step 1: Write failing test for 85->70 cleanup**

```ts
it("cleans oldest media from 85% down to 70%", async () => {
  const result = await cleaner.cleanup({ used: 85, target: 70 });
  expect(result.finalUsed).toBeLessThanOrEqual(70);
});
```

**Step 2: Implement oldest-first cleanup algorithm**

Delete by media creation time and preserve metadata records.

**Step 3: Add audit events**

Record cleanup start, deleted items count, and completion watermark.

**Step 4: Run tests**

Run: `npm test -- tests/storage/watermark-cleaner.test.ts`  
Expected: PASS

---

### Task 7: GAT1400 Bidirectional Adapter

**Files:**
- Create: `src/protocol-gat1400-service/inbound.ts`
- Create: `src/protocol-gat1400-service/outbound.ts`
- Create: `src/protocol-gat1400-service/mapping.ts`
- Create: `src/protocol-gat1400-service/retry-queue.ts`
- Test: `tests/protocol/gat1400-bidi.test.ts`

**Step 1: Write failing tests for inbound/outbound mapping**

Validate transform to and from unified event model.

**Step 2: Implement idempotent outbound with ack/retry**

Use `event_id + target_platform` as idempotency key.

**Step 3: Add dead-letter handling**

Failed retries move to dead-letter queue with manual replay endpoint.

**Step 4: Run tests**

Run: `npm test -- tests/protocol/gat1400-bidi.test.ts`  
Expected: PASS

---

### Task 8: GB28181 Access and Cascade Services

**Files:**
- Create: `src/protocol-gb28181-service/sip-agent.ts`
- Create: `src/protocol-gb28181-service/catalog-sync.ts`
- Create: `src/protocol-gb28181-service/cascade.ts`
- Create: `src/protocol-gb28181-service/channel-map.ts`
- Test: `tests/protocol/gb28181-cascade.test.ts`

**Step 1: Write failing tests for register/keepalive/catalog**

Cover SIP registration lifecycle and channel sync behavior.

**Step 2: Implement channel mapping**

Normalize GB channel IDs to internal camera/channel identities with tenant/project binding.

**Step 3: Implement cascade forwarding contract**

Support upstream and downstream relay with audit logs.

**Step 4: Run tests**

Run: `npm test -- tests/protocol/gb28181-cascade.test.ts`  
Expected: PASS

---

### Task 9: Auth, RBAC, Data Scope, and Project Switching

**Files:**
- Create: `src/auth-rbac-service/models.sql`
- Create: `src/auth-rbac-service/permissions.ts`
- Create: `src/auth-rbac-service/project-context.ts`
- Create: `src/auth-rbac-service/middleware.ts`
- Test: `tests/auth/project-scope.test.ts`

**Step 1: Create schema**

Tables: users, roles, permissions, user_role, tenant, project, user_project_membership, audit_log.

**Step 2: Write failing scope-isolation tests**

Ensure users cannot query cross-project cameras/events without membership.

**Step 3: Implement middleware**

Require project context per request and validate against membership.

**Step 4: Add project switch audit**

Every project switch writes an audit record.

**Step 5: Run tests**

Run: `npm test -- tests/auth/project-scope.test.ts`  
Expected: PASS

---

### Task 10: Layered B/S UI and Branding Config

**Files:**
- Create: `src/cockpit-ui/routes.ts`
- Create: `src/cockpit-ui/pages/realtime/*`
- Create: `src/cockpit-ui/pages/analysis/*`
- Create: `src/cockpit-ui/pages/tasks/*`
- Create: `src/cockpit-ui/pages/protocol/*`
- Create: `src/cockpit-ui/pages/cockpit/*`
- Create: `src/cockpit-ui/pages/system/*`
- Create: `src/cockpit-ui/pages/system/branding.tsx`
- Test: `tests/ui/nav-structure.test.ts`

**Step 1: Write failing test for layered navigation**

Assert six top-level centers exist and no single-page collapsed mega view.

**Step 2: Implement route groups**

Realtime Center, Analysis Center, Task Center, Protocol Center, Cockpit, System Management.

**Step 3: Implement branding settings**

Fields: logo_light, logo_dark, system_title, browser_title.  
Apply by project/tenant context.

**Step 4: Run UI tests**

Run: `npm test -- tests/ui/nav-structure.test.ts`  
Expected: PASS

---

### Task 11: Observability and Operational Safety

**Files:**
- Create: `src/ops/metrics.ts`
- Create: `src/ops/health.ts`
- Create: `src/ops/alerts.ts`
- Test: `tests/ops/health-and-alerts.test.ts`

**Step 1: Define mandatory metrics**

Preview sessions, per-camera fps, inference p95, queue depth, dropped frames, protocol retry rates, storage usage.

**Step 2: Add health endpoints**

Each service exposes `/healthz` and `/readyz`.

**Step 3: Add alarms**

High watermark reached, queue sustained high, protocol dead-letter growth.

**Step 4: Run tests**

Run: `npm test -- tests/ops/health-and-alerts.test.ts`  
Expected: PASS

---

### Task 12: Integration Validation for 20-Channel Target

**Files:**
- Create: `tests/integration/e2e-20ch.test.ts`
- Create: `scripts/load/20ch-scenario.yaml`
- Create: `docs/architecture/performance-baseline-v1.md`

**Step 1: Build 20-channel synthetic scenario**

Mix H.264/H.265 streams and mixed alarm rates.

**Step 2: Execute integration test**

Run: `npm run test:integration -- --scenario scripts/load/20ch-scenario.yaml`  
Expected: all core flows pass.

**Step 3: Capture baseline**

Record CPU/NPU/memory/temperature, alert latency, preview stability, protocol success rate.

**Step 4: Produce go-live checklist**

Output required checklist with pass/fail and blocker items.

---

## Rollout Gates

1. Gate A: 4-channel internal dry run.
2. Gate B: 8-channel stability for 72 hours.
3. Gate C: 20-channel target with all mandatory KPIs.
4. Gate D: protocol interoperability (GAT1400 + GB28181) sign-off.
5. Gate E: security and permission review sign-off.

---

## Risks and Controls

1. H.265 browser fragmentation
- Control: capability matrix + whitelist + explicit compatibility messaging.

2. Preview pressure impacting inference
- Control: ingest fan-out isolation + adaptive scheduler + queue backpressure.

3. Protocol retry storms
- Control: idempotency keys + exponential backoff + dead-letter queue.

4. Cross-project data leakage
- Control: strict project context middleware + DB-level scope checks + audit.

---

Plan complete and saved to `docs/plans/2026-03-03-rk3588-ai-box-v1-implementation.md`. Two execution options:

1. Subagent-Driven (this session) - execute task-by-task in this session with review gates.
2. Parallel Session (separate) - open a fresh execution session dedicated to this plan.

Which approach?
