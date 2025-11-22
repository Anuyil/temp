# OBS Integration Guide

## 🎯 Obiettivo
Usare **OBS Studio** come broadcaster invece del WHIP test client.

---

## 📋 Opzioni Disponibili

### **Opzione A: OBS con WHIP Plugin (RACCOMANDATO)**

#### Requisiti
- OBS Studio 30+ (supporto WHIP nativo)
- Oppure plugin `obs-whip` per versioni precedenti

#### Pro
✅ **Zero latency aggiuntivo** (WebRTC nativo)
✅ **Nessun transcoding** (direct RTP forwarding)
✅ **Compatibile con architettura esistente**
✅ **Produzione-ready**

#### Contro
❌ Richiede OBS 30+ o plugin separato
❌ WHIP spec ancora in evoluzione (draft IETF)

---

### **Opzione B: OBS → RTMP → Janus (FALLBACK)**

#### Pro
✅ **Compatibile con TUTTE le versioni OBS**
✅ **Setup familiare** (RTMP è standard)
✅ **Zero configurazione OBS complessa**

#### Contro
❌ **Aggiunge 1-3 secondi di latency** (transcoding)
❌ Richiede componente bridge (ffmpeg/gstreamer)
❌ CPU overhead per transcoding

---

### **Opzione C: OBS → SRT → Janus (ALTERNATIVA)**

#### Pro
✅ Low-latency (~500ms vs 1-3s RTMP)
✅ Error correction nativo (retransmissioni automatiche)
✅ Firewall-friendly

#### Contro
❌ Richiede SRT plugin in OBS
❌ Serve bridge SRT → RTP
❌ Più complesso di RTMP

---

## 🏆 RACCOMANDAZIONE: Opzione A (WHIP)

### Architettura

```
┌──────────────────┐
│   OBS Studio     │
│  (Broadcaster)   │
│                  │
│  Sources:        │
│  - Camera        │
│  - Screen share  │
│  - Overlay       │
└────────┬─────────┘
         │ WHIP (HTTP POST)
         │ SDP Offer/Answer
         │
         ▼
┌─────────────────────────────────────┐
│     InjectionNode :7070             │
│                                     │
│  ┌──────────────────────────────┐  │
│  │   janus-whip-server          │  │
│  │   /whip/endpoint/{sessionId} │  │
│  └──────────┬───────────────────┘  │
│             │                       │
│             ▼                       │
│  ┌──────────────────────────────┐  │
│  │  Janus VideoRoom             │  │
│  │  Room: {roomId}              │  │
│  │  Publisher: OBS              │  │
│  └──────────┬───────────────────┘  │
│             │                       │
│             │ RTP Forwarding        │
│             │ SSRC: audio/video     │
└─────────────┼───────────────────────┘
              │
              ▼
       [Relay Nodes...]
```

---

## 🛠️ IMPLEMENTAZIONE

### Step 1: Verifica WHIP Endpoint Esistente

Il tuo `InjectionNode` già espone WHIP endpoint tramite `janus-whip-server`:

```javascript
// injection-node/src/index.js (ESISTENTE)
const whipServer = new WhipServer({
  janusConnection: janodeConnection,
  port: 7070,
  basePath: '/whip/endpoint'
});

// Endpoint disponibile:
// POST http://injection-node:7070/whip/endpoint/{sessionId}
```

**✅ Nessuna modifica necessaria al server!**

---

### Step 2: Setup OBS

#### 2.1 Installa OBS WHIP Plugin

**Per OBS 30+** (built-in):
- Nessuna installazione necessaria

**Per OBS <30** (plugin separato):
```bash
# Linux
mkdir -p ~/.config/obs-studio/plugins
cd ~/.config/obs-studio/plugins
git clone https://github.com/obsproject/obs-whip-output.git
cd obs-whip-output
mkdir build && cd build
cmake .. && make
sudo make install

# macOS
brew install obs-whip-output

# Windows
# Scarica da: https://github.com/obsproject/obs-whip-output/releases
# Estrai in: C:\Program Files\obs-studio\obs-plugins\64bit\
```

#### 2.2 Configura OBS Stream Settings

1. **Apri OBS Studio**
2. **Settings → Stream**

```
┌─────────────────────────────────────┐
│ Stream Settings                     │
├─────────────────────────────────────┤
│ Service: [WHIP                    ▼]│
│                                     │
│ Server:                             │
│ http://YOUR_IP:7070/whip/endpoint/  │
│ my-session-123                      │
│                                     │
│ Bearer Token: (leave empty)         │
│                                     │
│ [✓] Use authentication              │
│ [ ] Custom codec settings           │
└─────────────────────────────────────┘
```

**IMPORTANTE**: Sostituisci:
- `YOUR_IP` → IP del server InjectionNode
- `my-session-123` → sessionId che vuoi usare

#### 2.3 Configura Output Settings

**Settings → Output → Streaming**

```
┌─────────────────────────────────────┐
│ Output Mode: [Advanced            ▼]│
├─────────────────────────────────────┤
│ ENCODER (Video):                    │
│ ┌─────────────────────────────────┐ │
│ │ Encoder: libx264 / H.264        │ │
│ │ Rate Control: CBR               │ │
│ │ Bitrate: 2500 Kbps              │ │
│ │ Keyframe Interval: 2s           │ │
│ │ CPU Preset: veryfast            │ │
│ │ Profile: baseline               │ │
│ └─────────────────────────────────┘ │
│                                     │
│ ENCODER (Audio):                    │
│ ┌─────────────────────────────────┐ │
│ │ Encoder: Opus                   │ │
│ │ Bitrate: 128 Kbps               │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
```

**Note**:
- **H.264 baseline profile**: Massima compatibilità
- **CBR (Constant Bitrate)**: Prevedibile per streaming
- **Keyframe ogni 2s**: Riduce latency (default è 10s)
- **Opus audio**: Standard WebRTC

---

### Step 3: Crea Sessione nel Sistema

Prima di avviare OBS, devi **creare la sessione** nel tuo sistema:

```bash
# API call per creare sessione
curl -X POST http://injection-node:7070/session \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "my-session-123",
    "roomId": "room-456",
    "audioSsrc": 111111,
    "videoSsrc": 222222
  }'

# Response:
# {
#   "success": true,
#   "whipEndpoint": "http://injection-node:7070/whip/endpoint/my-session-123"
# }
```

**Questo crea**:
1. Janus VideoRoom con roomId `room-456`
2. WHIP endpoint `/whip/endpoint/my-session-123`
3. RTP forwarding ai children configurati in Redis

---

### Step 4: Start Streaming da OBS

1. **Apri OBS**
2. **Configura scene** (camera, desktop, overlay, ecc.)
3. **Click "Start Streaming"**

**Cosa succede dietro le quinte**:

```
1. OBS → HTTP POST /whip/endpoint/my-session-123
   Body: SDP Offer (video H.264, audio Opus)

2. janus-whip-server riceve offer
   → Forward a Janus VideoRoom

3. Janus VideoRoom crea PeerConnection
   → Genera SDP Answer

4. janus-whip-server → HTTP 201 Created
   Body: SDP Answer
   Location: http://injection-node:7070/whip/resource/{resourceId}

5. OBS completa ICE negotiation
   → WebRTC connection ESTABLISHED

6. OBS inizia a inviare RTP packets

7. Janus riceve RTP packets
   → Forward ai children Relay/Egress
   → Demux per SSRC
   → Viewer ricevono stream
```

---

### Step 5: Monitoring

```bash
# Check se sessione è attiva
curl http://injection-node:7070/sessions

# Response:
# {
#   "sessions": [
#     {
#       "sessionId": "my-session-123",
#       "roomId": "room-456",
#       "active": true,
#       "viewerCount": 42
#     }
#   ]
# }
```

---

## 🐛 TROUBLESHOOTING

### Problema: "Connection failed"

**Causa**: Firewall blocca porte WebRTC

**Soluzione**:
```bash
# Apri porte UDP per ICE candidates
sudo ufw allow 10000:20000/udp

# O configura TURN server
```

---

### Problema: "No video/audio in output"

**Causa**: Codec mismatch

**Check**:
```bash
# Verifica codec supportati da Janus
curl http://injection-node:8088/janus/info

# Assicurati che OBS usi:
# - Video: H.264 (baseline/main)
# - Audio: Opus
```

---

### Problema: "High latency (>5s)"

**Causa**: Keyframe interval troppo alto

**Soluzione**:
- OBS Settings → Output → Keyframe Interval: **2 secondi**
- Riduci jitter buffer in Janus Streaming plugin

---

## 📊 ALTERNATIVA: RTMP Bridge (se WHIP non funziona)

### Architettura

```
OBS → RTMP → nginx-rtmp → ffmpeg → RTP → Janus VideoRoom
```

### Setup Rapido

```yaml
# docker-compose.yml
services:
  nginx-rtmp:
    image: tiangolo/nginx-rtmp
    ports:
      - "1935:1935"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf

  rtmp-bridge:
    image: jrottenberg/ffmpeg:latest
    command: >
      -listen 1 -i rtmp://nginx-rtmp:1935/live/my-session-123
      -c:v copy -c:a aac
      -f rtp rtp://injection-node:5004?rtcpport=5005
      -f rtp rtp://injection-node:5002?rtcpport=5003
    restart: always
```

**OBS Configuration**:
```
Service: Custom
Server: rtmp://YOUR_IP:1935/live
Stream Key: my-session-123
```

**Pro**: Funziona al 100% con qualsiasi OBS
**Contro**: +1-3s latency, CPU overhead

---

## ✅ CHECKLIST FINALE

Prima di andare in produzione con OBS:

- [ ] WHIP endpoint funzionante (`POST /whip/endpoint/{sessionId}`)
- [ ] Sessione creata in Redis (`session:{sessionId}`)
- [ ] Janus VideoRoom creato
- [ ] RTP forwarding configurato
- [ ] OBS plugin WHIP installato
- [ ] Codec settings corretti (H.264 baseline, Opus)
- [ ] Keyframe interval = 2s
- [ ] Firewall porte aperte (7070, 10000-20000/udp)
- [ ] Test connessione: OBS → Injection → Egress → Viewer

---

## 🎬 DEMO FLOW COMPLETO

```bash
# 1. Crea sessione
curl -X POST http://localhost:7070/session -d '{
  "sessionId": "obs-demo-001",
  "roomId": "room-001",
  "audioSsrc": 111111,
  "videoSsrc": 222222
}'

# 2. Configura OBS
# Settings → Stream → WHIP
# Server: http://localhost:7070/whip/endpoint/obs-demo-001

# 3. Start Streaming in OBS

# 4. Verifica viewer può connettersi
# WHEP endpoint su egress node:
# http://egress-1:7073/whep/endpoint/obs-demo-001

# 5. Monitor
curl http://localhost:7070/metrics
```

**Aspettativa**: Latency glass-to-glass <500ms 🚀
