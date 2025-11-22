# Media-Tree: Next Steps Roadmap

## 🎯 PRIORITÀ IMMEDIATE (Prossime 2 settimane)

### ✅ MILESTONE 1: Production-Ready Monitoring
**Obiettivo**: Sistema osservabile e debuggabile

#### Tasks
1. **Implementa Prometheus `/metrics` endpoint**
   - [ ] InjectionNode: sessioni attive, RTP stats
   - [ ] RelayNode: throughput, destinations
   - [ ] EgressNode: mountpoints, port pool, viewers

2. **Deploy Prometheus + Grafana**
   - [ ] Docker Compose setup
   - [ ] Scrape config per tutti i nodi
   - [ ] Dashboard base (CPU, memory, network)

3. **Grafana Dashboard RTP**
   - [ ] Packets/sec per stream (audio, video)
   - [ ] Bytes/sec throughput
   - [ ] Packet loss % (se disponibile da GStreamer)
   - [ ] Active sessions/mountpoints

4. **Alerting base**
   - [ ] CPU >80% per 5 min → Slack notification
   - [ ] Port pool >85% → Warning
   - [ ] Pipeline crashed → Critical alert

**Deliverable**: Dashboard funzionante + alert Slack
**Effort**: 3-4 giorni

---

### ✅ MILESTONE 2: OBS Integration
**Obiettivo**: Broadcaster possono usare OBS invece di test client

#### Tasks
1. **Verifica WHIP endpoint esistente**
   - [ ] Test con `curl` (POST SDP offer)
   - [ ] Verifica response (SDP answer)

2. **Test OBS con WHIP plugin**
   - [ ] Installa plugin obs-whip-output
   - [ ] Configura server endpoint
   - [ ] Test streaming 5 minuti

3. **Documenta setup OBS**
   - [ ] Guide step-by-step
   - [ ] Screenshot configurazione
   - [ ] Troubleshooting common issues

4. **Fallback RTMP bridge (se WHIP fail)**
   - [ ] Deploy nginx-rtmp container
   - [ ] ffmpeg bridge RTMP → RTP
   - [ ] Test latency (<2s accettabile)

**Deliverable**: OBS funzionante end-to-end
**Effort**: 2-3 giorni

---

## 🚀 MILESTONE 3: Scaling Controller (Settimane 3-4)

### Obiettivo
Automatizzare scale-out/scale-in basato su metriche Prometheus

### Architettura

```javascript
// scaling-controller/index.js
const PrometheusClient = require('prom-client');
const Redis = require('ioredis');
const NodeProvisioner = require('./provisioner'); // Docker/K8s API

class ScalingController {
  constructor() {
    this.redis = new Redis(REDIS_URL);
    this.prometheus = new PrometheusClient();
    this.provisioner = new NodeProvisioner();

    // Check metriche ogni 30 secondi
    setInterval(() => this.evaluateScaling(), 30000);
  }

  async evaluateScaling() {
    const trees = await this.redis.smembers('trees:active');

    for (const treeId of trees) {
      const metrics = await this.queryPrometheus(treeId);

      // Scale-OUT decision
      if (this.shouldScaleOut(metrics)) {
        await this.scaleOut(treeId);
      }

      // Scale-IN decision
      if (this.shouldScaleIn(metrics)) {
        await this.scaleIn(treeId);
      }
    }
  }

  async queryPrometheus(treeId) {
    // Query avg CPU ultimi 5 min per tutti egress del tree
    const cpuQuery = `
      avg(avg_over_time(
        node_cpu_usage_percent{tree_id="${treeId}",node_type="egress"}[5m]
      ))
    `;

    const cpu = await this.prometheus.query(cpuQuery);

    // Query port pool utilization
    const portPoolQuery = `
      max(port_pool_utilization{tree_id="${treeId}"})
    `;

    const portPool = await this.prometheus.query(portPoolQuery);

    return { cpu, portPool };
  }

  shouldScaleOut(metrics) {
    return (
      metrics.cpu > 70 ||           // CPU >70%
      metrics.portPool > 85         // Port pool >85%
    );
  }

  async scaleOut(treeId) {
    console.log(`[SCALE-OUT] Tree ${treeId}`);

    // 1. Trova relay-slot disponibile
    const relaySlot = await this.findAvailableRelaySlot(treeId);

    if (!relaySlot) {
      console.error('No relay-slot available!');
      return;
    }

    // 2. Provision sub-tree (16 egress nodes)
    const subTreeId = await this.provisioner.provisionSubTree({
      template: 'subTreeStandard',
      size: 16,
      parentId: relaySlot
    });

    // 3. Aspetta health checks
    await this.waitForHealthy(subTreeId, { timeout: 90000 });

    // 4. Attach a relay-slot
    await this.attachSubTree(relaySlot, subTreeId);

    console.log(`[SCALE-OUT] Complete: ${subTreeId} attached to ${relaySlot}`);
  }

  async scaleIn(treeId) {
    // TODO: Graceful drain + decommission
  }
}

module.exports = ScalingController;
```

### Tasks
- [ ] Implementa ScalingController base
- [ ] Integrazione Prometheus query
- [ ] NodeProvisioner (Docker API wrapper)
- [ ] Test scale-out manuale
- [ ] Test scale-in con graceful drain
- [ ] Deploy in container separato

**Effort**: 5-7 giorni

---

## 🏗️ MILESTONE 4: Relay-Slot Management (Settimana 5)

### Obiettivo
Gestire pool di relay-slot (warm + cold)

### Implementazione

```javascript
// relay-slot-manager/index.js
class RelaySlotManager {
  constructor() {
    this.warmPool = [];   // Relay-slot attivi (hot standby)
    this.coldPool = [];   // Relay-slot spenti (cold start)
  }

  async provisionRelaySlot(type = 'warm') {
    const nodeId = `relay-slot-${Date.now()}`;

    if (type === 'warm') {
      // Deploy container + start forwarder
      await this.provisioner.createNode({
        nodeId,
        nodeType: 'relay',
        autoStart: true
      });

      this.warmPool.push(nodeId);

    } else {
      // Deploy container ma NON start forwarder
      await this.provisioner.createNode({
        nodeId,
        nodeType: 'relay',
        autoStart: false
      });

      this.coldPool.push(nodeId);
    }
  }

  async activateRelaySlot(nodeId) {
    // Se cold, start forwarder (30s startup)
    if (this.coldPool.includes(nodeId)) {
      await this.provisioner.startForwarder(nodeId);

      // Wait health check
      await this.waitForHealthy(nodeId, { timeout: 40000 });

      // Move to warm pool
      this.coldPool = this.coldPool.filter(id => id !== nodeId);
      this.warmPool.push(nodeId);
    }

    return nodeId;
  }

  getAvailableSlot() {
    // Prefer warm (instant)
    if (this.warmPool.length > 0) {
      return this.warmPool[0];
    }

    // Fallback cold (30s delay)
    if (this.coldPool.length > 0) {
      return this.activateRelaySlot(this.coldPool[0]);
    }

    return null;
  }
}
```

### Tasks
- [ ] Implementa RelaySlotManager
- [ ] Integra con ScalingController
- [ ] Template config (warm: 1, cold: 2)
- [ ] Test activation time (cold start <35s)
- [ ] Cost analysis (warm vs cold)

**Effort**: 3-4 giorni

---

## 📊 MONITORING FINALE: Perché Prometheus

### Decision Matrix

| Requisito | Prometheus | Elastic Stack |
|-----------|------------|---------------|
| **Time-series metrics** | ✅ Native | ⚠️ Workaround |
| **Real-time alerts** | ✅ Built-in | ⚠️ Watcher (paid) |
| **Resource usage** | ✅ 2-4GB RAM | ❌ 8-16GB RAM |
| **Cost (AWS)** | ✅ $40/mese | ❌ $300/mese |
| **Query latency** | ✅ <100ms | ⚠️ 500ms-2s |
| **Kubernetes native** | ✅ Yes | ❌ No |
| **Packet-level analysis** | ❌ No | ✅ Packetbeat |
| **Log aggregation** | ❌ Use Loki | ✅ Native |

### Verdict
**START CON PROMETHEUS**

Ragioni:
1. **Costo**: $40/mese vs $300/mese
2. **Semplicità**: Single binary vs cluster ES
3. **Performance**: <100ms query latency
4. **Standard**: Tutti usano Prom per metrics
5. **Sufficient**: RTP metrics da app (non serve packet capture)

**Quando considerare Elastic**:
- Compliance/audit trail (retention anni)
- ML/analytics su network flows
- Full-text search su logs (ma Loki è sufficiente)

---

## 🎬 OBS: Perché WHIP è la scelta giusta

### Comparison

| Metodo | Latency | Compatibilità | Complessità | Production Ready |
|--------|---------|---------------|-------------|------------------|
| **WHIP** | ✅ <500ms | OBS 30+ | ⚠️ Plugin required | ✅ Yes |
| **RTMP** | ❌ 1-3s | ✅ All OBS | ✅ Simple | ✅ Yes |
| **SRT** | ⚠️ ~800ms | Plugin needed | ⚠️ Bridge complex | ⚠️ Emerging |

### Recommendation
1. **Primary**: WHIP (target latency <500ms)
2. **Fallback**: RTMP bridge (se WHIP non disponibile)

### Implementation Priority
```
Week 1-2: Implement WHIP (80% effort)
Week 2: Add RTMP fallback (20% effort, insurance policy)
```

**Ragione**: WHIP è futuro WebRTC, RTMP è legacy ma universale

---

## 📅 TIMELINE COMPLETA

```
Week 1-2: Prometheus + Grafana
  └─ Deliverable: Dashboard + alerts

Week 2-3: OBS Integration
  └─ Deliverable: OBS WHIP + RTMP fallback

Week 3-4: Scaling Controller
  └─ Deliverable: Auto scale-out funzionante

Week 5: Relay-Slot Management
  └─ Deliverable: Warm/cold pool gestito

Week 6: End-to-End Testing
  └─ Deliverable: Load test 10K viewer simulati

Week 7-8: Production Hardening
  └─ Redis Sentinel (HA)
  └─ Kubernetes manifests
  └─ CI/CD pipeline
```

**TOTAL: 8 settimane a MVP production-ready**

---

## 💰 COST ESTIMATE (AWS, eu-west-1)

### Development (1 tree, 8 egress)
```
- 1 × t3.medium (Injection): $30/mese
- 1 × t3.small (Relay-slot warm): $15/mese
- 8 × t3.small (Egress): $120/mese
- 1 × t3.micro (Redis): $7/mese
- 1 × t3.small (Prometheus+Grafana): $15/mese

TOTALE: ~$187/mese
```

### Production (multi-tree, 40K viewers)
```
- 2 × t3.large (Injection, HA): $120/mese
- 4 × t3.medium (Relay): $120/mese
- 32 × t3.small (Egress): $480/mese
- 1 × ElastiCache Redis (cache.m5.large): $150/mese
- 1 × t3.medium (Prometheus): $30/mese
- Data transfer (1TB/mese): $90/mese

TOTALE: ~$990/mese
```

**Cost per viewer (40K)**: $0.025/viewer/mese = **2.5 cents**

---

## ✅ SUCCESS CRITERIA

### Technical
- [ ] Latency glass-to-glass <500ms (p95)
- [ ] Packet loss <0.1%
- [ ] Uptime >99.9%
- [ ] Scale-out time <60s (warm) / <90s (cold)
- [ ] Auto-recovery da crash <30s

### Business
- [ ] Support 40K concurrent viewers (single tree)
- [ ] Cost <$0.03/viewer/mese
- [ ] OBS broadcaster experience semplice (<5 min setup)
- [ ] Zero-disruption scaling

---

## 🚨 RISKS & MITIGATION

### Risk 1: Prometheus retention insufficiente
**Mitigation**: Deploy Thanos/Cortex per long-term storage (later)

### Risk 2: WHIP plugin incompatibilità OBS
**Mitigation**: RTMP fallback già nel design

### Risk 3: Redis single point of failure
**Mitigation**: Redis Sentinel (settimana 7)

### Risk 4: Scaling controller bug causa over-provisioning
**Mitigation**:
- Dry-run mode in dev
- Max nodes hard limit (100)
- Manual approval for production (first month)

---

## 🎓 LEARNING RESOURCES

### Prometheus
- Official docs: https://prometheus.io/docs/
- PromQL tutorial: https://prometheus.io/docs/prometheus/latest/querying/basics/
- Best practices: https://prometheus.io/docs/practices/naming/

### WHIP
- IETF draft: https://datatracker.ietf.org/doc/draft-ietf-wish-whip/
- OBS plugin: https://github.com/obsproject/obs-whip-output

### GStreamer Metrics
- gst-stats: https://gstreamer.freedesktop.org/documentation/coreelements/stats.html
- RTP jitter buffer: https://gstreamer.freedesktop.org/documentation/rtpmanager/rtpjitterbuffer.html

---

## 🎯 DOMANDE DA RISOLVERE

Prima di procedere, chiarisci:

1. **Target viewer count per evento?**
   - Impatta: numero relay-slot, template tree

2. **Budget mensile disponibile?**
   - Impatta: warm vs cold relay-slot ratio

3. **Latency target?**
   - <500ms → WHIP mandatory
   - <2s → RTMP acceptable

4. **Geographic distribution?**
   - Single region → single tree
   - Multi-region → tree sharding

5. **HA requirement?**
   - 99.9% → Redis Sentinel + multi-AZ
   - 99.5% → single Redis OK

**Rispondi a queste e posso refine il roadmap!**
