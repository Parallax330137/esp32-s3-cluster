# ESP32 Distributed Compute Cluster

A custom 5-node distributed computing cluster built on ESP32 hardware, communicating over ethernet using a hand-written UDP protocol with RTS/CTS bus arbitration.

this is a personal project, still a work in progress.

---

## What is this

5 ESP32 nodes connected via Cat5 ethernet through a 5-port switch. one node acts as admin (Alpha), the other four are workers. Alpha assigns compute tasks, manages the bus, monitors temps, and talks to the PC over USB serial.

the nodes dont run linux or any real OS. just FreeRTOS + ESP-IDF with a custome protocol on top.

---

## Hardware

| Node | Board | Role | IP |
|------|-------|------|----|
| Alpha | ESP32 DOIT DevKit V1 | Admin + bus controller | 192.168.1.10 |
| Bravo | ESP32-S3-N16R8 | Worker 1 | 192.168.1.11 |
| Charlie | ESP32-S3-N16R8 | Worker 2 | 192.168.1.12 |
| Delta | ESP32-S3-N16R8 | Worker 3 | 192.168.1.13 |
| Echo | ESP32-S3-N16R8 | Worker 4 | 192.168.1.14 |

each worker has 16MB flash and 8MB PSRAM. Alpha is a standart DevKit (no PSRAM).

**Other hardware:**
- 5x LAN8720 PHY module (one per node, RMII interface)
- 5-port 10/100 unmanaged ethernet switch
- Cat5 patch cables
- secondhand mining PSU (3.3V @ 7.2A, 5V @ 4.66A)

---

## Current status

- [x] Protocol designed and implemented (v3.1)
- [x] RTS/CTS bus arbitration working
- [x] Task distribution (POLY, TENSOR, PIPELINE, STATS)
- [x] Thermal throttling (hot-cap)
- [x] Serial monitor commands on Alpha
- [x] Wokwi simulation working
- [ ] Real hardware wiring (parts arriving)
- [ ] OTA firmware update (planned, see docs/ota_plan.md)
- [ ] WiFi CSI sensing (future idea, see docs/csi_plan.md)

---

## Repo structure

```
/firmware
  alpha_admin1.ino     admin node
  bravo_node2.ino      worker node (NODE_ID 1)
  charlie_node3.ino    worker node (NODE_ID 2)
  delta_node4.ino      worker node (NODE_ID 3)
  echo_node5.ino       worker node (NODE_ID 4)

/docs
  hardware.md          full hardware specs and power budget
  protocol.md          UDP protocol, message types, bus arbitration
  network.md           ethernet setup, switch vs hub, MAC/ARP notes
  ota_plan.md          planned OTA firmware update system
  csi_plan.md          future WiFi CSI sensing idea
```

---

## How to flash (for now)

1. open the worker firmware
2. change `NODE_ID` at the top to match the node (1=Bravo, 2=Charlie, etc.)
3. flash via USB
4. repeat for each node

once OTA is implemented, only the initial flash needs USB.

---

## Serial commands (Alpha)

connect to Alpha over USB serial at 115200 baud.

```
HANDSHAKE          re-run node discovery
RUN HEAVY          full pipeline + tensor on all nodes
RUN SPEED          tensor only, all nodes
RUN LIGHT <n>      pipeline, n nodes only
STATUS             cluster overview
NODES              per-node detail
TASKS              task manifest
RESET              reset cluster state
```

see docs/protocol.md for the full list including test and sim commands.
