# Hardware

## Nodes

### Alpha — ESP32 DOIT DevKit V1
- dual core Xtensa LX6 @ up to 240MHz
- 520KB SRAM, no PSRAM
- 4MB flash
- built-in USB (micro USB) for serial monitoring
- role: admin, bus controller, task distributor
- connects to PC via USB serial (115200 baud) for monitoring and commands

### Workers (Bravo, Charlie, Delta, Echo) — ESP32-S3-N16R8
- dual core Xtensa LX7 @ up to 240MHz (can go 360MHz when all 4 active)
- 512KB internal SRAM + 8MB PSRAM (octal SPI)
- 16MB flash
- role: compute workers
- each handles one task chunk at a time

---

## Networking

- 1x LAN8720 PHY module per node (5 total)
- connected to ESP32 via RMII interface (~10 wires per node)
- 5-port 10/100 unmanaged ethernet switch
- Cat5 patch cables
- speed: 100Mbps full duplex per port

the switch handles all routing automatically, no config needed. each node gets its own dedicated 100Mbps link.

---

## Power

secondhand PSU from bitcoin mining equipment.

| Rail | Voltage | Current | Used for |
|------|---------|---------|----------|
| 3.3V | 3.3V | 7.2A | Worker nodes + LAN8720s |
| 5V | 5V | 4.66A | Alpha DevKit (VIN) |
| 12V | 12V | 0.42A | not used |

**estimated draw:**

| Device | Current |
|--------|---------|
| ESP32-S3 (active) | ~300mA |
| LAN8720 | ~150mA |
| Per worker node total | ~450mA |
| 4x worker nodes | ~1.8A |
| Alpha DevKit (5V) | ~500mA |

3.3V rail: 1.8A used out of 7.2A available — plenty of headroom.

**note:** voltages were verified with a multimeter before connecting anything. rails are marked on the PSU wiring.

---

## Clock scaling

clock frequency scales based on active node count and temperature:

| Active nodes | Clock |
|-------------|-------|
| 4 | 360MHz |
| 3 / 2 / 1 | 240MHz (floor) |

thermal throttle kicks in at 75°C. node reports CHUNK_HOT, Alpha forces it to sleep. wake treshold depends on how many other nodes are still active.
