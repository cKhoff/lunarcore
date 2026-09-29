# LunarCore

**Multi-protocol mesh firmware for ESP32-S3 LoRa devices.**

LunarCore turns a single Heltec WiFi LoRa 32 V3 (or any ESP32-S3 + SX1262 board) into a universal mesh node that speaks **MeshCore**, **Meshtastic**, and **RNode/KISS (Reticulum)** — auto-detected from the first bytes over serial or BLE. No configuration needed: plug in an app, and the firmware figures out which protocol you're using.

---

## Features

### Multi-Protocol Radio Bridge

The device acts as a transparent bridge between your app (over USB serial or BLE) and the LoRa radio:

| Protocol | Serial Sync | LoRa Config | Use Case |
|----------|------------|-------------|----------|
| **MeshCore** | `0xAA 0xC3` | 910.525 MHz, SF7, 62.5 kHz, private sync word | Custom mesh networks with onion routing |
| **Meshtastic** | `0x94 0xC3` | 906.875 MHz, SF11, 500 kHz, LDRO | Standard Meshtastic mesh (phone apps, web UI) |
| **RNode/KISS** | KISS FEND framing | Configurable (EU868/US915 presets) | Reticulum networking over LoRa |

Protocol is **auto-detected** from the first bytes received on serial or BLE. The radio reconfigures itself (frequency, spreading factor, bandwidth, sync word) when the protocol changes. You can also force a reset back to detection mode with `AT+SWITCH`.

### Standalone Repeater Mode

When no app is connected (no serial activity for 30 s, no BLE link), the device automatically becomes a **LoRa repeater**:

- Relays MeshCore packets it hears on the air
- 32-entry deduplication ring (CRC16-based fingerprint) prevents loops
- Random jitter (20–100 ms, derived from node ID + timestamp) staggers retransmissions
- Enabled by default; toggle with `AT+REPEATER=0/1` (persisted in NVS)
- LED flashes 3× for relayed packets, 2× for direct receives

This lets you deploy nodes in hard-to-reach spots (attics, rooftops, trees) that extend your mesh without needing a phone or laptop nearby.

### Bluetooth Low Energy

Full BLE GATT server with two services:

- **Nus** (Nordic UART Service) — generic serial-over-BLE for any app
- **Meshtastic** — native `FromRadio`/`ToRadio` notification channels so Meshtastic phone apps connect directly without a USB cable

The device advertises as **"LunarCore"** and supports multiple simultaneous connections. BLE disconnect triggers protocol hot-switching back to detection mode after 5 s of no serial activity.

### Wired Crypto Stack (No Heap, No Std)

All cryptographic primitives are implemented from scratch in Rust (`no_std`, `heapless` buffers only):

| Primitive | File | Notes |
|-----------|------|-------|
| SHA-256 | `crypto/sha256.rs` | Constant-time |
| AES-128/256 (CTR/GCM) | `crypto/aes.rs` | Constant-time S-box via GF(2⁸) inversion |
| ChaCha20 / XChaCha20 | `crypto/chacha20.rs` | IETF + RFC 8439 variants |
| Poly1305 MAC | `crypto/poly1305.rs` | Universal hash |
| ChaCha20-Poly1305 AEAD | `crypto/poly1305.rs` | Combined seal/open |
| HMAC-SHA256 | `crypto/hmac.rs` | |
| HKDF-SHA256 | `crypto/hkdf.rs` | Key derivation |
| Ed25519 signatures | `crypto/ed25519.rs` | Sign/verify, constant-time |
| X25519 key exchange | `crypto/x25519.rs` | ECDH for session keys |

Supporting utilities: constant-time comparison, secure zeroization, hardware RNG with health check (`esp_fill_random`).

### Session & Onion Routing

- **Session encryption:** X25519 ECDH → HKDF → ChaCha20-Poly1305 per-message AEAD. Sessions are keyed by peer public key, persisted to NVS across reboots.
- **Onion routing:** Up to 7-layer encrypted hop-by-hop forwarding through intermediate mesh nodes. Each layer is X25519-encrypted to the next hop's key; intermediates decrypt one layer, learn the next hint, and forward. Final recipient decrypts the innermost layer.
- **Address translation:** A single 32-byte Ed25519 public key derives all protocol addresses (MeshCore 16-bit, Meshtastic 32-bit, Reticulum 128-bit hash).

### Node Identity

- Generated once at first boot: random 32-byte seed → Ed25519 keypair + X25519 keypair + 32-bit node ID
- Stored in NVS (`lunarcore` namespace) — survives power cycles
- Privacy-first: identity is random, not derived from MAC address
- Hardware serial (efuse) mixed into the seed for uniqueness
- Factory reset supported (erases NVS, generates new identity)

### OTA Firmware Update

- Signed update pipeline: Ed25519 signature verification against trusted keys
- Dual-bank flash layout (partition table: `ota_0` / `ota_1`)
- Version gating (min major/minor/patch)
- Progress reporting, rollback on bad signature
- Managed via `OtaManager` — begin → write chunks → verify → commit

### Contact Exchange

- **ContactHello** message format: signed (Ed25519) introduction packet containing public key, node ID, and optional petname
- QR-code encodable for face-to-face pairing
- Trust levels: Unknown → Verified → Trusted
- Fingerprint (8-byte hash) for human-readable verification

### Power Management

- Light sleep with GPIO wake sources
- CPU frequency scaling (80 / 160 / 240 MHz)
- Battery voltage monitoring via ADC (GPIO1, 4.9× divider)
- Non-linear discharge curve (LiPo 3.0–4.2 V → 0–100 %)
- Low (< 3.4 V) and critical (< 3.2 V) thresholds with LED error blink
- Charging detection (> 4.2 V)

### OLED Status Display

SSD1306 128×64 over I²C (GPIO17/18, RST on GPIO21):

- Current protocol name
- Node ID (hex)
- Battery %
- RX/TX packet counts
- Last RSSI
- IRQ status, DIO1 count
- Chip mode, device errors
- Repeater active indicator + relay count
- Boot animation

Refreshes every 2 seconds.

### Watchdog & Fault Tolerance

- Task watchdog: 30 s timeout, panics on expiry
- Fed every main-loop iteration
- All external I/O (SPI, I²C, UART, ADC) has error handling
- Radio errors counted and displayed

---

## AT Command Reference

Connect at 115200 baud (or via BLE Nus service). Commands are case-insensitive, terminated by `\r` or `\n`.

| Command | Response | Description |
|---------|----------|-------------|
| `AT` | `OK` | Test / alive |
| `ATI` / `AT+VERSION` | Version info | Firmware version, protocol list |
| `AT+STATUS` | Status block | RX state, battery, TX/RX counts |
| `AT+NODEID` | `Node ID: XXXXXXXX` | 32-bit hex node ID |
| `AT+MAC` | `MAC: AA:BB:CC:DD:EE:FF` | Default MAC from efuse |
| `AT+FREQ=<Hz>` | `OK` / `ERROR` | Set LoRa center frequency |
| `AT+SF=<7-12>` | `OK` / `ERROR` | Set spreading factor |
| `AT+TXPOWER=<-9..22>` | `OK` / `ERROR` | Set TX power (dBm) |
| `AT+RX` | `OK` / `ERROR` | Start continuous RX |
| `AT+RSSI` | `RSSI: -XX dBm` | Current RSSI reading |
| `AT+RESET` | `OK` / `ERROR` | Reinitialize radio |
| `AT+SWITCH` | `Protocol reset: Detecting...` | Reset to protocol auto-detection |
| `AT+REPEATER` | `Repeater: ON/OFF, Relayed: N` | Query repeater status |
| `AT+REPEATER=0` | `Repeater: OFF` | Disable repeater mode |
| `AT+REPEATER=1` | `Repeater: ON` | Enable repeater mode |
| `AT+HELP` / `AT?` | Command list | Show available commands |

---

## Hardware

**Primary target:** [Heltec WiFi LoRa 32 V3](https://heltec.org/project/wifi-lora-32/)

| Component | Spec |
|-----------|------|
| MCU | ESP32-S3 (Xtensa dual-core, 240 MHz) |
| Radio | Semtech SX1262 (LoRa, 150–960 MHz) |
| Flash | 8 MB QSPI |
| PSRAM | 8 MB Octal |
| Display | SSD1306 128×64 OLED (I²C) |
| Battery | LiPo via onboard charger, ADC on GPIO1 |

**Pin map:**

| Function | GPIO |
|----------|------|
| SPI MOSI | 10 |
| SPI MISO | 11 |
| SPI SCK | 9 |
| LoRa NSS | 8 |
| LoRa RESET | 12 |
| LoRa BUSY | 13 |
| LoRa DIO1 (IRQ) | 14 |
| LED | 35 |
| VEXT enable | 36 |
| Battery ADC | 1 |
| I²C SDA | 17 |
| I²C SCL | 18 |
| OLED RST | 21 |
| UART TX | 43 |
| UART RX | 44 |

Also compatible with LilyGo T3-S3 and other ESP32-S3 + SX1262 boards (pin remap may be required).

---

## Flash (Prebuilt Binary)

Download `lunarcore-esp32s3.bin` from [Releases](../../releases). The binary includes bootloader + partition table + application merged at offset `0x0`.

```bash
pip install esptool
esptool.py --chip esp32s3 -p PORT write_flash 0x0 lunarcore-esp32s3.bin
```

Replace `PORT`:
- **Linux:** `/dev/ttyACM0` or `/dev/ttyUSB0`
- **macOS:** `/dev/cu.usbmodem*`
- **Windows:** `COM3` (check Device Manager)

## Build from Source

Prerequisites: [espup](https://github.com/ivmarkov/espup), Rust stable.

```bash
espup install
. ~/export-esp.sh
cargo build --target xtensa-esp32s3-espidf --release
espflash flash target/xtensa-esp32s3-espidf/release/lunarcore --monitor
```

CI builds a merged `.bin` on tag push (`v*`) and attaches it to the GitHub Release automatically.

---

## Architecture

```
src/
├── main.rs              # Main loop, AT commands, protocol dispatch, repeater logic
├── main_minimal.rs      # Minimal test entry point (crypto + protocol + sx1262 smoke test)
├── protocol.rs          # MeshCore frame parser/builder (sync 0xAA 0xC3, CRC16)
├── protocol_router.rs   # Protocol auto-detection state machine
├── meshtastic/
│   ├── mod.rs           # Meshtastic handler: ToRadio/FromRadio, serial framing
│   ├── channel.rs       # Channel config, modem presets, LoRa params
│   ├── encryption.rs    # AES-128-CTR channel encryption + MIC
│   ├── packet.rs        # MeshPacket parse/build, routing decisions, dedup cache
│   └── protobuf.rs      # Minimal protobuf encode/decode for Meshtastic messages
├── rnode.rs             # KISS parser, RNode commands, EU868/US915 presets
├── ble.rs               # BLE GATT server (Nus + Meshtastic services)
├── display.rs           # SSD1306 OLED driver + status rendering
├── transport.rs         # WirePacket format, address translation, universal addressing
├── session.rs           # Session crypto (X25519→HKDF→ChaCha20-Poly1305), NVS persistence
├── onion.rs             # Onion router: wrap/unwrap up to 7 layers, route builder
├── sx1262.rs            # SX1262 SPI driver: init, configure, TX/RX, IRQ handling
├── ota.rs               # OTA manager: signed updates, dual-bank, version gating
├── contact.rs           # ContactHello: signed intro messages, QR encoding, trust levels
├── identity.rs          # Node identity: keygen, signing, encrypted storage
├── wifi.rs              # WiFi + TCP client (for future internet gateway mode)
├── power.rs             # Power management: light sleep, GPIO wake, CPU freq
├── packet_id.rs         # Packet ID generation with session rotation
├── rng.rs               # Hardware RNG wrapper with health check
└── crypto/
    ├── mod.rs           # Utilities: constant-time compare, secure zero, random
    ├── sha256.rs        # SHA-256
    ├── aes.rs           # AES-128/256 (constant-time)
    ├── chacha20.rs      # ChaCha20, XChaCha20, HChaCha20
    ├── poly1305.rs      # Poly1305, ChaCha20-Poly1305 AEAD
    ├── hmac.rs          # HMAC-SHA256
    ├── hkdf.rs          # HKDF-SHA256
    ├── ed25519.rs       # Ed25519 sign/verify
    └── x25519.rs        # X25519 ECDH
```

### Data Flow

```
App (serial/BLE)
    │
    ▼
ProtocolDetector ──► MeshCore / Meshtastic / RNode parser
    │                        │
    │                   handle_*_frame()
    │                        │
    │                   radio.transmit() ◄──► SX1262 (SPI)
    │                        │
    ▼                   LoRa air
Repeater (dedup + jitter) ──► radio.transmit()
    ▲
    │
radio.read_packet() ──► route_rx_packet() ──► App (serial/BLE)
                         │
                         └──► maybe_relay_packet() (if repeater active)
```

### Protocol Hot-Switching

The device continuously monitors for protocol changes:

1. **Serial idle > 30 s** (and no BLE) → reset to detection mode
2. **BLE disconnect** (and serial idle > 5 s) → reset to detection mode
3. **Manual** → `AT+SWITCH`

On reset, all parsers are cleared, the dedup ring is flushed, and the detector starts fresh from the next byte.

---

## Threat Model & Security Notes

| Aspect | Implementation |
|--------|---------------|
| Transport encryption | ChaCha20-Poly1305 AEAD per message (session-keyed) |
| Key exchange | X25519 ECDH |
| Authentication | Ed25519 signatures (identity, ContactHello, OTA) |
| Replay protection | Per-session nonce counter + packet ID cache |
| Loop prevention | 32-entry CRC16 dedup ring + random jitter |
| Key storage | NVS flash (plaintext — physical access threat) |
| RNG | `esp_fill_random` with health check; panics on weak entropy |
| Constant-time crypto | S-box, comparison, MAC verification all CT |

**Known limitations:**
- Private key in NVS is unencrypted (acceptable for embedded; JTAG/SPI bus access exposes it)
- AT command interface has no authentication (any serial/BLE peer can issue commands)
- Dedup ring is small (32 entries) — busy meshes may evict before jitter window closes
- `millis()` assumes 1 kHz FreeRTOS tick rate

---

## License

MIT
