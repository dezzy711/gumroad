# NEXUS OMEGA — System-of-Systems Front-End Architecture

This document proposes an incremental architecture for evolving the current single-file NEXUS OMEGA app into a production-grade modular runtime, while preserving the existing HTML/CSS shell and visual language.

---

## 1) Blueprint: folder and module structure

```txt
nexus-omega/
  index.html                     # existing shell; keep CSS/DOM IDs
  src/
    app.js                       # boot entrypoint
    core/
      shell.js                   # boot sequence + root lifecycle
      router.js                  # tab/hash route activation
      eventBus.js                # typed-ish pub/sub + wildcards
      store.js                   # reactive immutable store + history
      pluginHost.js              # module registration/lifecycle
      config.js                  # tabs, thresholds, capability flags
      selectors.js               # shared derived selectors
    modules/
      pipeline/module.js
      threats/module.js
      environment/module.js
      surveillance/module.js
      aiConsole/module.js
      tasks/module.js
    visuals/
      neuralEngine.js
      surveillanceEngine.js
      backgroundEngine.js
      rafScheduler.js
    infra/
      simulationEngine.js
      agents/
        pipelineAgent.js
        threatAgent.js
        envAgent.js
      persistence.js
      migrations.js
      speech.js
      settings.js
```

### Existing DOM reuse strategy

Keep your current HTML and attach modules by IDs/data attributes:

- `#boot-screen` → `core/shell.js`
- `#tab-pipeline`, `#tab-threats`, … → `core/router.js`
- `#pane-pipeline`, `#pane-threats`, … → feature module root containers
- `#neural-canvas`, `#surveillance-canvas`, `#quantum-bg` → visual engines
- `#ai-query-input`, `#ai-submit`, `#ai-response` → AI console module

---

## 2) Core config and capability layer

`src/core/config.js`

```js
export const config = {
  schemaVersion: 3,
  capabilities: {
    enableSpeech: true,
    enableSimulation: true,
    enableSurveillance: true,
    enableBackgroundFx: true,
    enableTimeTravelDebug: true,
  },
  tabs: [
    { id: "pipeline", label: "Pipeline", moduleId: "pipeline" },
    { id: "threats", label: "Threat Matrix", moduleId: "threats" },
    { id: "environment", label: "Environment", moduleId: "environment" },
    { id: "surveillance", label: "Surveillance", moduleId: "surveillance" },
    { id: "tasks", label: "Tasks", moduleId: "tasks" },
    { id: "ai", label: "AI Console", moduleId: "aiConsole" },
  ],
  kpiThresholds: {
    targetRevenue: 2_000_000,
    criticalThreatSeverity: 0.85,
    aqiWarning: 140,
    co2Warning: 1100,
  },
  simulation: {
    tickMs: 1000,
    pipelineVolatility: 0.04,
    anomalySpawnChance: 0.22,
    anomalyDecayPerTick: 0.08,
    envNoise: { aqi: 2.5, co2: 12, temp: 0.08 },
  },
  themes: {
    neon: { primary: "#00f6ff", danger: "#ff3b6e", ok: "#4dff88" },
    omega: { primary: "#8e7dff", danger: "#ff6b9d", ok: "#6fffcd" },
  },
};
```

---

## 3) Event bus with wildcard routing

`src/core/eventBus.js`

```js
export function createEventBus() {
  const exact = new Map(); // type -> Set<handler>
  const wildcards = []; // [{ pattern, prefix, handler }]

  function subscribe(type, handler) {
    if (type.includes("*")) {
      const prefix = type.replace("*", ""); // SIM_* => SIM_
      const entry = { pattern: type, prefix, handler };
      wildcards.push(entry);
      return () => {
        const i = wildcards.indexOf(entry);
        if (i >= 0) wildcards.splice(i, 1);
      };
    }

    if (!exact.has(type)) exact.set(type, new Set());
    exact.get(type).add(handler);
    return () => exact.get(type)?.delete(handler);
  }

  function publish(type, payload = {}, meta = {}) {
    const event = {
      type,
      payload,
      ts: Date.now(),
      correlationId: meta.correlationId || crypto.randomUUID(),
      source: meta.source || "unknown",
    };

    exact.get(type)?.forEach((h) => h(event));
    wildcards
      .filter((w) => type.startsWith(w.prefix))
      .forEach((w) => w.handler(event));

    return event;
  }

  return { subscribe, publish };
}
```

---

## 4) Reactive store with slices, selectors, and history

`src/core/store.js`

```js
function deepFreezeDev(obj) {
  if (!obj || typeof obj !== "object") return obj;
  Object.freeze(obj);
  for (const v of Object.values(obj)) {
    if (v && typeof v === "object" && !Object.isFrozen(v)) deepFreezeDev(v);
  }
  return obj;
}

export function createStore({ maxHistory = 120 } = {}) {
  const slices = new Map(); // name -> { reducerMap }
  let state = {};
  const listeners = new Set();
  const selectors = new Set();
  const history = [];

  function registerSlice(name, initialState, reducerMap) {
    slices.set(name, { reducerMap });
    state = { ...state, [name]: structuredClone(initialState) };
  }

  function dispatch(type, payload = {}, meta = {}) {
    const prev = state;
    let next = prev;

    for (const [sliceName, { reducerMap }] of slices.entries()) {
      const reducer = reducerMap[type];
      if (!reducer) continue;
      const prevSlice = prev[sliceName];
      const nextSlice = reducer(prevSlice, payload, { state: prev, meta });
      if (nextSlice !== prevSlice) {
        if (next === prev) next = { ...prev };
        next[sliceName] = nextSlice;
      }
    }

    if (next !== prev) {
      if (meta.captureHistory !== false) {
        history.push({ ts: Date.now(), type, payload, prev, next });
        if (history.length > maxHistory) history.shift();
      }
      state = deepFreezeDev(next);
      listeners.forEach((l) => l(state, prev, { type, payload, meta }));
      selectors.forEach((sub) => {
        const nextValue = sub.selector(state);
        if (!Object.is(nextValue, sub.lastValue)) {
          const prevValue = sub.lastValue;
          sub.lastValue = nextValue;
          sub.listener(nextValue, prevValue);
        }
      });
    }
  }

  function select(selector, listener) {
    const sub = { selector, listener, lastValue: selector(state) };
    selectors.add(sub);
    listener(sub.lastValue, undefined); // eager emit
    return () => selectors.delete(sub);
  }

  function subscribe(listener) {
    listeners.add(listener);
    return () => listeners.delete(listener);
  }

  function getState() {
    return state;
  }

  function hydrate(partialState) {
    state = deepFreezeDev({ ...state, ...partialState });
    listeners.forEach((l) => l(state, undefined, { type: "HYDRATE" }));
  }

  function getHistory() {
    return history.slice();
  }

  return { registerSlice, dispatch, select, subscribe, getState, hydrate, getHistory };
}
```

### Sample slices and selectors

`src/core/selectors.js`

```js
export const selectors = {
  pipelineTotals: (s) => {
    const deals = s.pipeline.deals;
    const total = deals.reduce((acc, d) => acc + d.value, 0);
    const weighted = deals.reduce((acc, d) => acc + d.value * d.probability, 0);
    const targetProgress = weighted / s.pipeline.target;
    return { total, weighted, targetProgress };
  },

  threatSeverityIndex: (s) => {
    const anomalies = s.threats.anomalies.filter((a) => !a.resolved);
    const severityIndex = anomalies.length
      ? anomalies.reduce((acc, a) => acc + a.severity, 0) / anomalies.length
      : 0;
    return { count: anomalies.length, severityIndex, anomalies };
  },

  envSmoothed: (s) => {
    const h = s.environment.history;
    const window = h.slice(-10);
    const avg = (k) => (window.reduce((acc, it) => acc + it[k], 0) || 0) / Math.max(window.length, 1);
    return { aqiMA: avg("aqi"), co2MA: avg("co2"), tempMA: avg("temp") };
  },
};
```

---

## 5) Plugin host + module lifecycle contract

`src/core/pluginHost.js`

```js
// Shape (type-like):
// module = {
//   id, label, route,
//   capabilities: ["pipeline", "canvas:neural", ...],
//   onInit(ctx), onActivate(ctx), onDeactivate(ctx), onDispose(ctx)
// }

export function createPluginHost(context) {
  const modules = new Map();
  let activeId = null;

  function register(moduleDef) {
    modules.set(moduleDef.id, { ...moduleDef, initialized: false });
  }

  function initAll() {
    modules.forEach((m) => {
      if (m.initialized) return;
      m.onInit?.(context);
      m.initialized = true;
    });
  }

  function activate(moduleId) {
    if (activeId === moduleId) return;
    const prev = modules.get(activeId);
    const next = modules.get(moduleId);
    prev?.onDeactivate?.(context);
    next?.onActivate?.(context);
    activeId = moduleId;
  }

  function dispose() {
    modules.forEach((m) => m.onDispose?.(context));
    modules.clear();
  }

  return { register, initAll, activate, dispose };
}
```

---

## 6) Simulation engine + agents

`src/infra/simulationEngine.js`

```js
export function createSimulationEngine({ tickMs, store, eventBus, config }) {
  const agents = [];
  let timer = null;
  let lastTs = 0;
  let paused = false;

  function registerAgent(agent) {
    agents.push(agent);
  }

  function tick() {
    const now = performance.now();
    const dt = lastTs ? (now - lastTs) / 1000 : tickMs / 1000;
    lastTs = now;

    const context = { store, eventBus, config, now };
    for (const agent of agents) agent.onTick(dt, context);
    eventBus.publish("SIM_TICK", { dt, at: Date.now() }, { source: "simulation" });
  }

  function start() {
    if (timer || paused) return;
    lastTs = 0;
    timer = setInterval(tick, tickMs);
    eventBus.publish("SIM_STARTED", {}, { source: "simulation" });
  }

  function stop() {
    if (!timer) return;
    clearInterval(timer);
    timer = null;
    eventBus.publish("SIM_STOPPED", {}, { source: "simulation" });
  }

  function pause() {
    paused = true;
    stop();
    eventBus.publish("SIM_PAUSED", {}, { source: "simulation" });
  }

  function resume() {
    paused = false;
    start();
    eventBus.publish("SIM_RESUMED", {}, { source: "simulation" });
  }

  return { registerAgent, start, stop, pause, resume, isPaused: () => paused };
}
```

### Agent examples

`src/infra/agents/pipelineAgent.js`

```js
export const pipelineAgent = {
  id: "pipeline-agent",
  onTick(dt, { store, eventBus, config }) {
    const state = store.getState();
    const deals = state.pipeline.deals;

    deals.forEach((d) => {
      const drift = (Math.random() - 0.5) * config.simulation.pipelineVolatility * dt;
      const nextProb = Math.min(0.99, Math.max(0.05, d.probability + drift));
      if (Math.abs(nextProb - d.probability) > 0.12) {
        eventBus.publish("PIPELINE_PROBABILITY_SWING", { dealId: d.id, from: d.probability, to: nextProb }, { source: "pipelineAgent" });
      }
      store.dispatch("PIPELINE_DEAL_PATCHED", { id: d.id, patch: { probability: nextProb } }, { source: "pipelineAgent" });

      if (nextProb > 0.95 && Math.random() < 0.03) {
        store.dispatch("PIPELINE_DEAL_CLOSED", { id: d.id }, { source: "pipelineAgent" });
        eventBus.publish("DEAL_CLOSED", { dealId: d.id, value: d.value }, { source: "pipelineAgent" });
      }
    });
  },
};
```

`src/infra/agents/threatAgent.js`

```js
const ZONES = ["N-01", "N-02", "E-07", "S-03", "W-09"];

export const threatAgent = {
  id: "threat-agent",
  onTick(dt, { store, eventBus, config }) {
    if (Math.random() < config.simulation.anomalySpawnChance * dt) {
      const anomaly = {
        id: crypto.randomUUID(),
        zone: ZONES[Math.floor(Math.random() * ZONES.length)],
        severity: +(0.35 + Math.random() * 0.65).toFixed(2),
        ageSec: 0,
        resolved: false,
      };
      store.dispatch("THREAT_ANOMALY_ADDED", anomaly, { source: "threatAgent" });
      eventBus.publish("ANOMALY_DETECTED", anomaly, { source: "threatAgent" });
    }

    store.dispatch("THREAT_ANOMALY_AGED", { dt, decay: config.simulation.anomalyDecayPerTick }, { source: "threatAgent" });
  },
};
```

`src/infra/agents/envAgent.js`

```js
function noise(amp) {
  return (Math.random() - 0.5) * 2 * amp;
}

export const envAgent = {
  id: "env-agent",
  onTick(dt, { store, eventBus, config }) {
    const curr = store.getState().environment.current;
    const next = {
      aqi: Math.max(0, curr.aqi + noise(config.simulation.envNoise.aqi) * dt),
      co2: Math.max(350, curr.co2 + noise(config.simulation.envNoise.co2) * dt),
      temp: curr.temp + noise(config.simulation.envNoise.temp) * dt,
      ts: Date.now(),
    };

    store.dispatch("ENV_READING_CAPTURED", next, { source: "envAgent" });

    if (next.aqi > config.kpiThresholds.aqiWarning || next.co2 > config.kpiThresholds.co2Warning) {
      eventBus.publish("ENV_THRESHOLD_BREACH", next, { source: "envAgent" });
    }
  },
};
```

---

## 7) AI orchestration layer with intent graph

`src/modules/aiConsole/orchestrator.js`

```js
function keywordScore(query, terms = []) {
  const q = query.toLowerCase();
  return terms.reduce((acc, t) => (q.includes(t) ? acc + 0.2 : acc), 0);
}

export function buildIntentGraph(intents) {
  return intents
    .slice()
    .sort((a, b) => (b.priority || 0) - (a.priority || 0));
}

export function resolveQuery(query, context, intentGraph) {
  const scored = intentGraph
    .map((intent) => ({ intent, score: intent.matcher(query, context) }))
    .filter((it) => it.score > 0)
    .sort((a, b) => b.score - a.score);

  if (!scored.length) {
    return {
      primaryText: "No high-confidence intent detected.",
      secondaryText: "Try: 'pipeline status', 'threat matrix', or 'what is overdue?'.",
      payloads: [],
      followUps: ["Show threat matrix", "Summarize environment drift"],
    };
  }

  const winners = scored[0].score > 0.85 ? [scored[0]] : scored.slice(0, 2);
  const results = winners.map((w) => w.intent.handler(query, context, w.score));

  return {
    primaryText: results.map((r) => r.primaryText).join(" "),
    secondaryText: results.map((r) => r.secondaryText).filter(Boolean).join(" | "),
    payloads: results.flatMap((r) => r.payloads || []),
    followUps: [...new Set(results.flatMap((r) => r.followUps || []))].slice(0, 5),
  };
}

export const intents = [
  {
    id: "pipelineStatus",
    description: "Summarize weighted pipeline health",
    priority: 10,
    matcher: (q, { selectors }) => keywordScore(q, ["pipeline", "deal", "revenue", "weighted"]),
    handler: (_q, { selectors }) => {
      const p = selectors.pipelineTotals();
      return {
        primaryText: `Pipeline weighted value is ${Math.round(p.weighted).toLocaleString()} credits (${Math.round(p.targetProgress * 100)}% of target).`,
        secondaryText: `Gross pipeline is ${Math.round(p.total).toLocaleString()} credits.`,
        payloads: [{ type: "kpi", label: "Pipeline", data: p }],
        followUps: ["List top 5 deals by value", "Show deals at risk"],
      };
    },
  },
  {
    id: "threatMatrix",
    description: "Current anomaly load and severity",
    priority: 9,
    matcher: (q) => keywordScore(q, ["threat", "anomaly", "zone", "severity"]),
    handler: (_q, { selectors }) => {
      const t = selectors.threatSeverityIndex();
      return {
        primaryText: `${t.count} active anomalies. Severity index ${t.severityIndex.toFixed(2)}.`,
        payloads: [{ type: "table", label: "Active anomalies", data: t.anomalies.slice(0, 10) }],
        followUps: ["Escalate highest severity anomaly", "Open surveillance tab"],
      };
    },
  },
  {
    id: "environmentReport",
    description: "Air quality and carbon trend report",
    priority: 8,
    matcher: (q) => keywordScore(q, ["environment", "aqi", "co2", "temperature", "air"]),
    handler: (_q, { selectors }) => {
      const e = selectors.envSmoothed();
      return {
        primaryText: `Smoothed AQI ${e.aqiMA.toFixed(1)}, CO2 ${e.co2MA.toFixed(0)}ppm, temp ${e.tempMA.toFixed(2)}°C.`,
        payloads: [{ type: "kpi", label: "Environment", data: e }],
        followUps: ["Show threshold breaches", "Compare last 30 ticks"],
      };
    },
  },
  {
    id: "taskTriage",
    description: "Task prioritization and overdue triage",
    priority: 7,
    matcher: (q) => keywordScore(q, ["task", "overdue", "priority", "triage"]),
    handler: (_q, { store }) => {
      const tasks = store.getState().tasks.items;
      const overdue = tasks.filter((t) => !t.done && Date.now() > t.dueAt);
      return {
        primaryText: `${overdue.length} overdue tasks require intervention.`,
        payloads: [{ type: "list", label: "Overdue tasks", data: overdue.slice(0, 8) }],
        followUps: ["Prioritize overdue tasks", "Mark first overdue as escalated"],
      };
    },
  },
];
```

### Proactive intent subscriptions

```js
// Example during AI module init:
const unsubs = intents
  .filter((i) => Array.isArray(i.maybeSubscriptions))
  .flatMap((intent) =>
    intent.maybeSubscriptions.map((topic) =>
      eventBus.subscribe(topic, (evt) => {
        const response = intent.handler(`event:${evt.type}`, context, 1);
        renderAiResponse(response, { mode: "proactive" });
      })
    )
  );
```

---

## 8) Visual engines as independent render systems

`src/visuals/rafScheduler.js`

```js
export function createRafLoop(step) {
  let rafId = 0;
  let running = false;

  function frame(ts) {
    if (!running) return;
    step(ts);
    rafId = requestAnimationFrame(frame);
  }

  return {
    start() {
      if (running) return;
      running = true;
      rafId = requestAnimationFrame(frame);
    },
    stop() {
      running = false;
      cancelAnimationFrame(rafId);
    },
  };
}
```

`src/visuals/neuralEngine.js`

```js
import { createRafLoop } from "./rafScheduler.js";

export function createNeuralEngine() {
  let canvas, ctx, config, getState;
  let nodes = [];
  let resizeObs;

  function rebuild() {
    nodes = Array.from({ length: config.layers * config.nodesPerLayer }).map((_n, i) => ({
      id: i,
      x: Math.random() * canvas.width,
      y: Math.random() * canvas.height,
      phase: Math.random() * Math.PI * 2,
    }));
  }

  const loop = createRafLoop((ts) => {
    const s = getState();
    const coherence = Math.max(0.1, Math.min(1, s.pipelineHealth || 0.5));
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    ctx.globalAlpha = 0.15 + coherence * 0.35;
    nodes.forEach((n) => {
      const pulse = 0.5 + Math.sin(ts * 0.001 * config.speed + n.phase) * 0.5;
      ctx.fillStyle = pulse > 0.7 ? config.palette.hot : config.palette.cool;
      ctx.beginPath();
      ctx.arc(n.x, n.y, 1.2 + pulse * 2.4, 0, Math.PI * 2);
      ctx.fill();
    });
  });

  return {
    init(domNode, cfg, gs) {
      canvas = domNode;
      ctx = canvas.getContext("2d");
      config = cfg;
      getState = gs;
      resizeObs = new ResizeObserver(() => {
        canvas.width = canvas.clientWidth;
        canvas.height = canvas.clientHeight;
        rebuild();
      });
      resizeObs.observe(canvas);
      canvas.width = canvas.clientWidth;
      canvas.height = canvas.clientHeight;
      rebuild();
    },
    start() { loop.start(); },
    stop() { loop.stop(); },
    updateConfig(nextCfg) { config = { ...config, ...nextCfg }; rebuild(); },
    onEvent(evt) {
      if (evt.type === "DEAL_CLOSED") {
        config = { ...config, speed: Math.min(config.speed + 0.2, 3) };
      }
    },
    dispose() {
      loop.stop();
      resizeObs?.disconnect();
    },
  };
}
```

`src/visuals/surveillanceEngine.js`

```js
import { createRafLoop } from "./rafScheduler.js";

export function createSurveillanceEngine() {
  let canvas, ctx, getState;
  const flashByZone = new Map();

  const loop = createRafLoop(() => {
    const { anomalies } = getState();
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    drawZones(ctx, anomalies, flashByZone);

    flashByZone.forEach((v, k) => {
      const next = v - 0.04;
      if (next <= 0) flashByZone.delete(k);
      else flashByZone.set(k, next);
    });
  });

  return {
    init(domNode, _cfg, gs) {
      canvas = domNode;
      ctx = canvas.getContext("2d");
      getState = gs;
      canvas.width = canvas.clientWidth;
      canvas.height = canvas.clientHeight;
    },
    start() { loop.start(); },
    stop() { loop.stop(); },
    updateConfig() {},
    onEvent(evt) {
      if (evt.type.startsWith("THREAT_") || evt.type === "ANOMALY_DETECTED") {
        flashByZone.set(evt.payload.zone, 1);
      }
    },
  };
}

function drawZones(ctx, anomalies, flashByZone) {
  // Placeholder for current map projection logic; retain your existing geometry.
  // Use anomalies + flashByZone intensity to modulate fill/stroke.
}
```

### Lifecycle wiring on tab changes

```js
// surveillance module hooks
let engine;
let unsubscribeThreatEvents;

export const surveillanceModule = {
  id: "surveillance",
  route: "surveillance",
  capabilities: ["canvas:surveillance", "threats"],

  onInit({ eventBus, store }) {
    engine = createSurveillanceEngine();
    engine.init(document.querySelector("#surveillance-canvas"), {}, () => ({
      anomalies: store.getState().threats.anomalies,
    }));
    unsubscribeThreatEvents = eventBus.subscribe("THREAT_*", (evt) => engine.onEvent(evt));
  },

  onActivate() { engine.start(); },
  onDeactivate() { engine.stop(); },
  onDispose() { unsubscribeThreatEvents?.(); engine?.dispose?.(); },
};
```

---

## 9) Persistence, versioning, settings

`src/infra/migrations.js`

```js
export const CURRENT_SCHEMA = 3;

export function migratePersistedState(raw) {
  if (!raw || typeof raw !== "object") return null;
  let doc = structuredClone(raw);

  if (!doc.schemaVersion) doc.schemaVersion = 1;

  if (doc.schemaVersion === 1) {
    doc.settings = { ...doc.settings, themeVariant: doc.settings?.theme || "neon" };
    doc.schemaVersion = 2;
  }

  if (doc.schemaVersion === 2) {
    doc.simulation = doc.simulation || { paused: false };
    doc.schemaVersion = 3;
  }

  if (doc.schemaVersion !== CURRENT_SCHEMA) return null;
  return doc;
}
```

`src/infra/persistence.js`

```js
import { CURRENT_SCHEMA, migratePersistedState } from "./migrations.js";

const KEY = "nexus_omega_state";

export function loadState() {
  try {
    const raw = localStorage.getItem(KEY);
    if (!raw) return null;
    return migratePersistedState(JSON.parse(raw));
  } catch {
    return null;
  }
}

export function saveState({ slices, settings, simulation }) {
  try {
    const payload = {
      schemaVersion: CURRENT_SCHEMA,
      savedAt: Date.now(),
      slices,
      settings,
      simulation,
    };
    localStorage.setItem(KEY, JSON.stringify(payload));
  } catch {
    // storage full or blocked: fail soft
  }
}
```

`src/infra/settings.js`

```js
export function createSettings(initial = {}) {
  let s = {
    voiceEnabled: true,
    lastTab: "pipeline",
    themeVariant: "neon",
    ...initial,
  };
  return {
    get: () => s,
    patch: (p) => { s = { ...s, ...p }; return s; },
  };
}
```

---

## 10) Boot process wiring (works with existing HTML shell)

`src/app.js`

```js
import { config } from "./core/config.js";
import { createEventBus } from "./core/eventBus.js";
import { createStore } from "./core/store.js";
import { createPluginHost } from "./core/pluginHost.js";
import { createSimulationEngine } from "./infra/simulationEngine.js";
import { loadState, saveState } from "./infra/persistence.js";
import { pipelineAgent } from "./infra/agents/pipelineAgent.js";
import { threatAgent } from "./infra/agents/threatAgent.js";
import { envAgent } from "./infra/agents/envAgent.js";

export async function boot() {
  const eventBus = createEventBus();
  const store = createStore({ maxHistory: 300 });

  registerAllSlices(store); // pipeline/threats/environment/tasks/etc.

  // hydrate persisted state before module init
  const persisted = loadState();
  if (persisted?.slices) store.hydrate(persisted.slices);

  const simulation = createSimulationEngine({
    tickMs: config.simulation.tickMs,
    store,
    eventBus,
    config,
  });

  simulation.registerAgent(pipelineAgent);
  simulation.registerAgent(threatAgent);
  simulation.registerAgent(envAgent);

  const context = { config, eventBus, store, simulation };
  const pluginHost = createPluginHost(context);

  registerFeatureModules(pluginHost); // pipeline, threats, env, surveillance, ai, tasks
  pluginHost.initAll();

  // router activation uses existing tab buttons + panes
  wireRouter({
    tabs: config.tabs,
    onRouteChange: (tabId) => pluginHost.activate(tabId),
    initialTab: persisted?.settings?.lastTab || "pipeline",
  });

  if (config.capabilities.enableSimulation && !persisted?.simulation?.paused) {
    simulation.start();
  }

  // persist on important changes + periodic heartbeat
  const persistNow = () =>
    saveState({
      slices: pickPersistedSlices(store.getState()),
      settings: readSettingsFromUi(),
      simulation: { paused: simulation.isPaused() },
    });

  eventBus.subscribe("TASK_*", persistNow);
  eventBus.subscribe("DEAL_*", persistNow);
  setInterval(persistNow, 10_000);
}
```

---

## 11) Incremental migration plan (no big-bang rewrite)

1. **Introduce core/infra backbone first (no UI rewrite).**
   - Add `eventBus`, `store`, `config`, `persistence`, and `simulationEngine`.
   - Keep existing render functions intact; call them from `store.select(...)` listeners.

2. **Migrate one domain end-to-end (Pipeline recommended).**
   - Move pipeline data into a store slice.
   - Convert existing pipeline UI to subscribe to `selectors.pipelineTotals`.
   - Register `pipelineAgent` and publish `DEAL_*` events.

3. **Migrate Threat + Environment domain.**
   - Add anomaly and env slices.
   - Use `threatAgent` and `envAgent` for dynamic updates.
   - Replace imperative widget refreshes with selectors.

4. **Introduce plugin host and module lifecycle.**
   - Wrap each tab in a feature module with `onInit/onActivate/onDeactivate`.
   - Router only toggles active module and pane visibility.

5. **Replace keyword AI helper with intent orchestration.**
   - Implement `resolveQuery` and intent graph.
   - Add structured payload rendering in AI panel.
   - Enable optional speech from capability flag.

6. **Encapsulate canvases into render engines.**
   - Extract neural/surveillance/background loops into `visuals/`.
   - Start/stop engines on module activation.
   - Wire `onEvent` hooks for event-reactive visual bursts.

7. **Scale out to multi-dashboard / multi-tenant.**
   - Introduce `tenantId` in context and persistence key.
   - Namespaced event topics (`TENANT_A/THREAT_*`).
   - Capability flags per tenant profile.

---

## 12) Why this scales

- **System-of-systems runtime:** modules, agents, intents, and visuals evolve independently.
- **Deterministic dataflow:** reducers + immutable transitions simplify debugging and replay.
- **Capability-driven deployments:** easily ship reduced variants (kiosk mode, silent mode, no canvases).
- **Tenant-ready:** route/module/config overlays per environment without forked code.
- **Ops-friendly:** event stream can later be mirrored to telemetry or server-side analytics.

---

## 13) Minimal plugin definition example

```js
export const pipelineModule = {
  id: "pipeline",
  label: "Pipeline",
  route: "pipeline",
  capabilities: ["state:pipeline", "events:DEAL_*"],

  onInit({ store }) {
    this.unsub = store.select(
      (s) => s.pipeline.deals,
      (deals) => renderPipelineDeals(document.querySelector("#pane-pipeline"), deals)
    );
  },

  onActivate() {
    document.querySelector("#pane-pipeline").hidden = false;
  },

  onDeactivate() {
    document.querySelector("#pane-pipeline").hidden = true;
  },

  onDispose() {
    this.unsub?.();
  },
};
```

