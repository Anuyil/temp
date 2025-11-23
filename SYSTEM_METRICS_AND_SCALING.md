# System Metrics Collection & Scaling Architecture

## 📊 PARTE 1: Come Raccogliere System Metrics (senza Metricbeat)

### ❌ Problema con Metricbeat
Metricbeat è parte dell'Elastic Stack:
- Agente che gira su ogni nodo
- Invia metrics a Elasticsearch
- Overhead: ~50MB RAM + CPU

**Se usiamo Prometheus, NON serve Metricbeat!**

---

### ✅ Soluzione: Prometheus Ecosystem

```
┌─────────────────────────────────────────────────────────┐
│                  PROMETHEUS SERVER                      │
│                  (Scrape ogni 15s)                      │
└───────────┬─────────────────────────────────────────────┘
            │
            │ HTTP GET /metrics
            │
    ┌───────┼────────┬─────────┬──────────┬──────────┐
    ▼       ▼        ▼         ▼          ▼          ▼
┌─────────────────────────────────────────────────────────┐
│                    TARGET NODES                         │
├─────────────────────────────────────────────────────────┤
│  [Injection-1]     [Relay-1]      [Egress-1]            │
│                                                          │
│  ┌──────────────────────────────────────────────┐       │
│  │ 1. node_exporter :9100/metrics               │       │
│  │    → System metrics (CPU, RAM, disk, net)    │       │
│  │                                               │       │
│  │ 2. cAdvisor :8080/metrics (se Docker)        │       │
│  │    → Container metrics (per-container stats) │       │
│  │                                               │       │
│  │ 3. Application :7070/metrics                 │       │
│  │    → Custom metrics (sessions, RTP, etc)     │       │
│  └──────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Componenti Prometheus per System Metrics

### 1️⃣ **node_exporter** (System Metrics)

**Cosa fa**:
- Raccoglie metriche OS-level
- CPU, memoria, disk I/O, network, filesystem
- Standard de facto per Prometheus

**Metriche esposte** (esempi):
```
# CPU
node_cpu_seconds_total{cpu="0",mode="idle"}
node_cpu_seconds_total{cpu="0",mode="system"}
node_cpu_seconds_total{cpu="0",mode="user"}

# Memory
node_memory_MemTotal_bytes
node_memory_MemAvailable_bytes
node_memory_MemFree_bytes

# Network
node_network_receive_bytes_total{device="eth0"}
node_network_transmit_bytes_total{device="eth0"}
node_network_receive_drop_total{device="eth0"}

# Disk
node_disk_io_time_seconds_total{device="sda"}
node_filesystem_avail_bytes{mountpoint="/"}
```

**Deployment**:
```yaml
# docker-compose.yml
services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"
```

**Prometheus scrape config**:
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets:
        - 'injection-1:9100'
        - 'relay-1:9100'
        - 'egress-1:9100'
```

---

### 2️⃣ **cAdvisor** (Container Metrics)

**Cosa fa**:
- Metrics per-container (se usi Docker/K8s)
- CPU, memoria, network per OGNI container
- Resource usage, limits, throttling

**Metriche esposte**:
```
# Per-container CPU
container_cpu_usage_seconds_total{name="injection-1"}
container_cpu_system_seconds_total{name="injection-1"}

# Per-container Memory
container_memory_usage_bytes{name="injection-1"}
container_memory_working_set_bytes{name="injection-1"}

# Network
container_network_receive_bytes_total{name="injection-1"}
container_network_transmit_bytes_total{name="injection-1"}
```

**Deployment**:
```yaml
# docker-compose.yml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - "8080:8080"
```

---

### 3️⃣ **Application Metrics** (Custom)

**Cosa fa**:
- Metriche specifiche della tua applicazione
- Sessions attive, mountpoints, port pool, RTP stats

**Implementazione**:
```javascript
// injection-node/src/metrics.js
const client = require('prom-client');

// Registra default metrics (heap, event loop, etc)
client.collectDefaultMetrics();

// Custom metrics
const activeSessions = new client.Gauge({
  name: 'media_tree_active_sessions',
  help: 'Number of active broadcast sessions',
  labelNames: ['tree_id', 'node_id']
});

const rtpPacketsSent = new client.Counter({
  name: 'media_tree_rtp_packets_sent_total',
  help: 'Total RTP packets sent',
  labelNames: ['stream_type', 'session_id']
});

// Endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});
```

---

## 🔄 PARTE 2: Chi Decide lo Scaling?

### ⚡ Domanda Critica

**Kubernetes ha autoscaling built-in (HPA, VPA, Cluster Autoscaler).**
**Ma il nostro sistema ha topologia custom (albero con relay-slot).**

**Chi comanda?**

---

## 🎯 Risposta: HYBRID APPROACH

### Architettura Recommended

```
┌──────────────────────────────────────────────────────────┐
│              LAYER 1: Kubernetes (Infra)                 │
│                                                           │
│  - Provisioning pods                                     │
│  - Health checks & restarts                              │
│  - Resource limits enforcement                           │
│  - NO decisioni di scaling (HPA disabled)               │
└──────────────────────────────────────────────────────────┘
                         ▲
                         │ kubectl apply / K8s API
                         │
┌──────────────────────────────────────────────────────────┐
│      LAYER 2: Custom Scaling Controller (CERVELLO)       │
│                                                           │
│  ┌────────────────────────────────────────────────┐      │
│  │  Decision Engine                               │      │
│  │  - Query Prometheus                            │      │
│  │  - Analyze: CPU, memory, port pool, viewers    │      │
│  │  - Decision: scale-out / scale-in / noop       │      │
│  └────────────┬───────────────────────────────────┘      │
│               │                                           │
│  ┌────────────▼───────────────────────────────────┐      │
│  │  Orchestration Engine                          │      │
│  │  1. Find available relay-slot                  │      │
│  │  2. Provision sub-tree (16 egress pods)        │      │
│  │  3. Update Redis topology                      │      │
│  │  4. Publish events → nodes reconfigure         │      │
│  └────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────┘
                         ▲
                         │ Metrics scrape
                         │
┌──────────────────────────────────────────────────────────┐
│              LAYER 3: Prometheus                         │
│                                                           │
│  - Scrape node_exporter (system)                         │
│  - Scrape cAdvisor (containers)                          │
│  - Scrape app metrics (RTP, sessions)                    │
│  - Store time-series (15 days retention)                 │
└──────────────────────────────────────────────────────────┘
                         ▲
                         │ Expose /metrics
                         │
┌──────────────────────────────────────────────────────────┐
│         LAYER 4: Media-Tree Nodes (Data Plane)           │
│                                                           │
│  [Injection] [Relay] [Egress] × N                        │
│  - node_exporter                                         │
│  - cAdvisor (if Docker)                                  │
│  - Application /metrics                                  │
└──────────────────────────────────────────────────────────┘
```

---

## 🧠 Perché il NOSTRO Controller (non K8s HPA)?

### Kubernetes HPA (Horizontal Pod Autoscaler)

**Cosa fa**:
```yaml
# HPA standard
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: egress-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: egress
  minReplicas: 8
  maxReplicas: 32
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**Risultato**:
- Se CPU >70% → Kubernetes crea nuovi pod `egress-9`, `egress-10`, ...
- **PROBLEMA**: Chi aggiorna Redis topology?
- **PROBLEMA**: Chi configura RTP forwarding dal parent?
- **PROBLEMA**: Chi crea Janus mountpoints per sessioni esistenti?

**❌ HPA non basta perché non capisce la nostra topologia!**

---

### Custom Scaling Controller (NOSTRO)

**Logica**:
```javascript
// scaling-controller/decision-engine.js
async evaluateScaling(treeId) {
  // 1. Query Prometheus
  const metrics = await this.queryMetrics(treeId);

  // avg CPU ultimi 5 min per egress nodes
  const avgCPU = metrics.cpu.avg;
  const maxPortPool = metrics.portPool.max;

  // 2. Decisione
  if (avgCPU > 70 || maxPortPool > 85) {
    return { action: 'SCALE_OUT', reason: 'High load' };
  }

  if (avgCPU < 20 && maxPortPool < 30) {
    return { action: 'SCALE_IN', reason: 'Low load' };
  }

  return { action: 'NOOP' };
}

async scaleOut(treeId) {
  // 1. Trova relay-slot disponibile
  const relaySlot = await this.findRelaySlot(treeId);

  // 2. Provision sub-tree via Kubernetes API
  await this.k8sClient.createDeployment({
    name: `egress-subtree-${Date.now()}`,
    replicas: 16,
    image: 'media-tree/egress:v1.0',
    env: {
      TREE_ID: treeId,
      PARENT_NODE_ID: relaySlot
    }
  });

  // 3. Aspetta pods ready
  await this.k8sClient.waitForPods(deployment, timeout: 60000);

  // 4. Update Redis topology
  const pods = await this.k8sClient.getPods(deployment);
  for (const pod of pods) {
    await redis.set(`parent:${pod.name}`, relaySlot);
    await redis.sadd(`children:${relaySlot}`, pod.name);
  }

  // 5. Publish events → relay-slot riconfigura forwarding
  await redis.publish(`topology:${relaySlot}`, {
    type: 'children-added',
    childIds: pods.map(p => p.name)
  });

  // 6. Egress nodes auto-create mountpoints (discoverExistingSessions)
  // Questo è già implementato!

  console.log(`✅ Scaled out: ${pods.length} egress attached to ${relaySlot}`);
}
```

**✅ Il nostro controller SA:**
- Topologia tree (parent/children in Redis)
- Logica relay-slot
- RTP forwarding configuration
- Session discovery per nuovi nodi

---

## 🔀 Division of Responsibilities

| Responsabilità | Kubernetes | Custom Controller |
|----------------|------------|-------------------|
| **Provisioning pods** | ✅ K8s API | Controller chiama K8s |
| **Health checks** | ✅ Liveness probes | Controller ignora unhealthy |
| **Resource limits** | ✅ CPU/memory limits | Controller non gestisce |
| **Pod restarts** | ✅ Auto-restart | Controller ricrea topology |
| **SCALE-OUT decision** | ❌ | ✅ Controller (query Prometheus) |
| **Topology update** | ❌ | ✅ Controller (Redis) |
| **RTP forwarding config** | ❌ | ✅ Controller (Pub/Sub) |
| **Relay-slot selection** | ❌ | ✅ Controller (logica custom) |
| **Session recovery** | ❌ | ✅ Nodes (discoverExistingSessions) |

---

## 🛠️ Implementazione Concreta

### 1. Prometheus Scrape Config

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # System metrics (CPU, RAM, disk, network)
  - job_name: 'node-exporter'
    static_configs:
      - targets:
        - 'injection-1:9100'
        - 'relay-1:9100'
        - 'egress-1:9100'
        - 'egress-2:9100'
        # ... tutti i nodi

  # Container metrics (per-container stats)
  - job_name: 'cadvisor'
    static_configs:
      - targets:
        - 'host-1:8080'  # cAdvisor on each host

  # Application metrics (sessions, RTP, mountpoints)
  - job_name: 'media-tree'
    static_configs:
      - targets:
        - 'injection-1:7070'
        - 'relay-1:7071'
        - 'egress-1:7073'
```

---

### 2. Scaling Controller - Query Prometheus

```javascript
// scaling-controller/prometheus-client.js
const axios = require('axios');

class PrometheusClient {
  constructor(baseURL = 'http://prometheus:9090') {
    this.baseURL = baseURL;
  }

  async query(promql, time = new Date()) {
    const response = await axios.get(`${this.baseURL}/api/v1/query`, {
      params: {
        query: promql,
        time: time.toISOString()
      }
    });

    return response.data.data.result;
  }

  // Avg CPU ultimi 5 min per egress nodes di un tree
  async getAvgCPU(treeId) {
    const promql = `
      avg(
        avg_over_time(
          (1 - rate(node_cpu_seconds_total{mode="idle",tree_id="${treeId}",node_type="egress"}[1m]))[5m:]
        )
      ) * 100
    `;

    const result = await this.query(promql);
    return parseFloat(result[0].value[1]);
  }

  // Max port pool utilization per tree
  async getMaxPortPoolUtilization(treeId) {
    const promql = `
      max(
        port_pool_utilization_percent{tree_id="${treeId}",node_type="egress"}
      )
    `;

    const result = await this.query(promql);
    return parseFloat(result[0].value[1]);
  }

  // Total memory usage
  async getMemoryUsage(nodeId) {
    const promql = `
      (1 - (node_memory_MemAvailable_bytes{node_id="${nodeId}"} / node_memory_MemTotal_bytes{node_id="${nodeId}"})) * 100
    `;

    const result = await this.query(promql);
    return parseFloat(result[0].value[1]);
  }

  // Network dropped packets (packet loss indicator)
  async getDroppedPackets(nodeId) {
    const promql = `
      rate(node_network_receive_drop_total{node_id="${nodeId}",device!="lo"}[5m])
    `;

    const result = await this.query(promql);
    return result.reduce((sum, r) => sum + parseFloat(r.value[1]), 0);
  }
}

module.exports = PrometheusClient;
```

---

### 3. Decision Engine

```javascript
// scaling-controller/decision-engine.js
const PrometheusClient = require('./prometheus-client');

class DecisionEngine {
  constructor(redis) {
    this.redis = redis;
    this.prometheus = new PrometheusClient();
  }

  async evaluate(treeId) {
    // Query metrics
    const avgCPU = await this.prometheus.getAvgCPU(treeId);
    const maxPortPool = await this.prometheus.getMaxPortPoolUtilization(treeId);
    const droppedPackets = await this.prometheus.getDroppedPackets(treeId);

    console.log(`[Tree ${treeId}] CPU: ${avgCPU.toFixed(1)}%, PortPool: ${maxPortPool.toFixed(1)}%, Drops: ${droppedPackets}`);

    // Scale-OUT conditions
    if (avgCPU > 70) {
      return {
        action: 'SCALE_OUT',
        reason: `High CPU (${avgCPU.toFixed(1)}%)`,
        urgency: 'HIGH'
      };
    }

    if (maxPortPool > 85) {
      return {
        action: 'SCALE_OUT',
        reason: `Port pool exhausted (${maxPortPool.toFixed(1)}%)`,
        urgency: 'CRITICAL'
      };
    }

    if (droppedPackets > 100) {
      return {
        action: 'SCALE_OUT',
        reason: `Packet loss detected (${droppedPackets} pkt/s)`,
        urgency: 'HIGH'
      };
    }

    // Scale-IN conditions
    if (avgCPU < 20 && maxPortPool < 30) {
      const egressCount = await this.redis.scard(`nodes:${treeId}:egress`);

      if (egressCount > 8) {  // Non scale-in sotto baseline
        return {
          action: 'SCALE_IN',
          reason: `Low load (CPU ${avgCPU.toFixed(1)}%)`,
          urgency: 'LOW'
        };
      }
    }

    return { action: 'NOOP' };
  }
}

module.exports = DecisionEngine;
```

---

## 📊 Grafana Dashboard Queries

```
# CPU Usage per Node
avg by (node_id) (
  rate(node_cpu_seconds_total{mode!="idle"}[5m])
) * 100

# Memory Usage per Node
(1 - (
  node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
)) * 100

# Network Throughput (RTP)
rate(node_network_transmit_bytes_total{device="eth0"}[1m])

# Active Sessions
media_tree_active_sessions

# Port Pool Utilization
port_pool_utilization_percent

# RTP Packet Loss (se implementato)
rate(rtp_packets_lost_total[5m])
```

---

## ✅ SUMMARY: Come Funziona Fine-to-Fine

```
1. node_exporter espone system metrics (:9100/metrics)
   → CPU, memoria, disk, network

2. cAdvisor espone container metrics (:8080/metrics)
   → Per-container CPU, memoria

3. Application espone custom metrics (:7070/metrics)
   → Sessions, RTP, mountpoints, port pool

4. Prometheus scrape tutti ogni 15s
   → Store time-series (15 giorni retention)

5. Scaling Controller query Prometheus ogni 30s
   → Calcola avg CPU, max port pool, ecc.

6. Decision Engine decide: SCALE_OUT / SCALE_IN / NOOP
   → Basato su thresholds

7. Orchestration Engine esegue azione
   → K8s API (provision pods)
   → Redis (update topology)
   → Pub/Sub (notify nodes)

8. Nodes reagiscono a eventi
   → RelayNode: ADD destination
   → EgressNode: CREATE mountpoint

9. Grafana visualizza metrics + alerts
   → Dashboard real-time
   → Slack notification se CPU >80%
```

---

## 🎯 Risposta Finale alla Tua Domanda

### Q: "Come facciamo senza Metricbeat per CPU/memoria?"
**A**: Usiamo **node_exporter** (standard Prometheus)

### Q: "Chi decide lo scaling, K8s o il nostro controller?"
**A**: **Il nostro controller** decide (query Prometheus → decisione)
       **Kubernetes** esegue (provision pods via API)

### Q: "Kubernetes HPA è disabilitato?"
**A**: **SÌ**, perché non capisce la topologia tree + relay-slot

---

## 🚀 Next Steps

1. **Setup Prometheus stack**:
   - [ ] Deploy Prometheus
   - [ ] Deploy node_exporter su ogni host
   - [ ] Deploy cAdvisor (se Docker)

2. **Implement app metrics**:
   - [ ] `/metrics` endpoint su Injection/Relay/Egress
   - [ ] Expose: sessions, mountpoints, port pool

3. **Build Scaling Controller**:
   - [ ] PrometheusClient (query wrapper)
   - [ ] DecisionEngine (thresholds logic)
   - [ ] OrchestrationEngine (K8s API + Redis)

4. **Test end-to-end**:
   - [ ] Simulate high load → verify scale-out
   - [ ] Verify topology update in Redis
   - [ ] Verify mountpoints created on new egress
