# Network

## Physical setup

```
[Alpha]---[Switch]---[Bravo]
              |------[Charlie]
              |------[Delta]
              |------[Echo]
```

5-port 10/100 unmanaged switch. all 5 ports used by nodes. PC monitoring is done through Alpha's USB serial, not through the switch.

---

## Switch scenario (recommended)

the protocol is almost entirely unicast UDP. every `TASK_ASSIGN`, `CTS`, `ACK`, `STORE_DATA` etc goes to a specific node IP on a specific port.

on a switch, unicast frames go only to the correct port. the dst_id check in firmware (`if msg.dst_id != NODE_ID && msg.dst_id != 0xFF: discard`) still exists as a software safety layer but it almost never fires — the switch already made sure the wrong nodes never got the packet in the first place.

broadcast messages (STORE_LOCK, STORE_UNLOCK, CLOCK_SET → 192.168.1.255) flood to all ports on a switch automatically, which is correct behavior.

result: each node gets a clean dedicated 100Mbps link. no wasted CPU cycles parsing packets meant for someone else. RTS/CTS arbitration handles scheduling, switch handles routing.

---

## Hub scenario (works, but worse)

hubs are basically dead hardware at this point, but if someone ends up using one:

on a hub every port shares the same 100Mbps. every frame from any node gets broadcast to all ports simultaneously — there is no routing. every node's NIC receives every packet on the wire.

this is where the firmware's dst_id filter becomes load-bearing:

```
node receives packet
→ checks dst_id
→ if dst_id != my ID and dst_id != 0xFF → discard
→ otherwise process
```

on a hub this check fires constantly. Alpha sending a task to Bravo means Charlie, Delta and Echo all wake up, parse the 28-byte header, verify the checksum, check the dst_id, then throw it away. multiply that by every ACK, every CTS, every STORE_DATA chunk.

the RTS/CTS bus arbitration also becomes the actual collision prevention layer rather than just a scheduling tool, since on a shared medium two nodes transmitting simultaneously causes a real ethernet collision. on a switch collisions dont happen (full duplex per port).

it would still work. the protocol was designed with enough safety layers that a hub wont break it. its just slower and noisier than it needs to be.

**tl;dr:** switch = better. hub = functional but wasteful.

---

## How the switch learns where each node is

the switch is fully automatic, no config needed. it learns by watching source MAC adresses on each port.

1. when a node sends any ethernet frame, the switch notes `source MAC → this port` in its internal table
2. when a frame arrives destined for a known MAC, it goes out only that port
3. unknown MACs get flooded (like a hub) until learned
4. broadcast MAC (FF:FF:FF:FF:FF:FF) always floods

after boot and initial handshake ping/pong exchange, the switch has learned all 5 nodes within seconds.

---

## MAC and IP

MAC address: 48-bit hardware ID burned into each LAN8720/ESP32 at manufacture. format like `3C:71:BF:9A:1C:02`. first 3 bytes = manufacturer, last 3 = unique ID.

IP address: what the firmware uses (192.168.1.10 - .14). software layer, configured in code.

when Alpha calls `sendto("192.168.1.11", ...)` for the first time:
1. ESP32 IP stack doesn't know Bravo's MAC yet
2. broadcasts ARP: "who has 192.168.1.11?"
3. Bravo replies with its MAC
4. Alpha caches the mapping in its ARP table
5. from then on it builds ethernet frames with Bravo's MAC directly
6. switch routes those frames to Bravo's port only

---

## LAN8720 connection (RMII)

each node has its own LAN8720 PHY module connected via RMII to the ESP32. roughly 10 wires per node:

```
MDC, MDIO, TXD0, TXD1, TX_EN, RXD0, RXD1, CRS_DV, REF_CLK, 3.3V, GND
```

specific GPIO pinout will be documented after hardware is assembled and tested.
