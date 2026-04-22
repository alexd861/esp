# ESP32-S3 OTA image deep static analysis (`ota_1.bin`)

Date: 2026-04-22 (UTC)
Scope: static-only analysis of a **single OTA app image** (not full flash dump)

## 1) Executive technical summary
- The image is an ESP32-S3 application for product/project `xiaozhi` / `tikpal`, built with ESP-IDF `v5.5.1`, app version `2.0.4.20260417`, compile timestamp `Apr 17 2026 16:18:52`.
- Firmware is C++-heavy and semi-symbolic (many class names, source paths, method signatures are present).
- Board abstraction strongly points to `TikPalBoard` (`./main/boards/tikpal/esp32-s3-tikpal.cc`) with dedicated init routines for display (`CO5300`), touch (`CST820 over I2C`), SPI bus, codec I2C, PMIC, power management, BluFi provisioning.
- Display stack is LVGL + esp_lcd with SPI panel (`esp_lcd_new_panel_io_spi`) and vendor panel driver (`esp_lcd_new_panel_co5300`), plus touch via `esp_lcd_touch_new_i2c_cst820`.
- Audio stack indicates duplex path with I2S TX standard mode + RX TDM mode, codec chips `ES8311` + `ES7210`, Opus encode/decode wrappers, and wake-word hooks.
- Connectivity includes Wi-Fi + BLE (BluFi provisioning), MQTT and WebSocket transports, OTA URL default `https://groot.tikpal.io/api/ota/`, plus custom OTA URL UI path.
- Storage/assets architecture includes NVS, LittleFS, SPIFFS model loading path, SD-card in SPI mode, and an explicit assets partition workflow (`assets_A`) with mmap/write/update logic.
- Security posture cannot be fully determined from app image alone; `Secure version: 0` and standard hash/checksum are present, but eFuse runtime state (secure boot/flash encryption/JTAG fuse) is unknown without bootloader/eFuse dump.

## 2) Confirmed findings
### High-level identification
- SoC family: `ESP32-S3` (image header and chip ID).
- Project name: `xiaozhi`.
- App version: `2.0.4.20260417`.
- Build time: `Apr 17 2026 16:18:52`.
- ESP-IDF version: `v5.5.1`.
- Secure version field: `0`.
- Board/product identifiers in strings: `tikpal`, `TikPalBoard`, `esp32-s3-tikpal.cc`.

### Framework/components
- LVGL integration present (`Initialize LVGL library`, `lvgl_lcd_port_init`, `esp_lvgl_port`).
- esp_lcd integration present (`esp_lcd_new_panel_io_spi`, `esp_lcd_new_panel_co5300`, `esp_lcd_new_panel_io_i2c`).
- Touch driver integration present (`esp_lcd_touch_new_i2c_cst820`).
- Audio codec stack present (`BoxAudioCodec`, `esp_codec_dev_*`, `ES8311`, `ES7210`).
- Networking stacks include MQTT and WebSocket.

### Source path leakage / architecture
- Paths include:
  - `./main/boards/tikpal/esp32-s3-tikpal.cc`
  - `./main/boards/tikpal/blufi_manager.cc`
  - `./main/boards/tikpal/pmic.cc`
  - `./main/boards/tikpal/power_manager.cc`
  - `./main/boards/tikpal/touch_gesture_handler.cc`
  - `./main/audio/audio_codec.cc`
  - `./main/audio/codecs/box_audio_codec.cc`
  - `./main/display/lcd_display.cc`
  - `./main/display/lvgl_display/lvgl_display.cc`

### OTA/cloud evidence
- Explicit OTA endpoint string: `https://groot.tikpal.io/api/ota/`.
- Settings/UI supports `ota_url` override (`Custom OTA URL`).

## 3) Strongly likely findings
- This binary is **not** full flash dump; it is a single OTA app image (esptool app image metadata + no direct partition table entries as binary table).
- Device UX is voice-first + touch-first hybrid:
  - wake-word detection,
  - swipe gestures controlling volume,
  - long-press to switch chat mode,
  - double-click to start recorder.
- Product likely supports field app/extensions (AppManager, Bluetooth file transfer app, BatteryCharge app, fidget toy tools).
- Assets likely externalized and versioned (`index.json`, `fonts.bin`, `srmodels.bin`, assets download/update workflow).

## 4) Speculative findings (hypothesis)
- **Hypothesis:** Display controller `CO5300` may be custom/vendor-specific panel IC wrapper (name appears as esp_lcd component `espressif__esp_lcd_co5300`), likely SPI TFT/OLED class.
- **Hypothesis:** Audio front-end likely 1x playback codec + multichannel mic ADC topology (`ES8311` playback + `ES7210` mic array), due simultaneous references and duplex init pattern.
- **Hypothesis:** Board likely battery-powered portable with USB charging + PMIC control + motor/haptic + accelerometer (`QMI8658`) based on strings.

## 5) Full partition analysis (what is possible from this artifact)
### Hard limitation
Because input is OTA application image (6 MiB) rather than flash dump including partition table sector, exact on-flash offsets/sizes for all partitions cannot be fully reconstructed.

### Confirmed partition semantics from app logic
- OTA logic uses running/update partition APIs and otadata handling.
- Factory partition awareness (`Running from factory partition, skipping`).
- Assets partition handling (find/mmap/size checks/write/reinit).
- Potential labels seen in strings: `assets_A`, `littlefs`, SPIFFS partition label usage for models.

### Likely partition set (inferred from APIs and strings)
- `nvs`
- `otadata`
- `phy_init`
- `factory` and `ota_x` slots
- `assets_A` (custom data partition)
- `littlefs` data partition
- SPIFFS partition for speech/models (`srmodels.bin`)
- possibly SD card external FATFS (not partition table entry, external media)

### Recommended next extraction targets (priority)
1. Full flash dump including partition table sector (offset `0x8000` on typical ESP-IDF layout).
2. Bootloader region dump (for secure boot / flash enc behavior clues).
3. NVS and factory NVS partitions for calibration/model/pin/config keys.
4. `assets_A` raw extraction and carve for `index.json`, fonts, model blobs.

## 6) Board/hardware stack reconstruction
- Board class: `TikPalBoard`.
- Init routines observed:
  - `initCO5300Display()`
  - `initTouch()`
  - `initCodecI2c()`
  - `initSpi()`
  - `initPmic()` (inferred from lambda symbols + log)
- Supporting board components:
  - `Pmic`
  - `PowerManagement`
  - `PowerSaveTimer`
  - `Backlight`
  - `Button`/`BootButton`
  - `TouchGestureHandler`
  - `BluFiManager`

### Likely bring-up order (inferred)
1. Core board power/wakeup checks.
2. I2C master init (`initCodecI2c`) and PMIC.
3. SPI bus init (`initSpi`) + display panel.
4. Touch controller init over I2C.
5. Audio codec/i2s duplex channels.
6. SD card + filesystem/services.
7. Wi-Fi/BluFi provisioning stack.

## 7) Display analysis
### Confirmed
- LVGL + esp_lcd stack.
- SPI display IO via `esp_lcd_new_panel_io_spi(SPI2_HOST, ...)`.
- Panel creation via `esp_lcd_new_panel_co5300(...)`.
- LVGL display classes: `LvglDisplay`, `SpiLcdDisplay`, `CustomLcdDisplay`.
- Backlight class present, LEDC symbols present.

### Strongly likely
- Color mode likely RGB565 (esp_lvgl_port RGB565 constraints/errors present).
- Flush/DMA strategy likely via esp_lvgl_port callbacks (strings from esp_lvgl_port internals).

### Unknown
- Exact resolution, MADCTL settings, SPI clock, DC/RST/BL GPIO values.

## 8) Touch analysis
### Confirmed
- Touch controller path: `esp_lcd_touch_new_i2c_cst820(...)` => CST820 over I2C.
- Touch gestures subsystem exists (`TouchGestureHandler`, swipe detection logs).
- Integration with UI events (`Unified touch and swipe events bound to container`).

### Likely
- Interrupt-driven + periodic gesture timer mixed approach (gesture timer symbols + touch init logs).

### Unknown
- Exact IRQ/RST pins, coordinate transforms/calibration matrix.

## 9) Audio analysis
### Confirmed
- Main abstraction: `AudioCodec`, concrete `BoxAudioCodec`.
- Duplex constructor accepts many GPIO params (`BoxAudioCodec::BoxAudioCodec(...gpio_num_t...)`).
- Explicit channel creation: `CreateDuplexChannels(gpio_num_t, ... x5)`.
- I2S usage: TX in std mode, RX in TDM mode.
- Codec device layer: `esp_codec_dev_read/write/open/close`.
- Codec chips present in image: `ES8311`, `ES7210`.
- Opus encode/decode stack present (`OpusEncoderWrapper`, `OpusDecoderWrapper`, `OpusResampler`).
- Wake-word init hook exists (`Failed to initialize wake word`).

### Strongly likely topology
- Playback path: ESP32-S3 I2S TX -> ES8311 DAC/codec -> speaker amp.
- Capture path: mic(s) -> ES7210 ADC -> ESP32-S3 I2S RX (TDM for multichannel mic).
- Full duplex conversational pipeline with server/device sample rate adaptation (resampling warning).

### Unknown
- Exact sample rates/channels in normal operation.
- AEC/NS/AGC library identity (not directly leaked by unique symbols).
- Amp enable/mute GPIO.

## 10) GPIO and pin mapping clues
### Confirmed GPIOs
- `GPIO21` explicitly appears in multiple runtime logs, including wakeup checks and availability logs.

### Confirmed pin-bearing APIs (but values unknown)
- `Button::Button(gpio_num_t, ...)`
- `CreateDuplexChannels(gpio_num_t, gpio_num_t, gpio_num_t, gpio_num_t, gpio_num_t)`
- `BoxAudioCodec::BoxAudioCodec(... gpio_num_t x6 ..., uint8_t, uint8_t, bool)`
- Wakeup source configured for specific GPIO (`GPIO%d`).

### Inferred groups
- SPI2 host used for display and also SD SPI mode path.
- I2C bus shared by touch + PMIC + codec control plane.

### Unresolved critical signals
- Display: MOSI/MISO/SCLK/CS/DC/RST/BL
- Touch: SDA/SCL/INT/RST
- Audio: BCLK/WS/DIN/DOUT/MCLK + codec reset/PA enable
- Buttons: boot + additional keys
- PMIC IRQ/EN/charge detect pins

## 11) Buttons / PMIC / storage / SD
### Buttons/input
- Boot button framework present (`BootButton`, click/long press/double click actions).
- Actions mapped to UI/voice behavior (toggle display, chat mode, recorder start).

### PMIC/power/battery
- PMIC abstraction class exists (`Pmic`, `init pmic`).
- Battery monitoring/low-battery UX present.
- Charge mode transitions and power source info support present.
- Deep sleep wake reasons handled (GPIO/timer/touch/ULP).

### Storage/SD/filesystems
- SD card manager and SPI-mode init logs are present.
- LittleFS manager present (`/littlefs`).
- SPIFFS used for model loading fallback/path (`srmodels.bin`).
- Asset and media files referenced from `/tikpal/...` and `/sdcard/...`.

## 12) Network / OTA / cloud endpoints
### Confirmed transports
- Wi-Fi station/AP flows.
- BLE + BluFi provisioning.
- MQTT transport.
- WebSocket transport.

### Confirmed endpoint/config clues
- OTA endpoint: `https://groot.tikpal.io/api/ota/`.
- Configurable custom OTA URL in settings portal.
- Strings indicate optional protocol selection (MQTT fallback if unspecified).

### Likely cloud architecture
- Device can communicate via MQTT or WebSocket to backend.
- OTA metadata/config likely server-driven and user-overridable.
- TLS cert validation and certificate bundle usage enabled.

## 13) Voice assistant pipeline reconstruction
### Confirmed
- Wake-word detection hooks/logs.
- Opus codec stack and sample-rate adaptation.
- Audio speaker control API strings.
- Conversational status/notification framework exists.

### Strongly likely UX model
- Always-listening or conditionally listening wake-word model plus button fallback (double-click recorder).
- Full-duplex-capable pipeline or near-full-duplex with role transitions based on app state.
- Cloud-backed NLU/ASR/TTS likely, with local wake-word/VAD front-end.

## 14) UI architecture reconstruction
### Confirmed
- LVGL-based UI (`LvglDisplay`, lvgl port init/task).
- App/page manager semantics (`AppManager`, app install/uninstall/open/transition logs).
- Touch gesture integration into container events.
- Multilingual settings web UI strings embedded (many translated `ota_url` labels).

### Likely
- Internal app-based UX modules (chat, settings, bluetooth transfer, battery charge, tools).
- State/event bus (`XIAOZHI_STATE_EVENTS`, callbacks) driving screen updates.

## 15) Security / openness assessment
### Confirmed evidence
- App image contains valid checksum/hash and `Secure version: 0` in metadata.
- TLS/cert verification codepaths are present.

### Unknown from this artifact alone
- eFuse secure boot state.
- eFuse flash encryption state.
- JTAG disable fuses.
- UART/USB ROM download disable state.

### Practical implication
- Custom reflashing feasibility cannot be concluded safely without bootloader + eFuse readout.
- If secure boot and flash encryption are off in eFuse, custom ESP-IDF firmware should be straightforward.
- If secure boot is on (and keys unavailable), unsigned custom app boot is blocked.

## 16) Gaps / unknowns
- Exact partition table offsets/sizes/types/subtypes.
- Exact GPIO map for display/touch/audio/PMIC/buttons/SD.
- Exact display resolution/timing and touch transform config.
- Exact audio clocking and channel map.
- Exact backend hostnames beyond OTA URL.

## 17) Best next extraction targets
1. Full flash dump (bootloader + partition table + all partitions).
2. eFuse dump (`espefuse.py summary`) from live device.
3. Extract and parse `assets_A`, LittleFS, SPIFFS partitions.
4. Recover additional symbols by lifting image into Ghidra with ESP32-S3 Xtensa settings.
5. UART boot/runtime logs (boot mode, flash enc/secure boot prints, pin init logs).
6. PCB photos + continuity probing for definitive GPIO mapping.

## 18) Practical guidance for custom firmware
### What is already enough for BSP bootstrap
- Chip/SDK baseline (ESP32-S3 + ESP-IDF 5.5.x).
- Display family (SPI + CO5300 wrapper) and touch family (CST820 I2C).
- Audio subsystem direction (ES8311 + ES7210 via I2S + codec dev abstraction).
- Power/button/storage/network building blocks and voice-oriented software architecture.

### Missing must-have parameters
- Pin map for display/touch/audio/backlight/PMIC/buttons/SD.
- Exact panel resolution/init sequence compatibility.
- Exact mic topology and I2S line mapping.
- Device security fuse state.

### Feasibility for Home Assistant voice hub firmware
- Estimated difficulty: **moderate** if security is open and pins are recovered quickly.
- Could become **painful** if secure boot/flash encryption enabled or if panel/controller init is vendor-custom without source.

## 19) Appendix: raw evidence inventory
### Key metadata (`esptool_image_info.txt`)
- `Detected image type: ESP32-S3`
- `Project name: xiaozhi`
- `App version: 2.0.4.20260417`
- `Compile time: Apr 17 2026 16:18:52`
- `ESP-IDF: v5.5.1`
- `Secure version: 0`

### Key subsystem strings (`strings_ota0.txt`)
- Display/touch:
  - `void TikPalBoard::initCO5300Display()`
  - `esp_lcd_new_panel_co5300(...)`
  - `void TikPalBoard::initTouch()`
  - `esp_lcd_touch_new_i2c_cst820(...)`
- Audio:
  - `BoxAudioCodec::BoxAudioCodec(...gpio_num_t...)`
  - `CreateDuplexChannels(gpio_num_t, ... x5)`
  - `i2s_channel_init_std_mode(...)`
  - `i2s_channel_init_tdm_mode(...)`
  - `ES8311`, `ES7210`, `OpusEncoderWrapper`, `OpusDecoderWrapper`
- Network/OTA:
  - `https://groot.tikpal.io/api/ota/`
  - `No protocol specified in the OTA config, using MQTT`
  - `Connecting to websocket server: %s with version: %d`
- Storage/assets:
  - `assets_A`
  - `No assets partition found`
  - `Initializing models from SPIFFS, partition label: %s`
  - `/littlefs`
  - `Initializing SD card in SPI mode`
