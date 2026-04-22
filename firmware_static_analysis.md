# ESP32-S3 Firmware Static Analysis (`ota_0.bin`)

## Scope and input completeness
- Available artifact in this workspace: `ota_0.bin` only (6,291,456 bytes / 6 MiB). 
- Not available in workspace: `bootloader.bin`, `partitions.bin`, `ota_1.bin`, `assets.bin`, `tikpal.bin`, full flash dump, eFuse dump, UART boot log.
- Therefore: partition-map reconstruction and boot security state can only be **partially inferred** from app image internals.

## 1. Executive technical summary
- Target image is an **ESP32-S3 ESP-IDF app** with project name `xiaozhi`, version `2.0.4.20260417`, built on **Apr 17 2026 16:18:52**, using **ESP-IDF v5.5.1**.
- Board/product family is strongly tied to **TikPal** (`TikPalBoard`, `./main/boards/tikpal/...`, `tikpal_v2.0.4...changelog`).
- Display stack appears to be **SPI LCD via esp_lcd + CO5300 panel driver** and **LVGL 9** integration.
- Touch stack appears to be **CST820 over I2C** through `esp_lcd_touch`.
- Audio stack is custom C++ abstraction (`BoxAudioCodec`) with **full-duplex I2S channels** (TX std mode + RX TDM mode), `esp_codec_dev` HAL, Opus codec, and wake-word hooks.
- Connectivity includes Wi-Fi STA/AP, captive provisioning web UI on `192.168.4.1`, BluFi BLE provisioning, websocket transport, OTA over HTTPS, optional MQTT section in server JSON.
- Storage stack includes NVS + LittleFS + SD card + USB MSC mode. There is explicit logic for assets update and an `assets_A` marker, suggesting separate assets partition.
- Power stack includes PMIC abstraction, battery/charging logic, low-battery UI, deep-sleep wake by boot button/touch/timer, and aggressive power-save transitions.
- Binary is **semi-symbolic** (many source paths, class/method names leaked), enabling high-confidence reverse-oriented BSP reconstruction.

## 2. Confirmed findings
- SoC/image type: `Detected image type: ESP32-S3`.
- App metadata:
  - `Project name: xiaozhi`
  - `App version: 2.0.4.20260417`
  - `Compile time: Apr 17 2026 16:18:52`
  - `ESP-IDF: v5.5.1`
  - `Secure version: 0`
- Build profile hints: source paths from both app and IDF are embedded; C++ symbol names present; assertions and detailed logs present.
- Board namespace:
  - `void TikPalBoard::initCO5300Display()`
  - `void TikPalBoard::initTouch()`
  - `void TikPalBoard::initCodecI2c()`
  - `void TikPalBoard::initSpi()`
- Display:
  - `esp_lcd_new_panel_io_spi(SPI2_HOST, &io_config, &panel_io)`
  - `esp_lcd_new_panel_co5300(panel_io, &panel_config, &panel_)`
  - LVGL port component paths include `esp_lvgl_port/src/lvgl9/...`
- Touch:
  - `esp_lcd_new_panel_io_i2c(i2c_bus_, &tp_io_config, &tp_io_handle)`
  - `esp_lcd_touch_new_i2c_cst820(tp_io_handle, &tp_cfg, &tp)`
- Audio:
  - `BoxAudioCodec::CreateDuplexChannels(gpio_num_t, gpio_num_t, gpio_num_t, gpio_num_t, gpio_num_t)`
  - `i2s_new_channel(...tx_handle_, ...rx_handle_)`
  - `i2s_channel_init_std_mode(tx_handle_, &std_cfg)`
  - `i2s_channel_init_tdm_mode(rx_handle_, &tdm_cfg)`
  - `esp_codec_dev_open/read/write/set_out_vol/set_in_channel_gain`
  - Opus encode/decode error/log paths and wake-word init failure logs.
- Network/OTA:
  - OTA base URL string: `https://groot.tikpal.io/api/ota/`
  - Websocket connection logs: `Connecting to websocket server: %s with version: %d`
  - `No mqtt section found !` and `No websocket section found!` imply dynamic config sections.
- Provisioning:
  - BluFi manager class and event handlers.
  - SoftAP config server with routes `/scan`, `/submit`, `/advanced/config`, `/advanced/submit`, `/saved/*`, `/reboot`.
- Filesystems/storage:
  - `/littlefs`, `LittleFS size: total=%u, used=%u`
  - `/sdcard`, SD init logs
  - USB MSC init logs and restart-to-apply USB disk mode behavior.
- Power/PMIC/battery:
  - `Pmic::Pmic(i2c_master_bus_handle_t, int)`
  - `PowerManagement::PowerManagement(...)`
  - Battery level/voltage logs, charging/discharging states, low-battery policy logs.
- Input/button:
  - `Button::Button(gpio_num_t, bool, uint16_t, uint16_t, bool)`
  - `iot_button_new_gpio_device(...)`
  - Explicit boot button role and long press/double click actions.
- GPIO clue:
  - explicit `GPIO21` strings and wake/availability checks around GPIO21.

## 3. Strongly likely findings
- Product line likely named `Tikpal` with assistant/project codename `xiaozhi`.
- Image is likely from OTA slot (`ota_0.bin` name + app-image format + OTA update logic).
- Dedicated assets partition exists and is checked for free size/content size before assets download.
- Voice/chat pipeline likely cloud-assisted and session-based with websocket signaling + separate UDP audio channel (`OnAudioChannelOpened, udp connect to server ...`).
- UI architecture likely app/screen manager style (AppManager, install/uninstall app flow, multiple app modules e.g., `IdeaChatApp`, `BluetoothFileTransferApp`, `BatteryChargeApp`).
- Touch gestures are app-integrated (swipe up for volume, unified touch/swipe bindings, gesture handler singleton).
- IMU/motion present (QMI8658 init string, shake detection manager/task).

## 4. Speculative findings (explicit hypotheses)
- **Hypothesis:** hardware audio front-end may be multi-mic or digital array because RX is initialized in TDM mode. Could also be single-channel codec using TDM API for flexibility.
- **Hypothesis:** there may be a vendor custom partition `assets_A` storing UI/media bundles, changelog, icons, GIFs, audio prompts.
- **Hypothesis:** OTA server returns JSON with keys like `activation`, `server_time`, `firmware`, optional `mqtt` and `websocket`; runtime chooses transport/endpoints from that response.
- **Hypothesis:** exact display resolution likely configured in board source/constants not leaked in strings; only indirect hints (`Get screen info width/height`) are visible.
- **Hypothesis:** PMIC may expose temperature and battery through I2C + ADC hybrid path (both PMIC class and ADC voltage log exist).

## 5. Full partition analysis
### Hard limitation
Without `partitions.bin` or full flash dump, exact offsets/sizes/labels cannot be confirmed.

### What can still be inferred
| Item | Evidence | Confidence |
|---|---|---|
| App image is ESP-IDF app binary | esptool image header/app info | Confirmed |
| This artifact size is 6 MiB and likely matches OTA slot sizing | file size = 6,291,456 bytes | Strongly likely |
| OTA update targets update partition dynamically | `Failed to get update partition`, `Writing to partition %s at offset 0x%lx` | Confirmed |
| Separate assets storage exists | `assets_A`, `assets partition required`, `Assets download completed` | Strongly likely |
| NVS usage is present | `nvs_open`, `nvs_set_*`, `nvs_commit`, usage log | Confirmed |
| LittleFS partition exists | `/littlefs`, `LittleFS size` | Confirmed |
| SD card external storage exists | `/sdcard`, SD init and failures | Confirmed |

### Recommended extraction targets (priority)
1. `partitions.bin` (or `esptool.py read_flash 0x8000 0x1000`) to obtain exact map.
2. `bootloader.bin` for secure boot / flash enc policy indicators.
3. Full flash dump to carve `assets_A`, NVS/NVS factory, certificates, model blobs.
4. `ota_1.bin` to diff symbols/config and infer migration behavior.

## 6. Board/hardware stack reconstruction
Likely bring-up order (from leaked board functions/logs):
1. `initCodecI2c()` -> codec/pmic I2C infrastructure.
2. `initSpi()` -> SPI2 bus init.
3. `initCO5300Display()` -> SPI LCD panel creation.
4. `initTouch()` -> I2C touch panel init (CST820) + gesture layer.
5. PMIC init (`Init pmic`) + battery/power manager.
6. Boot button and wake sources init.
7. SD card + optional USB MSC + shake/motion feature setup.
8. Wi-Fi/BluFi provisioning + chat/voice service activation.

## 7. Display analysis
- Driver stack: `esp_lcd` + managed component `espressif__esp_lcd_co5300`.
- Bus: SPI (`esp_lcd_new_panel_io_spi(SPI2_HOST, ...)`).
- UI framework: LVGL 9 via `espressif__esp_lvgl_port/src/lvgl9/...` and app wrapper `LvglDisplay`/`SpiLcdDisplay`.
- Backlight: common board backlight component (`./main/boards/common/backlight.cc`), timer-based control (`backlight_timer`).
- Power/display interplay: boot button toggles display; explicit open/close display and low-power transitions.
- Unknowns: exact resolution, pixel format, MADCTL/orientation flags, SPI clock rate, DC/RST/BL GPIOs not leaked as plain strings.

## 8. Touch analysis
- Controller: CST820 (`esp_lcd_touch_new_i2c_cst820`).
- Bus: I2C panel IO (`esp_lcd_new_panel_io_i2c`).
- Integration: touch gesture handler singleton and unified touch+swipe binding to UI container.
- Runtime behavior: swipe up mapped to volume up; wakeup by touchpad exists.
- Unknowns: IRQ pin, RST pin, coordinate transform matrix, polling vs ISR strategy internals.

## 9. Audio analysis
### Confirmed pipeline components
- C++ audio facade in `./main/audio/...`.
- `BoxAudioCodec` class with constructor carrying multiple `gpio_num_t` parameters.
- Full-duplex I2S channel creation (`i2s_new_channel` with TX+RX handles).
- TX initialized in std mode; RX initialized in TDM mode.
- `esp_codec_dev` used for input/output codec devices (open/close/read/write/gain/volume).
- Opus codec in encode/decode path, with dedicated tasking and error handling.
- Wake-word init path exists (`Failed to initialize wake word`).
- Audio channel uses websocket+UDP combination in runtime logs.

### Practical topology (likely)
- Mic path: digital input -> I2S RX (TDM) -> processing (VAD/wake/opus) -> network uplink.
- Speaker path: network/audio source -> decode/resample -> I2S TX -> output codec/amp.
- Control path: I2C to codec/PMIC, volume/gain via `esp_codec_dev`.

### Unknowns
- Exact codec IC model, I2S pin numbers, MCLK requirement, channel count at runtime, AEC/NS/AGC implementation details.

## 10. GPIO and pin mapping clues
### Confirmed clues
- `GPIO21` appears explicitly in multiple logs (`GPIO21`, availability/state checks).
- Boot button exists and is used for wakeup + click/long-press/double-click semantics.
- Audio codec class signatures take several `gpio_num_t` parameters (indicating pin-map passed from board layer).

### Inferred groups (not exact pin numbers)
- SPI2 group for display.
- I2C bus for touch + codec control + PMIC.
- I2S group for duplex audio.
- Dedicated button GPIO (likely including boot button on GPIO21, but direct binding still not explicitly proven in one single string).

### Confidence scoring
- GPIO21 presence: high.
- GPIO21 == boot button pin: medium (strong contextual evidence, no direct assignment string seen).
- Exact SPI/I2C/I2S pinout: low (requires disassembly or additional dumps/logs).

## 11. Buttons / PMIC / storage / SD
- Buttons:
  - `iot_button` framework and custom Button wrapper.
  - Gesture/button interactions: click toggles display, long press switches to chat, double click starts recorder.
- PMIC/battery:
  - Dedicated PMIC class + power management class.
  - Battery level, charging/discharging, low battery popup/mode, temperature sensor handling.
- Storage:
  - NVS settings + usage statistics.
  - LittleFS mounted partition.
  - SD card for recordings/media (`/sdcard/sounds/...`, recordings hierarchy).
  - USB MSC mode support.

## 12. Network / OTA / cloud endpoints
### Confirmed endpoint/artifact inventory
- OTA API base URL: `https://groot.tikpal.io/api/ota/`.
- SoftAP local portal: `http://192.168.4.1`.
- HTTP routes: `/scan`, `/submit`, `/advanced/config`, `/advanced/submit`, `/saved/list`, `/saved/set_default`, `/saved/delete`, `/reboot`.
- Transport mentions: websocket, mqtt section, UDP audio connect/disconnect logs.
- Security/auth hints: activation challenge, `hmac-sha256`, serial_number, CA/client/server cert received in BluFi flows.

### Likely cloud architecture
- Device likely starts with activation/check-version request to OTA backend.
- Backend response may include dynamic websocket/mqtt settings and server time.
- Chat/voice sessions use websocket control + UDP media path.

## 13. Voice assistant pipeline reconstruction
- State/event model appears rich (`XIAOZHI_STATE_EVENTS`, app transitions, chat mode switching).
- Trigger inputs: wake-word and user controls (button long-press, recorder actions).
- Audio encoding/transport: Opus with packet sequencing and encryption/decryption checks.
- Recording subsystem supports WAV persistence, CRC metadata, storage maintenance.
- Likely UX model: always-available assistant with chat mode + push-to-talk/recording fallbacks + on-screen status.

## 14. UI architecture reconstruction
- LVGL 9 based rendering.
- App-style architecture with install/uninstall/start transitions and multiple modules.
- Gesture integration at container level.
- Icons/resources include battery/microphone/speaker symbols and GIF/image assets.
- Localization is extensive: many language bundles embedded in web portal JS (e.g., en-US, ru-RU, zh-CN, etc.).

## 15. Security / openness assessment
### What is evidenced
- App secure version is `0`.
- Firmware integrity hash/checksum valid at image footer.
- Image itself appears plaintext (high symbol/string leakage); not app-level encrypted blob.
- Audio packets have explicit encrypt/decrypt logic in application protocol.

### What is not confirmable without eFuse/bootloader
- Secure Boot enable state.
- Flash encryption enable state.
- JTAG/USB-Serial-JTAG fuse disable state.
- UART ROM download disable state.
- Anti-rollback enforcement state beyond app declaring secure_version 0.

### Practical modding implication
- From app image alone, firmware RE/BSP reconstruction is very feasible.
- Reflashing feasibility cannot be fully guaranteed until eFuse + bootloader policy are checked.

## 16. Gaps / unknowns
- Exact partition map offsets/sizes.
- Exact GPIO pin map (LCD DC/RST/BL, touch INT/RST, I2S BCLK/WS/DIN/DOUT/MCLK, PMIC IRQ, SD pins).
- Exact audio codec/amp silicon part numbers.
- Whether secure boot / flash encryption are actually burned in production units.
- Full cloud endpoint set (only OTA URL is explicit in strings).

## 17. Best next extraction targets
1. Dump/read partition table and bootloader first.
2. Obtain eFuse summary and UART boot log at reset.
3. Run Ghidra/IDA with ESP32-S3 Xtensa profile on `ota_0.bin` mapped segments; recover pin constants from board ctor calls.
4. Extract and parse `assets_A` partition (fonts/images/sounds/config/certs).
5. Collect PCB photos and continuity-map key peripherals to candidate GPIOs.
6. Diff with `ota_1.bin`/older builds to isolate board constants and feature flags.

## 18. Practical guidance for writing custom firmware
### A) Already usable for BSP bootstrap
- SoC/SDK baseline (ESP32-S3 + IDF 5.5.x class APIs).
- Display driver family (CO5300 via esp_lcd SPI) and LVGL9 usage pattern.
- Touch controller family (CST820 over I2C).
- Audio architecture pattern (duplex I2S + codec_dev + Opus + wake-word hooks).
- Power/button/storage/network building blocks and provisioning UX model.

### B) Missing data required to finish BSP
- Precise GPIO map for all major buses and control lines.
- Codec/amp chip IDs and I2C addresses.
- Exact display timing parameters and geometry.
- PMIC model/register map and battery sensing wiring.
- Partition table and NVS key schema.

### C) Feasibility estimate for custom HA voice-hub firmware
- **Rating: moderate**.
- Why not easy: missing exact pin map + uncertain secure-boot/eFuse policy.
- Why not painful/hell: rich symbol leakage, clear board module names, explicit driver stack clues, modern ESP-IDF APIs likely reusable.

## 19. Appendix: raw strings / symbols / evidence list
- Metadata: `xiaozhi`, `2.0.4.20260417`, `Apr 17 2026`, `v5.5.1`.
- Board files: `./main/boards/tikpal/esp32-s3-tikpal.cc`, `pmic.cc`, `touch_gesture_handler.cc`, `blufi_manager.cc`.
- Display/touch: `esp_lcd_new_panel_io_spi`, `esp_lcd_new_panel_co5300`, `esp_lcd_touch_new_i2c_cst820`, lvgl9 managed component paths.
- Audio: `BoxAudioCodec::CreateDuplexChannels(gpio_num_t...)`, `i2s_channel_init_std_mode`, `i2s_channel_init_tdm_mode`, `esp_codec_dev_*`, Opus logs, wake-word init failure log.
- OTA/cloud: `https://groot.tikpal.io/api/ota/`, websocket connect log, mqtt/websocket section logs, activation+hmac strings.
- Storage: `/littlefs`, `/sdcard`, recordings paths, `assets_A`.
- Provisioning: BluFi events, SoftAP web server routes, multilingual HTML/JS resources.
- Power/button: `GPIO21`, boot-button wakeup logs, charging/discharging and low-battery logs, PMIC/power manager classes.
