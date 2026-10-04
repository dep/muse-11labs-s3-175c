# This Muse gadget

Notes for the Waveshare board on this desk. The SDK docs live under `muse-gadget-sdk/`. This file records the hardware, current firmware behavior, resolved issues, and details needed to continue work.

Do not print or commit secrets from `.env`.

## Current status — 4 Oct 2026

All previously documented issues are resolved. ElevenLabs speech is working, confirmed by the user; the certificate error below is historical troubleshooting information, not an open issue.

| Resolved issue | Fix or current behavior |
|---|---|
| ElevenLabs replies stayed silent | Correct secret key and Cute Emily voice ID, chunked-response handling, and cross-signed certificate verification. Live playback was verified after the 3 Oct flash. |
| Floating top-button input triggered presses | Internal pull-down on active-high GPIO3, debounce, and a sustained press before listening. |
| Accidental touch or tap wake/send | One-second button hold to wake this 1.75C; sleeping touch is swallowed and face taps only animate. |
| Mascot overlapped reply text | Mascot hidden during thinking and speaking; replies use one left-aligned page. |

The latest bottom-button report prompted a code review, not a firmware change. A sleeping display and a fully powered-off board have different wake controls (below). No board was visible over USB during that review, so the reported failure was not reproduced or verified on hardware. If it recurs, capture the button/wake logs before attributing it to a firmware defect.

## Hardware

| Item | Value |
|---|---|
| Board | Waveshare ESP32-S3-Touch-AMOLED-1.75C |
| Build profile | `waveshare-s3-175c` (`s3` in `tools/muse/board.sh`) |
| Build directory | `muse-gadget-sdk/esp32/build-muse-waveshare-s3-175c` |
| USB port | `/dev/cu.usbmodem1101` (USB-Serial/JTAG; the name can change) |
| MAC | `80:45:6b:34:24:08` |
| Display | Round 466 px CO5300 AMOLED, CST9217 touch |
| Audio | ES8311 speaker, ES7210 dual mic, 16 kHz |
| PMU | AXP2101. DCDC1 is VCC3V3. ALDO1 is A3V3 for the codecs |
| Top button | PWR, talk. GPIO3, active high. A BSS138 inverts the key |
| Bottom button | BOOT, aux. GPIO0 |

The non-C 1.75 uses the same driver. It swaps talk and aux (BOOT talks, PWR is aux) and it does not set `wake_hold_ms`. Do not apply the hold-to-wake behavior to that board.

## Wake and buttons

Only this 1.75C sets `wake_hold_ms = 1000` on `muse_board_t`. Zero means the old tap or touch wake.

While the display is asleep and the board is still powered:

- A touch does not wake the screen, and a touch never sends a message. The sleep cover stays up and swallows the contact when `wake_hold_ms > 0`. A touch while asleep does not reset the idle timer.
- Hold either button for 1000 ms to wake. The line must stay down through the debounce. One high sample is not a press.
- Keep holding the top button after that. Listening starts about 300 ms later (`MIN_HELD_FRAMES` in `muse_voice.c`). A release as the screen lights only wakes it.
- The bottom hold only turns the display on. That hold is swallowed so it does not continue into power-off. Release, then hold again with the screen on, to power off.
- Pairing prompts, incoming images, and serial `w` can still light the screen.

While the screen is already on:

- The top button starts a listen only after it stays down. `TALK_ARM_MS` (300 ms) in `muse_input.c` blocks a spike from posting push-to-talk. `record()` then waits `MIN_HELD_FRAMES` (another 300 ms) before it shows "LISTENING" or opens a server turn. A shorter press sends nothing.
- A tap on the face does not send. It only plays the hop.
- A short bottom press sleeps the screen.
- A hold of about 1.5 s on the bottom button powers off.

After a full power-off, use the **top PWR button** to turn the board back on. The bottom BOOT button cannot start an unpowered board. A bottom hold that wakes a sleeping display is swallowed until release; a separate hold while awake can shut down the board. Two quick bottom presses toggle phone setup; the single-press sleep waits about 350 ms to distinguish them.

GPIO3 is active high and had no pull-down, so a floating line read as a stream of presses. `button_init()` now enables the internal pull-down on an active-high input. That is the only active-high button.

The input poll is in `muse_input.c`. While a hold is armed, the paused `wait_buttons` timeout is the time left in the hold, so a 10 s Wi-Fi nap cannot delay the 1 s wake.

## Answer screen

When a reply is on screen, the mascot canvas is hidden for the whole thinking and speaking layout. Heard and read replies share one left-aligned page from `fill_reply()` in `muse_ui.c`. The face comes back when the reply ends. The speaker button stays in the corner, clear of the words. The short "sending / waiting" phase uses the same layout, so the mascot hides there too.

## Speech

Speak replies is the existing speaker setting. Default is on. NVS key is `"speaker"`. Volume default is 70, NVS key `"volume"`.

Turn it off in Sound, or hold the speaker button on the face. Off keeps the answer on screen and paces it with silence (`TEXT_CHARS_PER_S`). `muse_audio_write()` still clocks that silence, and it writes zeros while the setting is off.

On, each finished voice reply is spoken with ElevenLabs from the hatch task in `muse_chat_session.cpp`:

- `POST https://api.elevenlabs.io/v1/text-to-speech/{voice}/stream?output_format=mp3_22050_32`
- Header `xi-api-key`. JSON `{"text":"...","model_id":"eleven_flash_v2_5"}`. `Accept: audio/mpeg`
- The body is chunked. `esp_http_client_fetch_headers()` returns 0 for chunked, and 0 is success. A non-200 status falls back to silent text.
- MP3 is decoded with minimp3, resampled from 22050 Hz to 16 kHz, and played on the same path as the old reply audio.
- A typed `chat=` turn does not speak. It skips speech on purpose.

The key and voice are compiled in. They are not read at runtime.

- Repo file: `/Users/dep/projects/muse/.env`
- `ELEVENLABS_API_KEY` must start with `sk_`. A 64-character hex value is an API key id. ElevenLabs rejects it with `api_key_id_used_as_api_key`. The build leaves the key empty in that case, and the firmware does not call the network.
- Optional `ELEVENLABS_VOICE_ID` overrides the default. This `.env` does not set it.
- CMake reads `${COMPONENT_DIR}/../../../../.env` (four levels up to this workspace root). Three levels is `muse-gadget-sdk/.env`, which is the wrong file.
- The filled header is `build-muse-waveshare-s3-175c/esp-idf/muse/elevenlabs_key.h`. It is not in the source tree. Changing `.env` does nothing until CMake runs again (`idf.py reconfigure`).
- The CMake status line says `key loaded` or `no secret key`. It does not print the key.

Voice id for Cute Emily, 20 characters. Character 9 is the digit zero. Character 10 is capital O:

`iYSNpS2X0Oqe8aXoNn0j`

A 19-character copy that drops the capital O is a different id. ElevenLabs returns `voice_not_found`, and the gadget shows the text with no audio. The host can confirm the id with `GET /v1/voices` (name `Cute Emily`, category `generated`). `POST .../stream` with `eleven_flash_v2_5` and `mp3_22050_32` returns HTTP 200, `Content-Type: audio/mpeg`, chunked, and the body starts with an ID3 header.

`.env` also holds `MUSE_SDK_TOKEN` and `DEEPGRAM_API_KEY`. Leave them alone.

## Resolved speech certificate issue

ElevenLabs speech and the speaker work. Serial `m` plays `test_reply.mp3` ("Hi, I'm Muse!"). Before the fix, a live reply stayed text-only because TLS rejected the ElevenLabs certificate. These are the old failure logs, retained to identify a regression:

```
E esp-x509-crt-bundle: Failed to verify certificate
E esp-tls-mbedtls: mbedtls_ssl_handshake returned -0x3000
E transport_base: Failed to open a new connection
E HTTP_CLIENT: Connection failed, sock < 0
W muse_chat_session: elevenlabs HTTP 0
I muse_chat_session: showing message ...
```

`-0x3000` is `MBEDTLS_ERR_X509_CERT_VERIFY_FAILED`. The site cert is `CN=elevenlabs.io`, issued by Google Trust Services `WR3`. The server chain is WR3, then GTS Root R1 cross-signed by `GlobalSign Root CA`. The IDF bundle has the self-signed GTS Root R1, and GlobalSign Root CA - R3 / R6, not that original GlobalSign root. `CONFIG_MBEDTLS_CERTIFICATE_BUNDLE_CROSS_SIGNED_VERIFY` accepts the cross-signed root when its key matches the bundled one. It is set in `sdkconfig.defaults`. It costs about 700 bytes of heap. Do not turn it off or ElevenLabs goes silent again.

`showing message` means the reply is paced in silence. It can be expected when Speak replies is off, or occur when the key/text is unavailable or the HTTPS call fails; the line alone does not prove a TLS failure. `speaking message` means the speech stream opened; subsequent decode/playback logs confirm audio reached the playback path.

Verified after the 3 Oct 2026 flash. A short voice turn logged `speaking message`, `reply audio: 22050 Hz, 1 ch, 32 kbps`, and `muse_voice: reply audio`. The certificate error was gone.

## Build and flash

ESP-IDF v6 is `$HOME/esp/esp-idf-v6`. `idf.py` is on `PATH` only after:

```sh
. "$HOME/esp/esp-idf-v6/export.sh"
```

Python: `/Users/dep/.espressif/python_env/idf6.0_py3.13_env`.

Use the activated IDF environment for the serial tools too: the system `python3` lacked `pyserial` during the 4 Oct review, while the IDF Python could run them.

From `muse-gadget-sdk/esp32`, incremental build:

```sh
idf.py -B build-muse-waveshare-s3-175c build
```

`tools/muse/board.sh` is a full build. It wipes `managed_components`. Do not use it for a one-file change.

CMakeCache `SDKCONFIG_DEFAULTS` includes `/tmp/sdkconfig.muse-token`. That file is not in the repo. If configure fails because it is missing, an empty replacement is sufficient only when the existing build's `sdkconfig` already retains its SDK token. A fresh build needs the token supplied securely; never print it. A full `idf.py reconfigure` may log `Deleting 20 unused components` (LVGL, the Waveshare BSP, and others). The directories were still there afterward, and the board still linked. Check `managed_components/lvgl__lvgl` and `managed_components/waveshare__esp32_s3_touch_amoled_1_75c` if a later configure looks empty.

Before flashing, identify the attached board and current port from `muse-gadget-sdk/esp32` with the IDF environment activated:

```sh
python tools/muse/ports.py --list
python tools/muse/chat.py --status
```

The port in the hardware table is historical; use the detected port. If none appears, connect a data cable and turn the board on with top PWR. Follow `esp32/AGENTS.md` for identification fallbacks. Do not erase flash to troubleshoot these issues: normal flashing preserves NVS pairing, Wi-Fi credentials, and settings.

Flash from the build directory. `@flash_args` is relative to that directory:

```sh
cd muse-gadget-sdk/esp32/build-muse-waveshare-s3-175c
python -m esptool --chip esp32s3 -p /dev/cu.usbmodem1101 -b 460800 \
  --before default-reset --after hard-reset write-flash "@flash_args"
```

Use the IDF Python above if `python` is not that env. The write is verified by hash, then the chip hard-resets.

## Serial bench

The USB console is also the log. Useful single bytes, from `muse_input.c`:

| Key | Action |
|---|---|
| `m` | Play the built-in MP3 through the speaker |
| `d` / `u` | Talk button down / up |
| `w` / `z` | Wake / sleep |
| `>` then a line | Setup or console command (`status`, `chat=`, `face=`) |

`chat=` sends a typed turn. Typed turns do not speak.

Log level is info. Speech lines use the tag `muse_chat_session`. `speaking message` means ElevenLabs opened. `showing message` means silent text, including when Speak replies is intentionally off. Test speech with a physical voice turn; typed `chat=` and `chat.py` turns intentionally do not speak. The `m` self-test verifies the local speaker/decoder path, not ElevenLabs connectivity.

From `muse-gadget-sdk/esp32`, `python tools/muse/monitor.py PORT 30` resets the board and captures boot logs. For a failure that depends on the current sleep state, use the read-only capture recipe in `AGENTS.md` instead; resetting first loses that state. Bottom wake should log `muse_input: waking (bottom)`. Record whether the top button still wakes it, whether the board was fully shut down, and whether it was on USB or battery. Serial `w` bypasses the physical button and hold logic, so it cannot verify bottom-button wake by itself.

## Future-agent handoff

- This README and `.env` are in `/Users/dep/projects/muse`, outside the Git repository. The repository is `/Users/dep/projects/muse/muse-gadget-sdk`; run Git commands there. Keep this workspace README updated as well as any relevant SDK documentation.
- The repository already has uncommitted changes across input, board, UI, speech, settings, and certificate configuration, plus the new `esp32/components/muse/elevenlabs_key.h.in`. Inspect `git status` and preserve that work. The template must accompany the CMake changes; generated `elevenlabs_key.h` contains the secret and must not be committed or printed. Firmware binaries also contain the compiled secret.
- Within `esp32/components/muse/`: input behavior lives in `muse_input.c`; GPIO debounce and interrupt waits in `muse_board.c`; this board's pin map, panel sleep, and touch pause/resume in `boards/board_waveshare_s3_175c.c`; screen sleep/brightness and reply layout in `muse_ui.c`.
- Speech lives in `esp32/components/muse/muse_chat_session.cpp`; credentials are generated by that component's `CMakeLists.txt`; the certificate fix is in `esp32/sdkconfig.defaults`. Preserve TLS verification and `CONFIG_MBEDTLS_CERTIFICATE_BUNDLE_CROSS_SIGNED_VERIFY=y`, including in the generated build configuration.
- Settings persist across flashes. Check Speak replies, volume, brightness, and sleep timeout when validating behavior; defaults do not overwrite existing NVS values.
- The local firmware binary was built on 4 Oct 2026 at 05:44, after the input source update. Build timestamps do not prove which binary is currently flashed. This README cleanup changes documentation only; it does not build, flash, or claim new hardware verification.
# muse-11labs-s3-175c
