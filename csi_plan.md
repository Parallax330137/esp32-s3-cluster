# WiFi CSI Sensing — Future Idea

**status: not started. just an idea for now.**

---

## What is this

WiFi Channel State Information (CSI). when a WiFi signal travels through a space it bounces off walls, furniture, and people. anything that disturbs the signal path changes the amplitude and phase of individual subcarriers. you can extract this data from the WiFi radio and use it to detect movement, presence, or even breathing in a room.

basically passive radar using the WiFi radio that's already in the ESP32.

---

## Why this cluster is interesting for it

each worker node has an ESP32-S3 with a WiFi radio. all cluster communication already runs over ethernet via LAN8720, so the WiFi radio on each node is completely free — not doing anything.

4 nodes = 4 antennas at different physical positions = spatial data from multiple points, not just one.

Alpha can collect all CSI readings from workers over the existing ethernet protocol and distribute the processing (FFT, stats) back out to the same workers using the task system thats already built.

---

## How it would work roughly

```
each node runs esp_wifi_set_csi() to enable CSI extraction
nodes sniff ambient WiFi traffic passively (doesnt need to connect to anything)
for each received WiFi frame, the radio returns per-subcarrier amplitude + phase data
node packages this as a data chunk, sends to Alpha over ethernet
Alpha distributes processing using TASK_MATH_STATS or a new TASK_CSI type
processed output = disturbance map of the room
```

---

## What ESP32 can realistically detect

- person walking through a room: yes, fairly reliably
- person standing still / presence detection: possible but harder
- breathing detection: needs good placement and filtering, some papers have done it
- through-wall: weak at 2.4GHz power levels, same-room is much more practical

---

## Hardware consideration

the ESP32-S3 devkits have a PCB trace antenna. for CSI sensing this is probably fine for a first test, same-room detection doesnt need a lot of range. external antenna would help but requires a board with an IPEX/U.FL connector which these dont have.

---

## References

- Espressif CSI tool: https://github.com/espressif/esp-csi
- there are quite a few papers on ESP32-specific CSI sensing if you search "ESP32 CSI human detection"

---

## When to think about this

after the cluster is physically working and OTA is done. this is a future expansion, not near-term.
