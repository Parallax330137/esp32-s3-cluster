# Protocol

custom UDP-based protocol. all cluster traffic runs over ethernet via LAN8720.

---

## Ports

| Port | Purpose |
|------|---------|
| 5000 | control messages (28-byte fixed struct) |
| 5001 | data transfer (raw binary chunks) |

broadcast IP: `192.168.1.255`

---

## Packet format

every control message is exactly 28 bytes, packed struct:

```c
typedef struct __attribute__((packed)) {
    uint8_t  src_id, dst_id, msg_type, task_type;
    uint8_t  mode, pipeline_stage, tensor_slice, total_slices;
    uint16_t task_id, chunk_id;
    uint32_t storage_addr, payload_size;
    uint32_t subtask_bitmask;
    uint8_t  subtask_total, temp_c, active_nodes, checksum;
} ClusterMsg_t;
```

checksum is a simple XOR of bytes 0-26. receiver verifies before processing.

nodes filter by dst_id — if it doesnt match their own ID and isnt 0xFF (broadcast), they discard the packet.

---

## Message types

### Task management
| Code | Name | Description |
|------|------|-------------|
| 0x10 | MSG_TASK_ASSIGN | Alpha assigns a compute chunk to a node |
| 0x11 | MSG_CHUNK_DONE | Node finished its chunk |
| 0x12 | MSG_CHUNK_HOT | Node is overheating, partial work returned |
| 0x13 | MSG_MORE_WORK | Alpha asking if node wants another chunk |

### Power / sleep
| Code | Name | Description |
|------|------|-------------|
| 0x20 | MSG_SLEEP | Alpha tells node to sleep |
| 0x21 | MSG_WAKE | Alpha wakes a sleeping node |
| 0x22 | MSG_MODE_CHANGE | Alpha changes compute mode |

### Temperature / clock
| Code | Name | Description |
|------|------|-------------|
| 0x30 | MSG_TEMP_REPORT | Node reports its temperature |
| 0x31 | MSG_CLOCK_SET | Alpha broadcasts new clock speed |

### Storage
| Code | Name | Description |
|------|------|-------------|
| 0x70 | MSG_STORE_REQ | Node requests data from Alpha storage |
| 0x71 | MSG_STORE_LOCK | Alpha locks the bus (all traffic stops) |
| 0x72 | MSG_STORE_DATA | Alpha sends data chunk over PORT_DATA |
| 0x73 | MSG_STORE_DONE | Transfer complete |
| 0x74 | MSG_STORE_UNLOCK | Alpha unlocks the bus |
| 0x75 | MSG_STORE_WRITE | Node writing results back to Alpha |
| 0x76 | MSG_STORE_WDONE | Write complete |

### Bus arbitration
| Code | Name | Description |
|------|------|-------------|
| 0x80 | MSG_RTS | Node requests the bus |
| 0x81 | MSG_CTS | Alpha grants the bus |
| 0x82 | MSG_BUS_RELEASE | Node done, releasing bus |

### ACK / handshake
| Code | Name | Description |
|------|------|-------------|
| 0xA0 | MSG_ACK | Message acknowledged |
| 0xA1 | MSG_NAK | Message rejected |
| 0xB0 | MSG_PING | Alpha pinging a node |
| 0xB1 | MSG_PONG | Node responding to ping |

---

## Bus arbitration (RTS/CTS)

this is basically a software traffic cop runing on Alpha. prevents nodes from all transmitting at once.

```
node wants to send → sends MSG_RTS to Alpha
Alpha checks priority queue (1→2→3→4)
Alpha sends MSG_CTS to the winning node
node drains its queue, each message waits for MSG_ACK
node sends MSG_BUS_RELEASE when done
Alpha rotates that node to back of the priority list
```

STORE_LOCK halts everything including RTS until STORE_UNLOCK is broadcast. used during large data transfers to prevent collision.

ACK timeout:
- real hardware: 2000ms
- Wokwi sim: 10000ms (manual tab switching)

if no ACK received, node retransmits once then logs an error and continues.

---

## Boot sequence

1. Alpha boots, prints node directory to serial
2. Alpha pings each node up to 3 times (5s timeout per attempt on real HW)
3. nodes that respond get marked as paired
4. clock tier set based on how many nodes paired
5. ready for RUN commands

---

## Node lifecycle

nodes dont just sit idle waiting for work. after boot they go to sleep until Alpha calls them specifically.

```
node boots
→ initializes ethernet, listens on PORT_CTRL
→ enters idle sleep (powered but sleeping, ethernet still active)
→ waits for a call from Alpha

Alpha sends a call to this node
→ node wakes up
→ asks Alpha: what mode?
→ Alpha replies with mode (HEAVY / SPEED / EFFICIENT)

if EFFICIENT and Alpha didnt call this node → stays asleep, nothing happens

if node was called:
→ node negotiates mode
→ receives second code from Alpha
→ executes second code
→ sends results back
→ returns to idle sleep
```

the mode logic (HEAVY, SPEED, EFFICIENT) is built into the node firmware itself — it knows what each mode means and how to behave. the second code is what actually runs on top of that.

---

## Second code

the second code is the actual task payload sent to a node after mode is established. it is not a replacement for the built-in mode/task logic — it runs on top of it.

think of it this way:
- **base firmware** (in flash): handles networking, sleep/wake, mode negotiation, RTS/CTS, thermal throttling. permanent, only changes via OTA.
- **second code** (received at runtime): the actual computation or job to run this session. loaded into PSRAM, executed, discarded when done.

this means the cluster is reprogrammable without reflashing. Alpha sends different second code depending on what job needs doing. nodes dont need to know in advance what they'll be running.

### How Alpha gets the second code

either from a PC or phone, whichever is connected at the time:
- PC: sent to Alpha over USB serial, or fetched over WiFi
- phone: same options

Alpha stores it temporarily then distributes to the relevant nodes over the existing ethernet protocol (PORT_DATA, chunked the same way as STORE_DATA transfers).

### What second code can be

anything that fits within the node's PSRAM budget and doesnt require OS-level features. compute tasks are the obvious use — matrix ops, signal processing, inference layers, custom algorithms. the base firmware provides the runtime environment, second code provides the logic.

---

## Compute modes

| Mode | Description |
|------|-------------|
| HEAVY | pipeline + tensor, all paired nodes active |
| SPEED | tensor only, all paired nodes active |
| EFFICIENT | only the nodes Alpha specifically calls are woken, rest stay asleep |

**efficient mode note:** in efficient mode Alpha decides per-job which nodes to call. nodes that dont get a call are never woken — they stay in idle sleep even though they're still powered and listening. this saves power when the workload doesnt need all 4 workers.

---

## Task types

| Code | Name | Description |
|------|------|-------------|
| 0x01 | TASK_MATH_POLY | polynomial evaluation |
| 0x02 | TASK_MATH_STATS | mean, stddev, statistics |
| 0x10 | TASK_TENSOR_MUL | tensor / matrix multiplication |
| 0x20 | TASK_PIPELINE | staged processing pipeline |

each task chunk is 512KB. task IDs and storage addresses are tracked in a manifest on Alpha.

---

## Serial commands (Alpha)

```
-- startup --
HANDSHAKE              re-run node discovery
SIM PONG <n>           fake node n responding (sim only)
SIM HANDSHAKE          all nodes respond instantly (sim only)

-- run modes --
RUN HEAVY
RUN SPEED
RUN EFFICIENT

-- test tasks --
TEST POLY <n>
TEST TENSOR <n>
TEST PIPELINE <n>
TEST STATS <n>
TEST ALL

-- simulate node messages --
SIM RTS <n>
SIM DONE <n>
SIM RELEASE <n>
SIM HOT <n> <temp>
SIM TEMP <n> <temp>
SIM STORE <n>

-- info --
STATUS
NODES
TASKS

-- control --
LOCK
UNLOCK
RESET
```
