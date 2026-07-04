# OTA Firmware Update — Plan

**status: not implemented yet. hardware not tested yet either.**

the goal is to never need to physically touch a node after the first flash. new firmware gets sent over ethernet from Alpha, node writes it to flash, reboots into new firmware.

---

## Why this matters

nodes are going to be wired up and potentially inside an enclosure eventually. reflashing via USB means desoldering or at least unplugging things. OTA avoids all of that.

also just a good practice — anything deployed should be updatable remotely.

---

## How it works (ESP-IDF)

ESP-IDF has built-in OTA support via `esp_ota_ops.h`. the flash is divided into two app partitions (ota_0 and ota_1). the bootloader tracks which one is active.

```
normal boot:  running ota_0
OTA update:   new firmware written to ota_1
              verified (checksum)
              bootloader told to switch
              reboot → now running ota_1
next OTA:     written to ota_0, toggle again
```

if the new firmware crashes on boot, ESP-IDF rolls back to the previous partition automatically. so a bad flash doesnt brick the node.

---

## Planned flow

```
1. PC compiles new firmware → .bin file
2. PC sends .bin to Alpha over USB serial (or Alpha fetches over WiFi)
3. Alpha sends MSG_OTA_BEGIN to target node
4. Alpha streams .bin chunks to node over existing Ethernet/UDP
5. Node writes each chunk using esp_ota_write()
6. Node calls esp_ota_end() → esp_ota_set_boot_partition() → reboot
7. Node boots new firmware, sends PONG to Alpha confirming its alive
8. Alpha marks node as updated
```

Alpha's role here is just to trigger and relay — it has no PSRAM to buffer the whole binary so it streams chunk by chunk as they come in from the PC side.

---

## New message types needed

```
MSG_OTA_BEGIN    Alpha tells node OTA is starting, includes total size
MSG_OTA_CHUNK    Alpha sends firmware chunk (with chunk index + checksum)
MSG_OTA_END      Alpha signals transfer complete
MSG_OTA_VERIFY   Node confirms write was successful (or reports error)
```

these would get added to the existing MsgType_t enum.

---

## Partition table

each node needs this partition table (one time via USB):

```
nvs        0x9000    24KB
otadata    0xE000    8KB
ota_0      0x10000   1.5MB
ota_1      0x190000  1.5MB
```

total: fits within 4MB flash on workers. Alpha only needs the same since OTA for Alpha itself would still go through USB (Alpha is the flasher, not a target).

---

## Notes / concerns

- UDP packet loss during OTA transfer is a real risk. need per-chunk ACK and retry, same as existing protocol already does for data transfers
- Alpha DevKit has no PSRAM so it cant buffer the whole .bin — must stream
- rollback is the safety net. if something goes wrong the node just boots old firmware
- this will probably take a few tries to get right. thats fine

---

## Alternative for now (before OTA is implemented)

PC runs a Python HTTP server, node pulls the .bin over WiFi:

```bash
python -m http.server 8080
```

Alpha just sends a trigger message to the target node. node fetches the file itself via `esp_http_client`. no Alpha buffering needed this way.

this is the simpler first step. full Alpha-as-flasher comes later.
