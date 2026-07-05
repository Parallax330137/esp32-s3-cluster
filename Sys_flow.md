# Communication Flows

detailed message sequences for every interaction in the cluster. read alongside protocol.md for message type definitions.

---

## Node startup / wake flow

```
node powers on
│
├── initialize LAN8720 over RMII
├── open UDP socket on PORT_CTRL (5000)
├── open UDP socket on PORT_DATA (5001)
│
└── enter idle sleep
    │
    └── [listening on PORT_CTRL, waiting for Alpha]
        │
        Alpha sends MSG_PING
        │
        node wakes up
        ├── sends MSG_PONG to Alpha
        │     └── includes: temp_c, NODE_ID
        │
        Alpha marks node as paired
        Alpha sends MSG_MODE_CHANGE
        │
        node reads mode:
        ├── HEAVY   → prepare full pipeline + tensor
        ├── SPEED   → prepare tensor only
        └── EFFICIENT → only woken nodes proceed, rest go back to sleep
            │
            [if this node was called]
            │
            node ready, waits for task assignment
            │
            receives second code from Alpha over PORT_DATA
            │
            executes second code
            │
            returns result to Alpha
            │
            returns to idle sleep ──────────────────────────┐
                                                             │
                                    [stays here until next call]
```

---

## Alpha to node

### task assignment
```
Alpha                              Node (target)
  │                                    │
  ├── MSG_TASK_ASSIGN ───────────────> │
  │     dst_id: target node            │
  │     task_type: POLY/TENSOR/etc     │
  │     task_id, chunk_id              │
  │     payload_size                   │
  │                                    ├── ACK task
  │ <── MSG_ACK ──────────────────────┤
  │                                    ├── execute task
  │                                    │
  │ <── MSG_CHUNK_DONE ───────────────┤
  │     task_id, chunk_id              │
  │     subtask_bitmask (what finished)│
  │                                    │
  ├── [if more work] MSG_MORE_WORK ──> │
  │   [if done]      MSG_SLEEP ──────> │
```

### sleep / wake
```
Alpha                              Node
  │                                    │
  ├── MSG_SLEEP ────────────────────> │  node enters low power sleep
  │                                    │
  [later]
  │                                    │
  ├── MSG_WAKE ─────────────────────> │  node wakes, re-initializes
  │ <── MSG_PONG ─────────────────────┤  confirms alive
```

### clock change (broadcast)
```
Alpha                         All Nodes
  │                                │
  ├── MSG_CLOCK_SET ─────────────> │  (sent to 192.168.1.255)
  │     new clock speed            │
  │                                ├── each node adjusts cpu freq
  │                                ├── sends MSG_ACK individually
  │ <── MSG_ACK (×4) ─────────────┤
```

---

## Node to Alpha

### bus request (RTS/CTS)
```
Node                               Alpha
  │                                    │
  ├── MSG_RTS ────────────────────> │  "i want to send something"
  │                                    │
  │                         Alpha checks priority queue
  │                         grants or queues the request
  │                                    │
  │ <── MSG_CTS ──────────────────────┤  "go ahead"
  │                                    │
  ├── [sends queued messages] ──────> │
  │ <── MSG_ACK (per message) ────────┤
  │                                    │
  ├── MSG_BUS_RELEASE ─────────────> │  "done"
  │                                    │
  │                         Alpha rotates node to back of queue
```

### temperature report
```
Node                               Alpha
  │                                    │
  ├── MSG_TEMP_REPORT ─────────────> │
  │     temp_c: current temp           │
  │                                    │
  │                         if temp >= 75°C:
  │ <── MSG_SLEEP ────────────────────┤  forced thermal sleep
  │                                    │
  │                         Alpha adjusts clock tier
  │ <── MSG_CLOCK_SET ────────────────┤  (broadcast to all)
```

### chunk overheating mid-task
```
Node                               Alpha
  │                                    │
  │  [running task, temp spikes]       │
  │                                    │
  ├── MSG_CHUNK_HOT ───────────────> │
  │     partial subtask_bitmask        │
  │     temp_c                         │
  │                                    │
  │ <── MSG_SLEEP ────────────────────┤
  │                                    │
  │                         Alpha redistributes unfinished chunks
  │                         to remaining active nodes
```

---

## Node to node

nodes do not communicate directly with each other. all traffic goes through Alpha.

```
Node A  ──────>  Alpha  ──────>  Node B
```

if Node A needs data that Node B computed, Alpha holds that data and sends it to Node A on request. this is intentional — Alpha maintains full visibility of all cluster state.

---

## SD card / storage request

> **NOT DONE — planned feature, not yet implemented**

flow for when a node needs to fetch data from Alpha's SD card storage.

```
Node                               Alpha                        All Nodes
  │                                    │                              │
  ├── MSG_STORE_REQ ───────────────> │                              │
  │     storage_addr                   │                              │
  │     payload_size                   │                              │
  │                                    │                              │
  │                         Alpha locks the bus                      │
  │                         ├── MSG_STORE_LOCK ──────────────────> │
  │                         │   (broadcast, 192.168.1.255)          │
  │                         │                           all nodes pause RTS
  │                         │                                        │
  │                         Alpha reads from SD card                 │
  │                         │                                        │
  │ <── MSG_STORE_DATA ──────┤  (chunk 1, over PORT_DATA)           │
  ├── MSG_ACK ─────────────> │                                      │
  │ <── MSG_STORE_DATA ──────┤  (chunk 2)                           │
  ├── MSG_ACK ─────────────> │                                      │
  │     [repeat for all chunks]                                      │
  │                                    │                              │
  │ <── MSG_STORE_DONE ───────────────┤                              │
  ├── MSG_ACK ─────────────────────> │                              │
  │                                    │                              │
  │                         Alpha unlocks the bus                    │
  │                         ├── MSG_STORE_UNLOCK ─────────────────> │
  │                         │   (broadcast)                          │
  │                         │                           all nodes resume normal
```

**write back (node → Alpha storage):**

> **NOT DONE**

```
Node                               Alpha
  │                                    │
  ├── MSG_STORE_WRITE ─────────────> │
  │     storage_addr (destination)     │
  │     payload_size                   │
  │                                    │
  │                         Alpha sends STORE_LOCK (broadcast)
  │                                    │
  ├── [raw data chunks over PORT_DATA]>│
  │ <── MSG_ACK (per chunk) ──────────┤
  │                                    │
  ├── MSG_STORE_WDONE ─────────────> │
  │                                    │
  │                         Alpha sends STORE_UNLOCK (broadcast)
```

---

## OTA firmware update flow

> **NOT DONE — see ota_plan.md**

```
PC/Phone                  Alpha                         Target Node
  │                          │                                │
  ├── sends .bin ──────────> │                                │
  │   (USB serial or WiFi)   │                                │
  │                          ├── MSG_OTA_BEGIN ─────────────> │
  │                          │     total_size                 │
  │                          │                                ├── ACK
  │                          │                                ├── esp_ota_begin()
  │                          │                                │
  │                          ├── MSG_OTA_CHUNK (×N) ────────> │
  │                          │     chunk_id, data             ├── esp_ota_write()
  │                          │ <── MSG_ACK ──────────────────┤
  │                          │     [repeat]                   │
  │                          │                                │
  │                          ├── MSG_OTA_END ──────────────> │
  │                          │                                ├── esp_ota_end()
  │                          │                                ├── set boot partition
  │                          │                                ├── reboot
  │                          │                                │
  │                          │ <── MSG_PONG ─────────────────┤  confirms new firmware alive
  │                          │                                │
  │ <── status update ───────┤
```
