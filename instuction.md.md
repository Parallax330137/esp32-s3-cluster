# How the Cluster Communicates

This document explains every message the cluster nodes send to each other, what each one means, and why it exists. Written so anyone can understand it, no prior networking knowledge needed.

## First, the basics

Think of this cluster like a small office.

- **Alpha** is the manager. It gives out work, tracks progress, and makes decisions.
- **Bravo, Charlie, Delta, Echo** are the workers. They do what Alpha says, report back when done, and ask for help when something goes wrong.

The only way they talk to each other is by sending small 28-byte messages over an ethernet cable. Every message has a type, a sender, and a destination. That's it.

There are two channels:

- **PORT 5000** — control messages (instructions, status, short replies)
- **PORT 5001** — raw data (actual chunks of work being transferred)

## The message types, explained one by one

### MSG_PING — "are you there?"

- **Who sends it:** Alpha
- **Who receives it:** a worker node
- **When:** at startup, before any work begins

Alpha just booted up and doesn't know which workers are online. It sends a PING to each node one by one. It's the simplest possible message — just "hey, you awake?"

If no reply comes back within 5 seconds, Alpha tries again. If a node doesn't respond after 3 tries, Alpha gives up on it and runs the cluster without it.

```
src_id: 0x00      ← Alpha is the sender
dst_id: 0x01      ← sending to Bravo (or 02 for Charlie, etc)
msg_type: 0xB0    ← this is a PING
everything else: 0
```

### MSG_PONG — "yes, I'm here"

- **Who sends it:** a worker node
- **Who receives it:** Alpha
- **When:** immediately after receiving a PING

The worker replies to say it is alive and ready. It also includes its current temperature so Alpha knows its starting condition before assigning any work.

```
src_id: 0x01      ← Bravo is replying
dst_id: 0x00      ← sending to Alpha
msg_type: 0xB1    ← this is a PONG
temp_c: 28        ← current temperature in celsius
everything else: 0
```

After Alpha collects all the PONGs, it knows exactly how many workers are online and what temperature they're starting at.

### MSG_MODE_CHANGE — "here's what kind of work we're doing today"

- **Who sends it:** Alpha
- **Who receives it:** worker nodes
- **When:** after handshake, before any tasks are assigned

Before sending any actual work, Alpha tells the workers what mode the cluster is running in. The mode determines how tasks are split up.

- **HEAVY** — all nodes work together, each handling a different stage of the same job (like an assembly line)
- **SPEED** — all nodes split the same job into equal slices and crunch it in parallel
- **EFFICIENT** — Alpha only wakes the nodes it actually needs. The rest stay asleep to save power

In EFFICIENT mode, nodes that don't receive this message simply stay asleep. They never know a job was running.

```
src_id: 0x00
dst_id: 0x01      ← sent to each worker individually
msg_type: 0x22    ← this is a MODE_CHANGE
mode: 0x01        ← 0x01=HEAVY, 0x02=SPEED, 0x03=EFFICIENT
```

The worker replies with MSG_ACK to confirm it received the mode.

### MSG_ACK — "got it"

- **Who sends it:** whoever just received a message
- **Who receives it:** whoever sent that message
- **When:** after almost every message, in both directions

ACK stands for "acknowledgement". It's just the receiver saying "I got your message and understood it". It includes a copy of the message type it is acknowledging so the sender knows exactly which one is being confirmed.

Without ACKs, Alpha would have no way to know if a message actually arrived. If an ACK doesn't come back within the timeout, the sender retries.

```
src_id: 0x01      ← whoever is acknowledging
dst_id: 0x00      ← whoever sent the original message
msg_type: 0xA0    ← this is an ACK
task_type: 0x22   ← echoing back: "I'm ACKing your MODE_CHANGE"
```

### MSG_NAK — "something went wrong"

- **Who sends it:** a worker node
- **Who receives it:** Alpha
- **When:** if a node receives something it can't process or a checksum fails

The opposite of ACK. If a message arrives corrupted or a node can't handle the request right now, it sends NAK. Alpha treats this as a failed delivery and can retry or skip.

```
src_id: 0x01
dst_id: 0x00
msg_type: 0xA1    ← this is a NAK
task_type: 0x10   ← "I'm rejecting your TASK_ASSIGN"
```

### MSG_TASK_ASSIGN — "here is your job"

- **Who sends it:** Alpha
- **Who receives it:** a specific worker node
- **When:** whenever a worker is idle and there is work to do

This is the main instruction. Alpha is telling a worker: go do this specific piece of computation. The message includes everything the node needs to know — what type of task, where to find the input data, how big the data is, and which part of the job this node specifically is responsible for.

Alpha always picks the worker with the fewest completed chunks so the workload stays balanced.

```
src_id: 0x00
dst_id: 0x02              ← sending to Charlie for example
msg_type: 0x10            ← this is a TASK_ASSIGN
task_type: 0x10           ← type of work: 0x01=POLY, 0x02=STATS,
                             0x10=TENSOR, 0x20=PIPELINE
mode: 0x02                ← current cluster mode
task_id: 0x0007           ← unique ID for this task
chunk_id: 0x00             ← which chunk (0 = first)
storage_addr: 0x02700000  ← where input data lives in Alpha's storage
payload_size: 524288      ← 512KB of input data
pipeline_stage: 0x01      ← which assembly line stage (HEAVY mode only)
tensor_slice: 0x01        ← which matrix slice (SPEED mode only)
total_slices: 0x04        ← always 4 (one per worker)
subtask_total: 0x08       ← how many subtasks inside this chunk
```

The node replies with MSG_ACK, then starts working. It fetches the input data from Alpha using MSG_STORE_REQ (explained below), does the computation, and when finished sends MSG_CHUNK_DONE.

### MSG_CHUNK_DONE — "I finished my job"

- **Who sends it:** a worker node
- **Who receives it:** Alpha
- **When:** after a node completes its assigned task chunk

The worker is reporting back: job done. It includes a bitmask of which subtasks completed — think of this like a checklist. If all 8 subtasks finished, every bit is set. If the node overheated halfway through, only some bits are set and the rest need to be picked up by another node.

It also includes the current temperature so Alpha can decide whether to give this node more work or let it cool down first.

```
src_id: 0x02                  ← Charlie reporting in
dst_id: 0x00
msg_type: 0x11                ← this is CHUNK_DONE
task_id: 0x0007               ← same task that was assigned
chunk_id: 0x00                ← the chunk that just finished
subtask_bitmask: 0b11111111   ← all 8 subtasks done (each bit = one subtask)
temp_c: 52                    ← temperature after finishing
```

Alpha ACKs this, then either assigns another task or sends MSG_SLEEP if there is nothing left to do.

### MSG_MORE_WORK — "do you want another job?"

- **Who sends it:** Alpha
- **Who receives it:** a worker node
- **When:** Alpha is checking if the node is ready for more before assigning

Sometimes Alpha wants to verify a node is truly idle before pushing another task. This is a lightweight check rather than sending a full TASK_ASSIGN blindly.

```
src_id: 0x00
dst_id: 0x01
msg_type: 0x13    ← this is MORE_WORK
```

Node replies MSG_ACK if ready. Alpha then follows up with TASK_ASSIGN.

### MSG_SLEEP — "go to sleep"

- **Who sends it:** Alpha
- **Who receives it:** a worker node
- **When:** no more work to give, or node needs to cool down

Alpha tells a node to stop what it's doing and go into low-power sleep mode. The node stays powered and keeps its ethernet connection open so it can still receive a wake command, but it's not running any computation.

This happens in two situations:

1. All tasks are done and the node has nothing left to do
2. The node overheated and Alpha needs it to rest

```
src_id: 0x00
dst_id: 0x03      ← telling Delta to sleep
msg_type: 0x20    ← this is SLEEP
mode: 0x02        ← includes current mode for the node's log
```

### MSG_WAKE — "wake up"

- **Who sends it:** Alpha
- **Who receives it:** a sleeping worker node
- **When:** new work arrived, or a sleeping node has cooled down enough

Alpha wakes a sleeping node back up. After waking, the node re-initializes its network connection and sends a MSG_PONG to confirm it is alive — same as the startup handshake.

```
src_id: 0x00
dst_id: 0x03      ← waking Delta
msg_type: 0x21    ← this is WAKE
everything else: 0
```

After receiving the PONG, Alpha immediately assigns a new task.

### MSG_TEMP_REPORT — "here is my temperature"

- **Who sends it:** a sleeping worker node
- **Who receives it:** Alpha
- **When:** periodically while the node is sleeping after overheating

A node that was forced to sleep because of heat doesn't stay silent forever. It wakes up on its own every so often, checks its temperature, and reports to Alpha. Alpha then decides: cool enough to come back? or sleep more?

The cooldown threshold changes based on how many nodes are already working. If 3 other nodes are running and generating heat, Alpha makes the sleeping one wait until it cools more than it would if the cluster was quieter.

```
src_id: 0x01      ← Bravo reporting
dst_id: 0x00
msg_type: 0x30    ← this is TEMP_REPORT
temp_c: 61        ← current temperature
```

Alpha replies with MSG_ACK that includes the current active node count. If the temperature is below the threshold, Alpha immediately follows up with MSG_WAKE and a new task. If still too hot, Alpha just waits for the next report.

### MSG_CHUNK_HOT — "I'm overheating, stopping now"

- **Who sends it:** a worker node
- **Who receives it:** Alpha
- **When:** node hits 75°C while actively running a task

This is an emergency. The node got too hot mid-job and had to stop immediately to protect itself. It reports exactly how far it got using the subtask bitmask — which subtasks it finished and which ones it didn't. This way Alpha knows exactly what still needs to be done and can give the unfinished subtasks to another cooler node. Nothing is lost, just redistributed.

```
src_id: 0x04                  ← Echo overheated
dst_id: 0x00
msg_type: 0x12                ← this is CHUNK_HOT
task_id: 0x0007               ← the task it was working on
subtask_bitmask: 0b00111111   ← got through 6 of 8 subtasks before overheating
temp_c: 76                    ← temperature when it had to stop
```

Alpha ACKs, marks the remaining subtasks as unfinished, and the next available worker picks them up. The overheating node goes to sleep and starts reporting temperature periodically (MSG_TEMP_REPORT above).

### MSG_RTS — "can I talk?"

- **Who sends it:** a worker node
- **Who receives it:** Alpha
- **When:** before sending any non-urgent message

RTS stands for Request To Send. Before a node sends something like CHUNK_DONE or STORE_REQ, it has to ask permission first. This prevents two nodes from transmitting at the same time and causing confusion.

Alpha keeps a priority queue. The order goes Bravo first, then Charlie, Delta, Echo — but it rotates after each turn so no single node hogs the line.

```
src_id: 0x02      ← Charlie wants to talk
dst_id: 0x00
msg_type: 0x80    ← this is RTS
task_id: 0x0007   ← related to this task
```

### MSG_CTS — "yes, go ahead"

- **Who sends it:** Alpha
- **Who receives it:** one specific worker node
- **When:** Alpha grants a node permission to transmit

CTS stands for Clear To Send. Alpha looks at the queue, picks the highest priority node that raised RTS, and tells it to go. Everyone else keeps waiting.

```
src_id: 0x00
dst_id: 0x02      ← Charlie gets the green light
msg_type: 0x81    ← this is CTS
```

The node that got CTS now sends everything it needs to send. It waits for a MSG_ACK after each individual message before sending the next one.

### MSG_BUS_RELEASE — "I'm done talking"

- **Who sends it:** a worker node
- **Who receives it:** Alpha
- **When:** after a node finishes sending all its queued messages

Once the node has sent everything and received all its ACKs, it releases the bus. Alpha clears the node from the queue, rotates it to the back, and immediately checks if anyone else is waiting. If yes, they get CTS right away.

```
src_id: 0x02      ← Charlie is done
dst_id: 0x00
msg_type: 0x82    ← this is BUS_RELEASE
```

### MSG_CLOCK_SET — "everyone, run at this speed"

- **Who sends it:** Alpha
- **Who receives it:** ALL nodes at once (broadcast)
- **When:** automatically, whenever the number of active nodes changes

All nodes should run at the same clock speed. The speed depends on how many are active — more active nodes means more total heat, so Alpha adjusts everyone up or down together.

- 4 active nodes → 360MHz
- 3 or fewer → 240MHz

This is one of the few messages sent to everyone at once using the broadcast address (0xFF). Every node applies the new speed to itself and sends an individual ACK back.

```
src_id: 0x00
dst_id: 0xFF               ← broadcast to everyone
msg_type: 0x31             ← this is CLOCK_SET
subtask_bitmask: 360       ← new speed in MHz (this field is reused here as a generic number)
```

### MSG_STORE_REQ — "I need data from storage"

**NOT DONE — SD card not yet implemented**

- **Who sends it:** a worker node
- **Who receives it:** Alpha
- **When:** after receiving a task assignment, node needs its input data

A node that just got a task needs the input data to work on. It asks Alpha to send it. The message includes the virtual address (which slot in storage the data is at) and how many bytes it needs.

The virtual address scheme works like numbered shelves in a warehouse. Task 7 always lives at shelf 0x02700000. Alpha knows what is on each shelf.

```
src_id: 0x01
dst_id: 0x00
msg_type: 0x70             ← this is STORE_REQ
storage_addr: 0x02700000   ← "i need what's at this address"
payload_size: 524288       ← "i need 512KB of it"
```

### MSG_STORE_LOCK — "everyone stop talking, I'm doing a transfer"

**NOT DONE**

- **Who sends it:** Alpha
- **Who receives it:** ALL nodes (broadcast)
- **When:** right before sending a large chunk of data to one node

Sending 512KB of data takes time and needs the full network link without interruption. Before starting, Alpha broadcasts a lock to everyone. All nodes immediately freeze their outbound queues — no more RTS, no more messages. They just wait.

```
src_id: 0x00
dst_id: 0xFF      ← broadcast to all
msg_type: 0x71    ← this is STORE_LOCK
```

### MSG_STORE_DATA — "here is the data you asked for"

**NOT DONE**

- **Who sends it:** Alpha
- **Who receives it:** the node that requested data
- **When:** after STORE_LOCK is sent

Alpha sends this as a header first over PORT_CTRL (5000) to tell the node how much data is coming. Immediately after, the actual raw binary data flows over PORT_DATA (5001). The node reads exactly payload_size bytes from that port to get its input data.

```
src_id: 0x00
dst_id: 0x01               ← only to the node that asked
msg_type: 0x72             ← this is STORE_DATA
storage_addr: 0x02700000   ← echoed so node can verify it got the right thing
payload_size: 524288       ← how many bytes are coming on PORT_DATA
```

### MSG_STORE_DONE — "I got all the data"

**NOT DONE**

- **Who sends it:** a worker node
- **Who receives it:** Alpha
- **When:** after the node finishes receiving all the data on PORT_DATA

The node confirms it has received the complete transfer. Alpha waits for this before unlocking the bus. If it never arrives, Alpha knows something went wrong with the transfer.

```
src_id: 0x01
dst_id: 0x00
msg_type: 0x73    ← this is STORE_DONE
```

### MSG_STORE_UNLOCK — "transfer done, everyone can talk again"

**NOT DONE**

- **Who sends it:** Alpha
- **Who receives it:** ALL nodes (broadcast)
- **When:** after STORE_DONE is received from the requesting node

Alpha broadcasts the all-clear. Every node that had queued up an RTS while waiting can now transmit. The priority queue picks back up where it left off.

```
src_id: 0x00
dst_id: 0xFF      ← broadcast to all
msg_type: 0x74    ← this is STORE_UNLOCK
```

### MSG_STORE_WRITE — "I'm sending my results back to you"

**NOT DONE**

- **Who sends it:** a worker node
- **Who receives it:** Alpha
- **When:** after a node finishes computing and needs to save its output

The reverse of STORE_REQ. The node finished its computation and now needs to send the results back to Alpha for storage. Alpha locks the bus the same way, receives the data on PORT_DATA, and confirms when done.

```
src_id: 0x01
dst_id: 0x00
msg_type: 0x75             ← this is STORE_WRITE
storage_addr: 0x03700000   ← where to put the results
payload_size: 524288       ← how many bytes are coming
```

### MSG_STORE_WDONE — "I received your results"

**NOT DONE**

- **Who sends it:** Alpha
- **Who receives it:** the node that sent results
- **When:** after Alpha finishes receiving the write-back data

Alpha confirms it received everything. It then broadcasts STORE_UNLOCK and the cluster resumes normal operation.

```
src_id: 0x00
dst_id: 0x01
msg_type: 0x76    ← this is STORE_WDONE
```

## Full startup sequence, start to finish

```
Alpha boots up
│
├── PING → Bravo "are you there?"
│   └── PONG ← Bravo "yes, temp is 28°C"
│
├── PING → Charlie
│   └── PONG ← Charlie
│
├── PING → Delta
│   └── PONG ← Delta
│
├── PING → Echo
│   └── PONG ← Echo
│
├── CLOCK_SET → everyone "run at 360MHz, all 4 of you are online"
│
├── MODE_CHANGE → Bravo "we're running in SPEED mode"
│   └── ACK ← Bravo
├── MODE_CHANGE → Charlie
│   └── ACK ← Charlie
├── MODE_CHANGE → Delta
│   └── ACK ← Delta
├── MODE_CHANGE → Echo
│   └── ACK ← Echo
│
├── TASK_ASSIGN → Bravo "here's your job, chunk 0, slice 0"
│   └── ACK ← Bravo
├── TASK_ASSIGN → Charlie "here's your job, chunk 0, slice 1"
│   └── ACK ← Charlie
├── TASK_ASSIGN → Delta "here's your job, chunk 0, slice 2"
│   └── ACK ← Delta
├── TASK_ASSIGN → Echo "here's your job, chunk 0, slice 3"
│   └── ACK ← Echo
│
│   [all nodes working...]
│
├── RTS ← Bravo "I finished, can I report?"
│   └── CTS → Bravo "go ahead"
│       └── CHUNK_DONE ← Bravo "done, all subtasks complete, 51°C"
│           └── ACK → Bravo
│               └── BUS_RELEASE ← Bravo
│                   └── TASK_ASSIGN → Bravo "here's your next job"
│
│   [Charlie, Delta, Echo finish similarly]
│
└── [all tasks done]
    ├── SLEEP → Bravo
    ├── SLEEP → Charlie
    ├── SLEEP → Delta
    └── SLEEP → Echo
```

## One important thing

Nodes never talk directly to each other. If Bravo needs something that Charlie computed, it has to go through Alpha. Alpha is always in the middle. This keeps things simple — only one node (Alpha) ever needs to know the full picture of what's happening in the cluster.
