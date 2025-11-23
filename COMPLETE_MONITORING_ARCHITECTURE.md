# ARCHITETTURA COMPLETA: Monitoring & Scaling Media-Tree

## 🎯 PANORAMICA

Sistema di monitoring a **3 livelli** dove **ogni nodo espone metriche multiple**:

```
┌────────────────────────────────────────────────────────────────┐
│                    PROMETHEUS SERVER                           │
│                  (Scrape Hub Centrale)                         │
│                                                                 │
│  - Scrape ogni 15 secondi                                      │
│  - Store time-series (15 giorni retention)                     │
│  - Expose API per query (/api/v1/query)                        │
└────────────────────┬───────────────────────────────────────────┘
                     │
                     │ HTTP GET ogni 15s
                     │
    ┌────────────────┼────────────────┬────────────────┐
    │                │                │                │
    ▼                ▼                ▼                ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ Injection-1 │ │  Relay-1    │ │  Egress-1   │ │  Egress-2   │
│             │ │             │ │             │ │             │
│ ┌─────────┐ │ │ ┌─────────┐ │ │ ┌─────────┐ │ │ ┌─────────┐ │
│ │  node_  │ │ │ │  node_  │ │ │ │  node_  │ │ │ │  node_  │ │
│ │exporter │ │ │ │exporter │ │ │ │exporter │ │ │ │exporter │ │
│ │  :9100  │ │ │ │  :9100  │ │ │ │  :9100  │ │ │ │  :9100  │ │
│ └────┬────┘ │ │ └────┬────┘ │ │ └────┬────┘ │ │ └────┬────┘ │
│      │      │ │      │      │ │      │      │ │      │      │
│ ┌────┴────┐ │ │ ┌────┴────┐ │ │ ┌────┴────┐ │ │ ┌────┴────┐ │
│ │  App    │ │ │ │  App    │ │ │ │  App    │ │ │ │  App    │ │
│ │ :7070   │ │ │ │ :7071   │ │ │ │ :7073   │ │ │ │ :7073   │ │
│ │/metrics │ │ │ │/metrics │ │ │ │/metrics │ │ │ │/metrics │ │
│ └─────────┘ │ │ └─────────┘ │ │ └─────────┘ │ │ └─────────┘ │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
```

**Principio chiave**: Ogni nodo è **auto-sufficiente** per esporre le proprie metriche.

---

## 📊 LIVELLO 1: System Metrics (node_exporter)

### Cos'è node_exporter?

**Official Prometheus exporter** per metriche OS-level:
- CPU usage per core
- Memory (total, available, free, buffers, cache)
- Disk I/O (read/write bytes, IOPS)
- Network (bytes, packets, drops, errors)
- Filesystem (disk usage per mountpoint)
- Load average

**Deployment**: **Un'istanza per ogni nodo** (host o container)

---

### Configurazione per Ogni Tipo di Nodo

#### A. INJECTION NODE

```yaml
# injection-node/docker-compose.yml
services:
  injection-app:
    build: .
    container_name: injection-1
    ports:
      - "7070:7070"
    networks:
      - media-tree

  node-exporter-injection:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter-injection-1
    restart: unless-stopped
    ports:
      - "9100:9100"
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    networks:
      - media-tree
```

**Metriche esposte** (esempi):
```
# GET http://injection-1:9100/metrics

# CPU
node_cpu_seconds_total{cpu="0",mode="idle"} 12345.67
node_cpu_seconds_total{cpu="0",mode="system"} 234.56
node_cpu_seconds_total{cpu="0",mode="user"} 567.89

# Memory
node_memory_MemTotal_bytes 8589934592       # 8GB
node_memory_MemAvailable_bytes 5368709120   # 5GB available
node_memory_MemFree_bytes 2147483648        # 2GB free

# Network
node_network_receive_bytes_total{device="eth0"} 123456789
node_network_transmit_bytes_total{device="eth0"} 987654321
node_network_receive_drop_total{device="eth0"} 0
node_network_transmit_drop_total{device="eth0"} 0

# Disk
node_disk_read_bytes_total{device="sda"} 12345678
node_disk_written_bytes_total{device="sda"} 98765432
node_filesystem_avail_bytes{mountpoint="/"} 10737418240  # 10GB available
```

---

#### B. RELAY NODE

```yaml
# relay-node/docker-compose.yml
services:
  relay-app:
    build: .
    container_name: relay-1
    ports:
      - "7071:7071"
      - "5002:5002/udp"
      - "5004:5004/udp"
    networks:
      - media-tree

  node-exporter-relay:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter-relay-1
    restart: unless-stopped
    ports:
      - "9100:9100"
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    networks:
      - media-tree
```

**Metriche critiche per Relay**:
```
# Network throughput (RTP forwarding)
node_network_receive_bytes_total{device="eth0"}
node_network_transmit_bytes_total{device="eth0"}

# Network drops (indicatore packet loss)
node_network_receive_drop_total{device="eth0"}
node_network_transmit_drop_total{device="eth0"}

# CPU (GStreamer forwarding load)
node_cpu_seconds_total{mode!="idle"}

# Memory (buffer usage)
node_memory_Buffers_bytes
node_memory_Cached_bytes
```

---

#### C. EGRESS NODE

```yaml
# egress-node/docker-compose.yml
services:
  egress-app:
    build: .
    container_name: egress-1
    ports:
      - "7073:7073"
      - "5002:5002/udp"
      - "5004:5004/udp"
      - "6000-6200:6000-6200/udp"
    networks:
      - media-tree

  node-exporter-egress:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter-egress-1
    restart: unless-stopped
    ports:
      - "9100:9100"
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    networks:
      - media-tree
```

**Metriche critiche per Egress**:
```
# CPU (demux + Janus load)
node_cpu_seconds_total

# Memory (mountpoints + viewer connections)
node_memory_MemAvailable_bytes

# Network (viewer throughput)
node_network_transmit_bytes_total{device="eth0"}

# Disk (logs, temp files)
node_filesystem_avail_bytes{mountpoint="/"}
```

---

## 📊 LIVELLO 2: Application Metrics (Custom /metrics)

Ogni nodo espone **metriche applicative** specifiche.

### A. INJECTION NODE (:7070/metrics)

```javascript
// injection-node/src/metrics.js
const client = require('prom-client');

// Enable default metrics (heap, event loop, GC)
const register = new client.Registry();
client.collectDefaultMetrics({ register });

// Custom metrics
const activeSessions = new client.Gauge({
  name: 'media_tree_injection_active_sessions',
  help: 'Number of active broadcast sessions',
  labelNames: ['tree_id', 'room_id'],
  registers: [register]
});

const rtpPacketsSent = new client.Counter({
  name: 'media_tree_injection_rtp_packets_sent_total',
  help: 'Total RTP packets forwarded to children',
  labelNames: ['stream_type', 'session_id', 'ssrc'],
  registers: [register]
});

const rtpBytesSent = new client.Counter({
  name: 'media_tree_injection_rtp_bytes_sent_total',
  help: 'Total RTP bytes forwarded',
  labelNames: ['stream_type', 'session_id'],
  registers: [register]
});

const childrenCount = new client.Gauge({
  name: 'media_tree_injection_children_count',
  help: 'Number of child nodes',
  labelNames: ['tree_id'],
  registers: [register]
});

// Update metrics (chiamato periodicamente)
function updateMetrics() {
  const sessions = SessionManager.getActiveSessions();

  activeSessions.set(
    { tree_id: TREE_ID, room_id: 'all' },
    sessions.length
  );

  childrenCount.set(
    { tree_id: TREE_ID },
    InjectionNode.children.length
  );
}

// Endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

module.exports = { activeSessions, rtpPacketsSent, rtpBytesSent, updateMetrics };
```

**Output esempio**:
```
# GET http://injection-1:7070/metrics

# HELP media_tree_injection_active_sessions Number of active broadcast sessions
# TYPE media_tree_injection_active_sessions gauge
media_tree_injection_active_sessions{tree_id="tree-1",room_id="all"} 3

# HELP media_tree_injection_rtp_packets_sent_total Total RTP packets forwarded
# TYPE media_tree_injection_rtp_packets_sent_total counter
media_tree_injection_rtp_packets_sent_total{stream_type="video",session_id="sess-123",ssrc="222222"} 45678
media_tree_injection_rtp_packets_sent_total{stream_type="audio",session_id="sess-123",ssrc="111111"} 12345

# HELP media_tree_injection_children_count Number of child nodes
# TYPE media_tree_injection_children_count gauge
media_tree_injection_children_count{tree_id="tree-1"} 10

# HELP nodejs_heap_size_total_bytes Total heap size
# TYPE nodejs_heap_size_total_bytes gauge
nodejs_heap_size_total_bytes 50331648

# HELP nodejs_heap_size_used_bytes Used heap size
# TYPE nodejs_heap_size_used_bytes gauge
nodejs_heap_size_used_bytes 25165824
```

---

### B. RELAY NODE (:7071/metrics)

```javascript
// relay-node/src/metrics.js
const client = require('prom-client');

const register = new client.Registry();
client.collectDefaultMetrics({ register });

const destinationsCount = new client.Gauge({
  name: 'media_tree_relay_destinations_count',
  help: 'Number of active destinations (children)',
  labelNames: ['tree_id', 'stream_type'],
  registers: [register]
});

const rtpPacketsForwarded = new client.Counter({
  name: 'media_tree_relay_rtp_packets_forwarded_total',
  help: 'Total RTP packets forwarded',
  labelNames: ['stream_type'],
  registers: [register]
});

const forwarderState = new client.Gauge({
  name: 'media_tree_relay_forwarder_state',
  help: 'Forwarder pipeline state (0=null, 1=ready, 2=paused, 3=playing)',
  labelNames: ['pipeline'],
  registers: [register]
});

// Update da forwarder C (via IPC)
async function updateFromForwarder() {
  const stats = await ForwarderIPC.getStats();

  destinationsCount.set(
    { tree_id: TREE_ID, stream_type: 'audio' },
    stats.audioDestinations.length
  );

  destinationsCount.set(
    { tree_id: TREE_ID, stream_type: 'video' },
    stats.videoDestinations.length
  );

  rtpPacketsForwarded.inc(
    { stream_type: 'audio' },
    stats.audioPackets
  );

  rtpPacketsForwarded.inc(
    { stream_type: 'video' },
    stats.videoPackets
  );

  // GStreamer state: 1=READY, 2=PAUSED, 3=PLAYING
  forwarderState.set(
    { pipeline: 'audio' },
    stats.audioPipelineState
  );
}

app.get('/metrics', async (req, res) => {
  await updateFromForwarder();
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

**Output esempio**:
```
# GET http://relay-1:7071/metrics

# HELP media_tree_relay_destinations_count Number of active destinations
# TYPE media_tree_relay_destinations_count gauge
media_tree_relay_destinations_count{tree_id="tree-1",stream_type="audio"} 16
media_tree_relay_destinations_count{tree_id="tree-1",stream_type="video"} 16

# HELP media_tree_relay_rtp_packets_forwarded_total Total RTP packets forwarded
# TYPE media_tree_relay_rtp_packets_forwarded_total counter
media_tree_relay_rtp_packets_forwarded_total{stream_type="audio"} 123456
media_tree_relay_rtp_packets_forwarded_total{stream_type="video"} 456789

# HELP media_tree_relay_forwarder_state Forwarder pipeline state
# TYPE media_tree_relay_forwarder_state gauge
media_tree_relay_forwarder_state{pipeline="audio"} 3
media_tree_relay_forwarder_state{pipeline="video"} 3
```

---

### C. EGRESS NODE (:7073/metrics)

```javascript
// egress-node/src/metrics.js
const client = require('prom-client');

const register = new client.Registry();
client.collectDefaultMetrics({ register });

const activeMountpoints = new client.Gauge({
  name: 'media_tree_egress_active_mountpoints',
  help: 'Number of active Janus streaming mountpoints',
  labelNames: ['tree_id', 'node_id'],
  registers: [register]
});

const activeViewers = new client.Gauge({
  name: 'media_tree_egress_active_viewers',
  help: 'Number of active WHEP viewers',
  labelNames: ['session_id'],
  registers: [register]
});

const portPoolUtilization = new client.Gauge({
  name: 'media_tree_egress_port_pool_utilization_percent',
  help: 'Port pool utilization percentage',
  labelNames: ['node_id'],
  registers: [register]
});

const portPoolAllocated = new client.Gauge({
  name: 'media_tree_egress_port_pool_allocated',
  help: 'Number of allocated ports',
  labelNames: ['node_id'],
  registers: [register]
});

const portPoolAvailable = new client.Gauge({
  name: 'media_tree_egress_port_pool_available',
  help: 'Number of available ports',
  labelNames: ['node_id'],
  registers: [register]
});

const rtpPacketsReceived = new client.Counter({
  name: 'media_tree_egress_rtp_packets_received_total',
  help: 'Total RTP packets received from parent',
  labelNames: ['stream_type', 'ssrc'],
  registers: [register]
});

// Update metrics
function updateMetrics() {
  const mountpoints = MountpointManager.getAll();

  activeMountpoints.set(
    { tree_id: TREE_ID, node_id: NODE_ID },
    mountpoints.length
  );

  // Port pool stats
  const poolStats = PortPool.getStats();
  portPoolAllocated.set({ node_id: NODE_ID }, poolStats.allocated);
  portPoolAvailable.set({ node_id: NODE_ID }, poolStats.available);
  portPoolUtilization.set(
    { node_id: NODE_ID },
    (poolStats.allocated / (poolStats.allocated + poolStats.available)) * 100
  );

  // Viewer count per session
  mountpoints.forEach(mp => {
    const viewers = WhepServer.getViewerCount(mp.sessionId);
    activeViewers.set({ session_id: mp.sessionId }, viewers);
  });
}

app.get('/metrics', async (req, res) => {
  updateMetrics();
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

**Output esempio**:
```
# GET http://egress-1:7073/metrics

# HELP media_tree_egress_active_mountpoints Number of active mountpoints
# TYPE media_tree_egress_active_mountpoints gauge
media_tree_egress_active_mountpoints{tree_id="tree-1",node_id="egress-1"} 5

# HELP media_tree_egress_active_viewers Number of active viewers
# TYPE media_tree_egress_active_viewers gauge
media_tree_egress_active_viewers{session_id="sess-123"} 234
media_tree_egress_active_viewers{session_id="sess-456"} 189

# HELP media_tree_egress_port_pool_utilization_percent Port pool utilization
# TYPE media_tree_egress_port_pool_utilization_percent gauge
media_tree_egress_port_pool_utilization_percent{node_id="egress-1"} 62.5

# HELP media_tree_egress_port_pool_allocated Allocated ports
# TYPE media_tree_egress_port_pool_allocated gauge
media_tree_egress_port_pool_allocated{node_id="egress-1"} 10

# HELP media_tree_egress_port_pool_available Available ports
# TYPE media_tree_egress_port_pool_available gauge
media_tree_egress_port_pool_available{node_id="egress-1"} 6

# HELP media_tree_egress_rtp_packets_received_total Total RTP packets received
# TYPE media_tree_egress_rtp_packets_received_total counter
media_tree_egress_rtp_packets_received_total{stream_type="audio",ssrc="111111"} 12345
media_tree_egress_rtp_packets_received_total{stream_type="video",ssrc="222222"} 45678
```

---

## 📊 LIVELLO 3: Container Metrics (cAdvisor) - OPZIONALE

Se deployment è Docker-based, cAdvisor fornisce metriche **per-container**.

```yaml
# docker-compose.yml (host level)
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.0
    container_name: cadvisor
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    privileged: true
    devices:
      - /dev/kmsg
```

**Metriche esposte**:
```
# GET http://host:8080/metrics

# Per-container CPU
container_cpu_usage_seconds_total{name="injection-1"} 123.45
container_cpu_system_seconds_total{name="injection-1"} 23.45

# Per-container Memory
container_memory_usage_bytes{name="injection-1"} 536870912  # 512MB
container_memory_working_set_bytes{name="injection-1"} 402653184

# Per-container Network
container_network_receive_bytes_total{name="injection-1"} 12345678
container_network_transmit_bytes_total{name="injection-1"} 98765432
```

**Nota**: Se usi **Kubernetes**, cAdvisor è **integrato** in kubelet (non serve deployment separato).

---

## 🔧 PROMETHEUS: Configurazione Scrape

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'media-tree-prod'
    region: 'eu-west-1'

scrape_configs:
  # ============================================
  # SYSTEM METRICS (node_exporter)
  # ============================================
  - job_name: 'node-exporter'
    static_configs:
      - targets:
        # Injection nodes
        - 'injection-1:9100'
        - 'injection-2:9100'

        # Relay nodes
        - 'relay-1:9100'
        - 'relay-2:9100'

        # Egress nodes (esempio 8 nodi)
        - 'egress-1:9100'
        - 'egress-2:9100'
        - 'egress-3:9100'
        - 'egress-4:9100'
        - 'egress-5:9100'
        - 'egress-6:9100'
        - 'egress-7:9100'
        - 'egress-8:9100'

    # Labels per identificare nodi
    relabel_configs:
      - source_labels: [__address__]
        regex: '([^:]+):.*'
        target_label: 'node_id'
        replacement: '${1}'

      - source_labels: [node_id]
        regex: 'injection-.*'
        target_label: 'node_type'
        replacement: 'injection'

      - source_labels: [node_id]
        regex: 'relay-.*'
        target_label: 'node_type'
        replacement: 'relay'

      - source_labels: [node_id]
        regex: 'egress-.*'
        target_label: 'node_type'
        replacement: 'egress'

  # ============================================
  # APPLICATION METRICS
  # ============================================
  - job_name: 'media-tree-injection'
    static_configs:
      - targets:
        - 'injection-1:7070'
        - 'injection-2:7070'
    metrics_path: '/metrics'
    relabel_configs:
      - source_labels: [__address__]
        regex: '([^:]+):.*'
        target_label: 'node_id'

  - job_name: 'media-tree-relay'
    static_configs:
      - targets:
        - 'relay-1:7071'
        - 'relay-2:7071'
    metrics_path: '/metrics'
    relabel_configs:
      - source_labels: [__address__]
        regex: '([^:]+):.*'
        target_label: 'node_id'

  - job_name: 'media-tree-egress'
    static_configs:
      - targets:
        - 'egress-1:7073'
        - 'egress-2:7073'
        - 'egress-3:7073'
        - 'egress-4:7073'
        - 'egress-5:7073'
        - 'egress-6:7073'
        - 'egress-7:7073'
        - 'egress-8:7073'
    metrics_path: '/metrics'
    relabel_configs:
      - source_labels: [__address__]
        regex: '([^:]+):.*'
        target_label: 'node_id'

  # ============================================
  # CONTAINER METRICS (cAdvisor) - OPZIONALE
  # ============================================
  - job_name: 'cadvisor'
    static_configs:
      - targets:
        - 'host-1:8080'
        - 'host-2:8080'
    metrics_path: '/metrics'

  # ============================================
  # REDIS METRICS (redis_exporter) - OPZIONALE
  # ============================================
  - job_name: 'redis'
    static_configs:
      - targets:
        - 'redis-exporter:9121'
```

---

## 🎯 SCALING CONTROLLER: Query Prometheus

### PrometheusClient Implementation

```javascript
// scaling-controller/prometheus-client.js
const axios = require('axios');

class PrometheusClient {
  constructor(baseURL = 'http://prometheus:9090') {
    this.baseURL = baseURL;
  }

  /**
   * Query Prometheus (instant query)
   */
  async query(promql, time = new Date()) {
    const response = await axios.get(`${this.baseURL}/api/v1/query`, {
      params: {
        query: promql,
        time: Math.floor(time.getTime() / 1000)
      }
    });

    if (response.data.status !== 'success') {
      throw new Error(`Prometheus query failed: ${response.data.error}`);
    }

    return response.data.data.result;
  }

  /**
   * Range query (time series)
   */
  async queryRange(promql, start, end, step = '15s') {
    const response = await axios.get(`${this.baseURL}/api/v1/query_range`, {
      params: {
        query: promql,
        start: Math.floor(start.getTime() / 1000),
        end: Math.floor(end.getTime() / 1000),
        step: step
      }
    });

    if (response.data.status !== 'success') {
      throw new Error(`Prometheus query failed: ${response.data.error}`);
    }

    return response.data.data.result;
  }

  // ============================================
  // HIGH-LEVEL QUERIES FOR SCALING DECISIONS
  // ============================================

  /**
   * Get average CPU usage for egress nodes in a tree (last 5 min)
   */
  async getAvgCPU(treeId) {
    const promql = `
      avg(
        avg_over_time(
          (1 - rate(node_cpu_seconds_total{mode="idle",tree_id="${treeId}",node_type="egress"}[1m]))[5m:]
        )
      ) * 100
    `;

    const result = await this.query(promql);
    return result.length > 0 ? parseFloat(result[0].value[1]) : 0;
  }

  /**
   * Get max port pool utilization across all egress nodes
   */
  async getMaxPortPoolUtilization(treeId) {
    const promql = `
      max(
        media_tree_egress_port_pool_utilization_percent{tree_id="${treeId}"}
      )
    `;

    const result = await this.query(promql);
    return result.length > 0 ? parseFloat(result[0].value[1]) : 0;
  }

  /**
   * Get total active viewers across all egress nodes
   */
  async getTotalViewers(treeId) {
    const promql = `
      sum(
        media_tree_egress_active_viewers{tree_id="${treeId}"}
      )
    `;

    const result = await this.query(promql);
    return result.length > 0 ? parseFloat(result[0].value[1]) : 0;
  }

  /**
   * Get network dropped packets rate (packet loss indicator)
   */
  async getDroppedPacketsRate(treeId) {
    const promql = `
      sum(
        rate(node_network_receive_drop_total{tree_id="${treeId}",node_type="egress",device!="lo"}[5m])
      )
    `;

    const result = await this.query(promql);
    return result.length > 0 ? parseFloat(result[0].value[1]) : 0;
  }

  /**
   * Get memory usage per node
   */
  async getMemoryUsage(nodeId) {
    const promql = `
      (1 - (
        node_memory_MemAvailable_bytes{node_id="${nodeId}"} /
        node_memory_MemTotal_bytes{node_id="${nodeId}"}
      )) * 100
    `;

    const result = await this.query(promql);
    return result.length > 0 ? parseFloat(result[0].value[1]) : 0;
  }

  /**
   * Get active sessions count
   */
  async getActiveSessions(treeId) {
    const promql = `
      sum(
        media_tree_injection_active_sessions{tree_id="${treeId}"}
      )
    `;

    const result = await this.query(promql);
    return result.length > 0 ? parseFloat(result[0].value[1]) : 0;
  }

  /**
   * Get RTP throughput (bytes/sec)
   */
  async getRTPThroughput(nodeId, streamType = 'video') {
    const promql = `
      rate(media_tree_egress_rtp_packets_received_total{node_id="${nodeId}",stream_type="${streamType}"}[1m])
    `;

    const result = await this.query(promql);
    return result.length > 0 ? parseFloat(result[0].value[1]) : 0;
  }

  /**
   * Check if any egress node is unhealthy (CPU >90% or Memory >90%)
   */
  async getUnhealthyNodes(treeId) {
    const promql = `
      (
        (1 - rate(node_cpu_seconds_total{mode="idle",tree_id="${treeId}",node_type="egress"}[1m])) * 100 > 90
        or
        (1 - (node_memory_MemAvailable_bytes{tree_id="${treeId}",node_type="egress"} / node_memory_MemTotal_bytes{tree_id="${treeId}",node_type="egress"})) * 100 > 90
      )
    `;

    const result = await this.query(promql);
    return result.map(r => r.metric.node_id);
  }
}

module.exports = PrometheusClient;
```

---

## 🧠 SCALING CONTROLLER: Decision Engine

```javascript
// scaling-controller/decision-engine.js
const PrometheusClient = require('./prometheus-client');

class DecisionEngine {
  constructor(redis, config = {}) {
    this.redis = redis;
    this.prometheus = new PrometheusClient(config.prometheusURL);

    // Thresholds
    this.thresholds = {
      cpu: {
        scaleOut: config.cpuScaleOut || 70,
        scaleIn: config.cpuScaleIn || 20
      },
      memory: {
        scaleOut: config.memoryScaleOut || 80,
        scaleIn: config.memoryScaleIn || 30
      },
      portPool: {
        scaleOut: config.portPoolScaleOut || 85,
        scaleIn: config.portPoolScaleIn || 30
      },
      droppedPackets: {
        scaleOut: config.droppedPacketsScaleOut || 100  // pkt/s
      }
    };
  }

  /**
   * Evaluate scaling decision for a tree
   */
  async evaluate(treeId) {
    console.log(`[DecisionEngine] Evaluating tree: ${treeId}`);

    // Query metrics
    const [
      avgCPU,
      maxPortPool,
      totalViewers,
      droppedPackets,
      activeSessions,
      unhealthyNodes
    ] = await Promise.all([
      this.prometheus.getAvgCPU(treeId),
      this.prometheus.getMaxPortPoolUtilization(treeId),
      this.prometheus.getTotalViewers(treeId),
      this.prometheus.getDroppedPacketsRate(treeId),
      this.prometheus.getActiveSessions(treeId),
      this.prometheus.getUnhealthyNodes(treeId)
    ]);

    console.log(`[DecisionEngine] Metrics:`, {
      avgCPU: avgCPU.toFixed(1),
      maxPortPool: maxPortPool.toFixed(1),
      totalViewers,
      droppedPackets,
      activeSessions,
      unhealthyNodes
    });

    // CRITICAL: Unhealthy nodes
    if (unhealthyNodes.length > 0) {
      return {
        action: 'SCALE_OUT',
        reason: `Unhealthy nodes detected: ${unhealthyNodes.join(', ')}`,
        urgency: 'CRITICAL',
        metrics: { unhealthyNodes }
      };
    }

    // SCALE-OUT conditions
    if (avgCPU > this.thresholds.cpu.scaleOut) {
      return {
        action: 'SCALE_OUT',
        reason: `High CPU usage (${avgCPU.toFixed(1)}%)`,
        urgency: 'HIGH',
        metrics: { avgCPU, maxPortPool, totalViewers }
      };
    }

    if (maxPortPool > this.thresholds.portPool.scaleOut) {
      return {
        action: 'SCALE_OUT',
        reason: `Port pool near exhaustion (${maxPortPool.toFixed(1)}%)`,
        urgency: 'CRITICAL',
        metrics: { avgCPU, maxPortPool, totalViewers }
      };
    }

    if (droppedPackets > this.thresholds.droppedPackets.scaleOut) {
      return {
        action: 'SCALE_OUT',
        reason: `Packet loss detected (${droppedPackets.toFixed(1)} pkt/s)`,
        urgency: 'HIGH',
        metrics: { droppedPackets }
      };
    }

    // SCALE-IN conditions
    const egressCount = await this.redis.scard(`nodes:${treeId}:egress`);

    if (
      avgCPU < this.thresholds.cpu.scaleIn &&
      maxPortPool < this.thresholds.portPool.scaleIn &&
      egressCount > 8  // Non scale-in sotto baseline
    ) {
      return {
        action: 'SCALE_IN',
        reason: `Low resource usage (CPU ${avgCPU.toFixed(1)}%, PortPool ${maxPortPool.toFixed(1)}%)`,
        urgency: 'LOW',
        metrics: { avgCPU, maxPortPool, egressCount }
      };
    }

    // NOOP
    return {
      action: 'NOOP',
      reason: 'Metrics within normal range',
      metrics: { avgCPU, maxPortPool, totalViewers, activeSessions }
    };
  }
}

module.exports = DecisionEngine;
```

---

## 🔄 COMPLETE FLOW: End-to-End

```
┌─────────────────────────────────────────────────────────────────┐
│  STEP 1: Metrics Collection (ogni 15s)                         │
└─────────────────────────────────────────────────────────────────┘

1a. Prometheus scrape node_exporter:9100
    → CPU, memory, network, disk metrics

1b. Prometheus scrape app:7070/metrics, app:7071/metrics, app:7073/metrics
    → Sessions, mountpoints, port pool, RTP stats

1c. Prometheus store time-series in local TSDB


┌─────────────────────────────────────────────────────────────────┐
│  STEP 2: Scaling Evaluation (ogni 30s)                         │
└─────────────────────────────────────────────────────────────────┘

2a. Scaling Controller query Prometheus:
    - avgCPU = query("avg(node_cpu_usage{tree_id='tree-1'})")
    - maxPortPool = query("max(port_pool_utilization{tree_id='tree-1'})")
    - droppedPackets = query("rate(network_drops{tree_id='tree-1'})")

2b. Decision Engine evaluate thresholds:
    if (avgCPU > 70% || maxPortPool > 85%) {
      decision = "SCALE_OUT"
    }


┌─────────────────────────────────────────────────────────────────┐
│  STEP 3: Orchestration (if SCALE_OUT)                          │
└─────────────────────────────────────────────────────────────────┘

3a. Find available relay-slot:
    relaySlot = redis.smembers("relay-slots:tree-1:available")[0]

3b. Provision sub-tree via K8s API:
    k8s.createDeployment({
      name: "egress-subtree-123",
      replicas: 16,
      image: "media-tree/egress:v1.0"
    })

3c. Wait for pods ready:
    k8s.waitForPods(deployment, timeout: 60s)

3d. Update Redis topology:
    pods.forEach(pod => {
      redis.set(`parent:${pod}`, relaySlot)
      redis.sadd(`children:${relaySlot}`, pod)
    })

3e. Publish event:
    redis.publish(`topology:${relaySlot}`, {
      type: "children-added",
      childIds: [pod1, pod2, ..., pod16]
    })


┌─────────────────────────────────────────────────────────────────┐
│  STEP 4: Node Reconfiguration (real-time)                      │
└─────────────────────────────────────────────────────────────────┘

4a. RelayNode riceve Pub/Sub event "children-added"
    → Legge info pods da Redis (host, porte)
    → Invia comandi al forwarder C:
        ADD egress-subtree-123-0 5002 5004
        ADD egress-subtree-123-1 5002 5004
        ...
    → GStreamer aggiunge destinations a multiudpsink
    → RTP packets iniziano a fluire ai nuovi egress

4b. EgressNode (nuovi pods) all'avvio:
    → discoverExistingSessions() legge da Redis
    → Per ogni sessione attiva:
        - Alloca porte dal PortPool
        - Crea Janus Streaming Mountpoint
        - Crea WHEP endpoint
        - Invia ADD al forwarder C (mapping SSRC → porte Janus)
    → RTP demux inizia, mountpoints pronti per viewer


┌─────────────────────────────────────────────────────────────────┐
│  STEP 5: Monitoring (continuous)                               │
└─────────────────────────────────────────────────────────────────┘

5a. Nuovi egress nodes espongono:
    - node_exporter:9100 (system metrics)
    - app:7073/metrics (mountpoints, port pool, viewers)

5b. Prometheus auto-discover nuovi target (K8s service discovery)
    → Inizia scrape metrics dai nuovi nodi

5c. Grafana dashboard aggiorna automaticamente:
    - Total viewer count aumenta
    - Port pool utilization scende (nuova capacity)
    - CPU avg si riequilibra

5d. Slack notification:
    "✅ Scaled out: tree-1 → +16 egress nodes, +16K capacity"
```

---

## 📈 GRAFANA DASHBOARD QUERIES

### Panel: CPU Usage per Node

```promql
(1 - rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100
```

**Visualization**: Time series line chart
**Legend**: `{{node_id}}`

---

### Panel: Memory Usage per Node

```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

---

### Panel: Network Throughput (RTP)

```promql
# Receive
rate(node_network_receive_bytes_total{device="eth0"}[1m])

# Transmit
rate(node_network_transmit_bytes_total{device="eth0"}[1m])
```

**Unit**: Bytes/sec
**Visualization**: Graph with dual Y-axis

---

### Panel: Port Pool Utilization

```promql
media_tree_egress_port_pool_utilization_percent
```

**Thresholds**:
- Green: <70%
- Yellow: 70-85%
- Red: >85%

---

### Panel: Active Viewers (Total)

```promql
sum(media_tree_egress_active_viewers)
```

**Visualization**: Single stat

---

### Panel: Active Sessions

```promql
sum(media_tree_injection_active_sessions)
```

---

### Panel: Packet Loss Rate

```promql
rate(node_network_receive_drop_total{device!="lo"}[5m])
```

**Alert**: >100 pkt/s → Critical

---

## 🚨 ALERTING RULES

```yaml
# prometheus-alerts.yml
groups:
  - name: media_tree_alerts
    interval: 30s
    rules:
      # High CPU
      - alert: HighCPUUsage
        expr: |
          (1 - rate(node_cpu_seconds_total{mode="idle",node_type="egress"}[1m])) * 100 > 80
        for: 5m
        labels:
          severity: warning
          component: scaling
        annotations:
          summary: "High CPU on {{ $labels.node_id }}"
          description: "CPU usage is {{ $value | humanize }}% on {{ $labels.node_id }}"

      # Port pool near exhaustion
      - alert: PortPoolExhaustion
        expr: |
          media_tree_egress_port_pool_utilization_percent > 85
        for: 2m
        labels:
          severity: critical
          component: scaling
        annotations:
          summary: "Port pool near exhaustion on {{ $labels.node_id }}"
          description: "Utilization: {{ $value | humanize }}%"

      # Packet loss
      - alert: PacketLoss
        expr: |
          rate(node_network_receive_drop_total{device!="lo"}[5m]) > 100
        for: 3m
        labels:
          severity: warning
          component: network
        annotations:
          summary: "Packet loss on {{ $labels.node_id }}"
          description: "Dropping {{ $value | humanize }} packets/sec"

      # Forwarder pipeline down
      - alert: ForwarderPipelineDown
        expr: |
          media_tree_relay_forwarder_state < 3
        for: 1m
        labels:
          severity: critical
          component: relay
        annotations:
          summary: "Forwarder pipeline not PLAYING on {{ $labels.node_id }}"
          description: "Pipeline state: {{ $value }}"

      # No active sessions but mountpoints exist
      - alert: OrphanMountpoints
        expr: |
          media_tree_injection_active_sessions == 0 and media_tree_egress_active_mountpoints > 0
        for: 10m
        labels:
          severity: info
          component: cleanup
        annotations:
          summary: "Orphan mountpoints detected"
          description: "{{ $value }} mountpoints exist but no active sessions"
```

---

## 🎯 SUMMARY: Ogni Nodo Espone

### INJECTION NODE
```
:9100/metrics → node_exporter (CPU, RAM, network, disk)
:7070/metrics → App (sessions, RTP packets/bytes, children count)
```

### RELAY NODE
```
:9100/metrics → node_exporter (CPU, RAM, network, disk)
:7071/metrics → App (destinations, RTP forwarded, pipeline state)
```

### EGRESS NODE
```
:9100/metrics → node_exporter (CPU, RAM, network, disk)
:7073/metrics → App (mountpoints, viewers, port pool, RTP packets)
```

### PROMETHEUS
```
:9090/api/v1/query → Query API per Scaling Controller
```

### GRAFANA
```
:3000 → Dashboard visualizzazione + alerting
```

---

## ✅ DEPLOYMENT CHECKLIST

- [ ] Ogni nodo ha node_exporter sidecar
- [ ] Ogni app espone `/metrics` endpoint
- [ ] Prometheus scrape config include tutti i target
- [ ] Grafana dashboard importato
- [ ] Alert rules configurati
- [ ] Slack webhook per notifiche
- [ ] Scaling Controller deployed
- [ ] Test: Simulate high load → verify scale-out
- [ ] Test: Simulate low load → verify scale-in

---

## 🎯 NEXT STEPS

1. **Implement `/metrics` endpoint** su tutti i nodi
2. **Deploy Prometheus + node_exporter** (Docker Compose)
3. **Create Grafana dashboard** (import JSON)
4. **Build Scaling Controller** (PrometheusClient + DecisionEngine)
5. **Test end-to-end** (load simulation)

Dimmi quale vuoi che implementi per primo! 🚀
