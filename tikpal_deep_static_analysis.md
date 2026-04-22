# Deep static reverse analysis: `tikpal.bin` (ESP32-S3 target)

## 1. Executive technical summary
- **Ключевой факт:** предоставленный `tikpal.bin` (8,388,608 bytes) по сигнатурам выглядит как **data/assets filesystem image (очень похоже на LittleFS-контент)**, а не как полноценный app image ESP-IDF. Доказательство: много строковых каталогов/ресурсов в начале образа (`littlefs/`, `*.bin`, `*.gif`) и отсутствие валидной таблицы партиций/ESP app header в типовых смещениях.
- Для восстановления **BSP/SoC/IDF/audio/display/touch/network/OTA/security** использованы также уже существующие в рабочей директории артефакты: `esptool_image_info.txt`, `firmware_static_analysis.md`, `strings_ota0.txt` (они содержат app-level метаданные и символические утечки из `ota_0.bin`).
- По совокупности данных:
  - платформа: ESP32-S3,
  - project/app: `xiaozhi`, vendor/product line: `TikPal` (по именам board class/path),
  - стек: ESP-IDF v5.5.1, LVGL9, esp_lcd (CO5300), esp_lcd_touch (CST820), duplex I2S + `esp_codec_dev`, OTA/cloud/websocket/BluFi.
- На базе имеющихся артефактов написание кастомной прошивки для HA voice hub **реально**, но остаются критичные пробелы по точным pin assignments (display/touch/i2s/pmic/buttons).

## 2. Confirmed findings
### From `tikpal.bin` directly
- Размер дампа: `8388608` bytes (8 MiB).
- SHA256: `8b5fddf1c8fd273edc7512f9ff98ca38c82990cb698dbdfe7e48cf3907e852f6`.
- Явные resource/file records:
  - `littlefs/`
  - `apply.bin`, `cancel.bin`, `desktop.bin`, `listening.gif`, `recording.bin`, `settings.bin`, `speaking.gif`, `tools.bin`, `tools-charging.gif`
  - `1.bin..6.bin`, `s1.gif..s6.gif`
  - `bright_vol.bin`, `chat.bin`, `language.bin`, `powersaver.bin`, `reset.bin`, `set.bin`, `skin.bin`
  - `AMBIENT.bin`, `INSPIRATION.bin`, `MEDITATION.bin`, `POMODORO.bin`
- Присутствуют XMP-строки Adobe Photoshop (`CreatorTool="Adobe Photoshop CC 2015.5 (Windows)"`) => assets likely generated/design-exported workflow.

### From app-level companion artifacts (`ota_0.bin` analysis files present in workspace)
- Target chip family: `ESP32-S3`.
- App metadata:
  - Project name: `xiaozhi`
  - App version: `2.0.4.20260417`
  - Compile time: `Apr 17 2026 16:18:52`
  - ESP-IDF: `v5.5.1`
  - Secure version: `0`
- Board/app namespaces and symbols include `TikPalBoard::*`, display/touch/audio init functions, `BoxAudioCodec`, `esp_codec_dev`, `esp_lcd_*`, `esp_lcd_touch_*`, websocket/OTA/provisioning strings.

## 3. Strongly likely findings
- `tikpal.bin` — это **отдельный resource partition dump** (скорее всего LittleFS или похожий FS blob), а не full-flash с bootloader+partition table+app.
- UI/UX ресурсы связаны с voice assistant интерфейсом (`listening`, `speaking`, `recording`, `chat`, `language`, `powersaver`, `skin`, `tools-charging`).
- `AMBIENT/INSPIRATION/MEDITATION/POMODORO` likely относятся к wellness/timer/ambient режимам (UI/аудио сценарии).
- По companion app image: устройство использует комплексный стек voice + touch UI + OTA + cloud session transport.

## 4. Speculative findings (hypotheses)
- **Hypothesis:** `tikpal.bin` может быть экспортом конкретной data partition (например `assets_A`/`littlefs`) из общей flash map.
- **Hypothesis:** дублирующиеся каталожные записи на смещениях с шагом ~0x1000 отражают метаданные/журнал файловой системы.
- **Hypothesis:** часть `*.bin` — preprocessed LVGL binary image/font/icon chunks.

## 5. Full partition analysis
> Ограничение: для `tikpal.bin` не извлечена валидная partition table сигнатура ESP (`0x50AA`) в стандартном смещении 0x8000, и дамп не содержит очевидного app image header в ожидаемом формате. Поэтому полный layout по `tikpal.bin` alone недоступен.

### Что можно утверждать
- Есть data/resource container с контентом, похожим на littlefs-assets.
- Есть признаки companion OTA app partition (из `esptool_image_info.txt` + `firmware_static_analysis.md`), но **внутри `tikpal.bin` это напрямую не видно**.

### Приоритеты следующих extraction targets
1. Снять `partitions.bin` (или full flash с адресами) для точного map.
2. Отдельно извлечь `ota_0.bin`/`ota_1.bin` из той же физической флешки (если текущий `tikpal.bin` — data partition only).
3. Извлечь `nvs`, `nvsfactory`, `otadata`, `assets_A/B` партиции адресно.

## 6. Board/hardware stack reconstruction
(на основании companion app strings/symbol leakage)
- Board abstraction: `TikPalBoard` c methods:
  - `initSpi`, `initCO5300Display`, `initTouch`, `initCodecI2c`.
- Вероятный bring-up order:
  1) codec/pmic i2c,
  2) spi display,
  3) touch,
  4) audio duplex channels,
  5) power mgmt / battery,
  6) connectivity/provisioning,
  7) app manager + UI states.

## 7. Display analysis
- Confirmed in companion app artifacts:
  - `esp_lcd_new_panel_io_spi(SPI2_HOST...)`
  - `esp_lcd_new_panel_co5300(...)`
  - LVGL9 integration via `esp_lvgl_port`.
- Likely:
  - Bus = SPI;
  - panel driver = CO5300;
  - custom wrappers: `LvglDisplay`, `SpiLcdDisplay`;
  - backlight control via board/common backlight component.
- Unknown critical values:
  - exact resolution, rotation, color format, SPI freq, DC/RST/BL pins.

## 8. Touch analysis
- Confirmed in companion app artifacts:
  - `esp_lcd_new_panel_io_i2c(...)`
  - `esp_lcd_touch_new_i2c_cst820(...)`.
- Strongly likely: CST820 I2C touch controller + gesture layer integrated with UI.
- Unknown: IRQ/RST GPIO, transform/calibration matrix.

## 9. Audio analysis
- Confirmed in companion app artifacts:
  - class `BoxAudioCodec` with multiple `gpio_num_t` ctor args;
  - duplex channel setup (`i2s_new_channel`), TX std mode, RX tdm mode;
  - `esp_codec_dev` API usage for read/write/gain/volume;
  - Opus encode/decode paths;
  - wake-word init hooks.
- Likely topology:
  - Mic path: I2S RX(TDM) -> processing -> Opus -> network.
  - Speaker path: network -> decode -> I2S TX -> codec/amp.
- Unknown:
  - exact codec IC, exact i2s pins/clocking, AEC/NS/AGC engine implementation.

## 10. GPIO and pin mapping clues
- `tikpal.bin` (assets) direct GPIO clues: none reliable.
- Companion app clues:
  - explicit `GPIO21` string appears;
  - `Button::Button(gpio_num_t, ...)` and board init functions indicate configurable pin map.
- Confidence split:
  - Confirmed: GPIO abstraction heavy usage.
  - Strongly likely: dedicated boot/wake button GPIO exists.
  - Speculative: GPIO21 may be one of key inputs (needs direct assignment evidence from code/disassembly/UART logs).

## 11. Buttons / PMIC / storage / SD
- Buttons (companion app): iot_button framework, click/long-press/double-click actions.
- PMIC/battery (companion app): PMIC class + charging/discharging + low battery logic.
- Storage:
  - `tikpal.bin`: clear evidence of UI/media assets container;
  - companion app: LittleFS + SD card + USB MSC logic.

## 12. Network / OTA / cloud endpoints
- Companion app confirmed strings:
  - OTA base URL: `https://groot.tikpal.io/api/ota/`
  - Provisioning AP portal: `http://192.168.4.1`
  - route hints: `/scan`, `/submit`, `/advanced/config`, `/advanced/submit`, `/saved/list`, `/saved/set_default`, `/saved/delete`, `/reboot`
  - websocket + optional mqtt section references.
- Likely architecture: dynamic backend config (websocket/mqtt sections), OTA check/update, cloud session for assistant.

## 13. Voice assistant pipeline reconstruction
- Companion artifacts indicate:
  - assistant state machine/events;
  - wake-word init path;
  - Opus transport;
  - audio streaming/session handling (websocket + UDP hints).
- Likely UX model: always-on voice assistant + fallback through button/recording interactions + touch UI feedback.

## 14. UI architecture reconstruction
- `tikpal.bin` confirms rich visual asset set with listening/speaking/recording/chat/settings/skin/powersaver semantics.
- Companion app confirms LVGL9 and app-manager style architecture with multiple app modules.
- Likely localization exists (companion strings mention multiple locale bundles).

## 15. Security / openness assessment
- Directly from `tikpal.bin`: security fuses/boot flags are **not observable**.
- Companion app metadata: `Secure version: 0` (anti-rollback version field in app desc).
- Not confirmed without bootloader + efuse dump:
  - secure boot enable,
  - flash encryption enable,
  - JTAG disable,
  - ROM download disable.
- Modding implication: пока нет blocker evidence, но без eFuse/bootloader нельзя оценить reflashing feasibility окончательно.

## 16. Gaps / unknowns
Critical unknowns for full BSP recreation:
- exact partition map (offsets/sizes/types),
- exact GPIO map for display/touch/audio/pmic/buttons/sd,
- exact codec and PMIC IC models,
- secure boot / flash encryption / JTAG policy,
- exact panel resolution/orientation/timing.

## 17. Best next extraction targets
1. `espefuse.py summary` (или UART boot logs + efuse dump) для security posture.
2. `esptool.py read_flash` по адресам партиций (особенно `0x8000` partition table region, bootloader region, OTA slots, nvs, otadata).
3. Carve app images и прогнать `esptool image_info` + `strings -td` + Ghidra/IDA.
4. Mount/extract `tikpal.bin` как littlefs (получить полный список файлов, размеры, magic/signatures).
5. Получить фото PCB и прозвонить линии дисплея/тача/кодека/PMIC для финализации pin map.
6. Снять runtime UART logs boot+init для pin printouts and driver init traces.

## 18. Practical guidance for writing custom firmware
### Можно ли сделать кастомный HA voice hub на этой базе?
**Да, вероятнее всего можно (оценка: MODERATE)**, потому что:
- стек устройства уже соответствует задаче (touch UI, audio duplex, wake-word path, network OTA/cloud, power mgmt);
- ESP32-S3 + ESP-IDF v5.5.1 + LVGL9/esp_lcd/esp_codec_dev — хороший фундамент.

### Что уже известно и полезно
- likely board family: TikPal/xiaozhi line;
- display stack: SPI + CO5300 + LVGL9;
- touch: CST820 over I2C;
- audio: duplex I2S + codec abstraction + opus pipeline;
- connectivity: Wi-Fi/BLE(BluFi)/websocket/OTA;
- assets: отдельный littlefs-like ресурсный blob подтверждён.

### Что ещё обязательно добить
- точные GPIO pin assignments (display/touch/audio/pmic/buttons/sd),
- точные power rails / enable pins,
- security fuse state,
- exact partition map.

## 19. Appendix: raw strings / symbols / evidence list
### `tikpal.bin` direct evidence (offset -> string)
- `0x000008`: `littlefs/`
- `0x000030`: `apply.bin 0`
- `0x000092`: `listening.gif 0`
- `0x0000af`: `recording.bin 0`
- `0x000100`: `speaking.gif 0`
- `0x000131`: `tools-charging.gif 0`
- `0x03e008`: `1.bin 0` ... `0x03e0f4`: `s6.gif 0`
- `0x0ce008`: `bright_vol.bin 0`, `chat.bin 0`, `language.bin 0`, `powersaver.bin 0`, `reset.bin 0`, `set.bin 0`, `skin.bin 0`
- `0x0fb008`: `AMBIENT.bin 0`, `INSPIRATION.bin 0`, `MEDITATION.bin 0`, `POMODORO.bin 0`
- multiple XMP Adobe strings with `xmp:CreatorTool="Adobe Photoshop CC 2015.5 (Windows)"`

### Companion app evidence files used
- `esptool_image_info.txt` (ESP32-S3, xiaozhi, version/time/IDF/secure_version)
- `firmware_static_analysis.md` (consolidated symbol-level findings)
- `strings_ota0.txt` (raw leaked strings from OTA app)
