# Monitoring Stack Comparison

## Elastic Stack (Metricbeat/Packetbeat + Elasticsearch + Kibana)

### ✅ PRO
- **Packetbeat eccellente per network flow analysis**
  - Cattura pacchetti a livello kernel
  - Analisi dettagliate TCP/UDP flows
  - Perfetto per forensics e troubleshooting
- **Kibana potente per log aggregation**
- **Query DSL molto espressivo**
- **Retention flessibile** (hot/warm/cold storage)

### ❌ CONTRO
- **MOLTO resource intensive**
  - Elasticsearch cluster: 8-16GB RAM minimo
  - CPU overhead significativo
  - Disk I/O elevato
- **Complessità operativa**
  - Gestione cluster (master, data, ingest nodes)
  - Sharding e replica configuration
  - Index lifecycle management
- **Costi elevati**
  - Retention: RTP flows = gigabytes/hour
  - Cluster HA = 3+ nodes
- **Latency query maggiore** (500ms-2s per aggregazioni complesse)
- **Overkill per metriche real-time semplici**

### 💰 COSTI STIMATI (AWS)
```
Elasticsearch cluster (3 × t3.large):
- Compute: $150/mese
- Storage (1TB retention): $100/mese
- Data transfer: $50/mese
TOTALE: ~$300/mese
```

---

## Prometheus + Grafana

### ✅ PRO
- **Lightweight e purpose-built per time-series**
  - Consuma 2-4GB RAM per deployment medio
  - Single binary, zero dependencies
- **Pull-based model**
  - Nodi espongono `/metrics` endpoint
  - Prometheus scrapia ogni 15-30s
  - Service discovery automatico (Kubernetes)
- **PromQL ottimizzato per time-series**
  - Query latency <100ms
  - Rate, aggregations, histograms built-in
- **Grafana alerting integrato**
  - Slack, PagerDuty, email
  - Templating potente
- **Industry standard per Kubernetes**
  - Operator nativo
  - Ecosystem maturo (exporters, libraries)

### ❌ CONTRO
- **Non sostituisce log aggregation**
  - Solo metriche numeriche
  - Per logs serve Loki/ELK separato
- **Retention limitato** (default 15 giorni)
  - Per long-term serve remote storage (Thanos, Cortex)
- **No packet-level analysis**
  - Devi implementare metriche applicative custom

### 💰 COSTI STIMATI (AWS)
```
Prometheus + Grafana (1 × t3.medium):
- Compute: $30/mese
- Storage (local SSD 100GB): $10/mese
TOTALE: ~$40/mese
```

---

## 🎯 RACCOMANDAZIONE: Hybrid Approach

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│              TIER 1: Real-Time Metrics                  │
│         Prometheus + Grafana (ALWAYS ON)                │
│  - Node CPU, memory, network                            │
│  - RTP packets/bytes counters (from app)                │
│  - Active sessions, viewers, mountpoints                │
│  - Port pool utilization                                │
│  - Pipeline state (GStreamer)                           │
│  - Alerting (CPU >80%, packet loss >0.1%)               │
└─────────────────────────────────────────────────────────┘
                        ▲
                        │ HTTP /metrics scrape every 15s
                ┌───────┴────────┬────────┬────────┐
                │                │        │        │
           [Injection]       [Relay]  [Egress]  [node-exporter]
            :7070/metrics   :7071     :7073     [cadvisor]


┌─────────────────────────────────────────────────────────┐
│         TIER 2: Logs (Optional, dev/debug only)         │
│              Loki + Promtail                            │
│  - Application logs (errors, warnings)                  │
│  - Structured JSON logs                                 │
│  - Retention: 7 giorni                                  │
└─────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────┐
│    TIER 3: Deep Analysis (On-Demand, troubleshooting)   │
│              tcpdump + Wireshark                        │
│  - Packet capture su richiesta (5-10 minuti)            │
│  - RTP stream analysis                                  │
│  - SSRC debugging                                       │
└─────────────────────────────────────────────────────────┘
```

---

## ⚡ Perché NON usare Packetbeat in Production

### 1. RTP è UDP high-throughput
```
1 sessione video HD:
- Video: 2 Mbps = ~180 packets/sec
- Audio: 128 Kbps = ~50 packets/sec

100 sessioni simultanee:
- 23.000 packets/sec
- Packetbeat overhead: ~15-20% CPU
- Elasticsearch ingest: 50MB/sec

❌ INSOSTENIBILE per 8K+ viewer
```

### 2. Alternative migliori
**Invece di catturare TUTTI i pacchetti**, implementa:

```javascript
// egress-forwarder.c
// Esponi metriche GStreamer via file/socket

typedef struct {
  uint64_t packets_received;
  uint64_t packets_sent;
  uint64_t bytes_received;
  uint64_t bytes_sent;
  uint32_t jitter_ms;
  float packet_loss_pct;
} StreamMetrics;

// Node.js legge e espone su /metrics
app.get('/metrics', async (req, res) => {
  const metrics = await readForwarderMetrics();

  res.send(`
# HELP rtp_packets_received_total Total RTP packets received
# TYPE rtp_packets_received_total counter
rtp_packets_received_total{stream="video"} ${metrics.video.packets_received}
rtp_packets_received_total{stream="audio"} ${metrics.audio.packets_received}

# HELP rtp_jitter_milliseconds Current RTP jitter
# TYPE rtp_jitter_milliseconds gauge
rtp_jitter_milliseconds{stream="video"} ${metrics.video.jitter_ms}

# HELP rtp_packet_loss_percent Packet loss percentage
# TYPE rtp_packet_loss_percent gauge
rtp_packet_loss_percent{stream="video"} ${metrics.video.packet_loss_pct}
  `);
});
```

---

## 🛠️ Implementation Plan

### Step 1: Aggiungi Prometheus `/metrics` endpoint
- [ ] InjectionNode: sessioni attive, RTP forwarding stats
- [ ] RelayNode: throughput, destinations count
- [ ] EgressNode: mountpoints attivi, port pool, viewer count

### Step 2: Deploy Prometheus + Grafana
- [ ] Docker Compose setup
- [ ] Scrape config per tutti i nodi
- [ ] Retention: 30 giorni (sufficiente)

### Step 3: Grafana Dashboards
- [ ] Overview: CPU, memory, network per nodo
- [ ] RTP: packets/sec, bytes/sec, jitter, loss
- [ ] Business: sessioni attive, viewer totali, capacity

### Step 4: Alerting
- [ ] CPU >80% per 5 minuti → Scale-out
- [ ] Packet loss >0.1% → Investigate
- [ ] Port pool >85% → Add egress nodes

---

## 📈 Migration Path

### Fase 1 (Now): Start con Prometheus
- Costi bassi ($40/mese)
- Setup rapido (1-2 giorni)
- Sufficiente per MVP e primi 10K viewer

### Fase 2 (Se necessario): Aggiungi Loki per logs
- Solo se troubleshooting frequente
- Retention breve (7 giorni)
- +$20/mese

### Fase 3 (Mai?): Elastic Stack
- Solo se serve compliance/audit trail
- Oppure analytics avanzate (ML on flows)
- +$300/mese

**VERDICT: Start with Prometheus. Elastic Stack is overkill.**
