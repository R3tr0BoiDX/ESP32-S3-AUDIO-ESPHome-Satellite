# Waveshare ESP32-S3 Audio Board as a Home Assistant voice satellite

ESPHome configuration that turns the [Waveshare ESP32-S3 Audio Board](https://www.waveshare.com/esp32-s3-audio-board.htm) into a Home Assistant voice satellite.

Features:
- Full voice assistant pipeline with on-device wake word detection (microWakeWord)
- Music and announcements at the same time, with ducking
- LED ring feedback and event sounds
- Hardware buttons for volume, play/pause and mute
- Built-in timers and alarm, kept running by the on-board RTC

## Install

1. Create `secrets.yaml` next to the config (it is gitignored):
   ```yaml
   wifi_ssid: ...
   wifi_password: ...
   api_encryption_key: ...   # 32 bytes, base64
   ota_password: ...
   device_name_web_username: ...
   device_name_web_password: ...
   ```
2. Flash: `esphome run waveshare-s3-audio.yaml`
3. Add the device in Home Assistant (it is discovered automatically).

## Controls

| Key | Click | Hold / double click |
|-----|-------|---------------------|
| 1 | Volume down | Hold: repeat every 200 ms |
| 2 | Stop ringing alarm/timer, else play/pause | Double click: toggle `Mute` |
| 3 | Volume up | Hold: repeat every 200 ms |

`Mute` stops wake word detection and the voice assistant. `LED ring enabled` turns all LED feedback off.

## LED ring

| State | LEDs |
|-------|------|
| Booting | Rainbow |
| No Wi-Fi / HA connection | Red pulse |
| Wake word detected | Solid blue |
| Listening / replying | Fades through blues, slow |
| Thinking | Fades through blues, fast |
| Error | Red fast pulse |
| Alarm/timer ringing | Purple strobe |
| Timer running | Purple slow pulse |
| Volume change | Green bar, one LED per 1/7 of the volume |

## Time

The on-board PCF85063 RTC is read at boot and updated on every Home Assistant time sync, so alarms keep working without HA.

Set the alarm in HA with `Alarm time` (`HH:MM`), `Alarm` (on/off) and `Alarm action`.

## Lower room speakers during a voice request

The device ducks its own media. To also turn down other speakers in the room, snapshot them when the satellite leaves `idle` and restore them when it returns to `idle`. Replace the entity IDs with your own:

```yaml
automation:
  - alias: Duck room speakers during voice requests
    mode: queued
    triggers:
      - trigger: state
        entity_id: assist_satellite.waveshare_audio_3_assist_satellite
        from: idle
        not_to: [unavailable, unknown]
        id: active
      - trigger: state
        entity_id: assist_satellite.waveshare_audio_3_assist_satellite
        to: idle
        not_from: [unavailable, unknown]
        id: idle
    actions:
      - choose:
          - conditions:
              - condition: trigger
                id: active
            sequence:
              - action: scene.create
                data:
                  scene_id: before_voice_request
                  snapshot_entities:
                    - media_player.living_room
              - action: media_player.volume_set
                target:
                  entity_id: media_player.living_room
                data:
                  volume_level: 0.1
          - conditions:
              - condition: trigger
                id: idle
            sequence:
              - action: scene.turn_on
                target:
                  entity_id: scene.before_voice_request
```

## Patched ES8311 component

The DAC (ES8311) and ADC (ES7210) share the I2S clock pins. To run two `i2s_audio` buses for simultaneous playback and recording, the bundled `es8311` component adds `force_master`, which makes the ES8311 the I2S master while the ESP32 and the ADC run as secondaries. It also fixes the MCLK/BCLK calculation (`mclk_multiple`).

To use it in your own config:

```yaml
substitutions:
  i2s_sample_rate: 16000 # the voice assistant needs 16 kHz on the mic
  i2s_bps: 16bit
  i2s_mclk_multiple: 256

esphome:
  on_boot:
    - priority: -100
      then:
        # The ES8311 derives BCLK/LRCK from MCLK, which the ESP32 only outputs while
        # the speaker port runs. Start it silently (timeout: never keeps it on),
        # otherwise the mic gets no clock until the first sound plays.
        - lambda: |-
            id(i2s_audio_speaker)->set_audio_stream_info(audio::AudioStreamInfo(16, 2, ${i2s_sample_rate}));
            id(i2s_audio_speaker)->start();

external_components:
  - source:
      type: git
      url: https://github.com/sw3Dan/waveshare-s2-audio_esphome_voice
      ref: main
    components: [es8311]
    refresh: 0s

i2c:
  - id: internal_i2c
    sda: GPIO11
    scl: GPIO10

i2s_audio:
  - id: i2s_output
    i2s_lrclk_pin: { number: GPIO14, allow_other_uses: true }
    i2s_bclk_pin: { number: GPIO13, allow_other_uses: true }
    i2s_mclk_pin: GPIO12 # tells the ESP32 which pin carries MCLK
  - id: i2s_input
    i2s_lrclk_pin: { number: GPIO14, allow_other_uses: true }
    i2s_bclk_pin: { number: GPIO13, allow_other_uses: true }

audio_dac:
  - platform: es8311
    id: es8311_dac
    i2c_id: internal_i2c
    use_mclk: true
    sample_rate: $i2s_sample_rate
    bits_per_sample: $i2s_bps
    force_master: true # new: ES8311 drives the I2S clocks
    mclk_multiple: $i2s_mclk_multiple # new: used for the MCLK/BCLK calculation

audio_adc:
  - platform: es7210
    id: adc_mic
    i2c_id: internal_i2c
    sample_rate: $i2s_sample_rate
    bits_per_sample: $i2s_bps

microphone:
  - platform: i2s_audio
    id: i2s_mics
    i2s_din_pin: GPIO15
    adc_type: external
    pdm: false
    i2s_audio_id: i2s_input
    i2s_mode: secondary # the ESP32 must not drive LRCLK/BCLK
    sample_rate: $i2s_sample_rate
    bits_per_sample: $i2s_bps
    mclk_multiple: $i2s_mclk_multiple

speaker:
  - platform: i2s_audio
    id: i2s_audio_speaker
    i2s_dout_pin: GPIO16
    i2s_audio_id: i2s_output
    i2s_mode: secondary # the ES8311 is master
    dac_type: external
    audio_dac: es8311_dac
    timeout: never
    buffer_duration: 100ms
    sample_rate: $i2s_sample_rate
    bits_per_sample: $i2s_bps
    mclk_multiple: $i2s_mclk_multiple
    channel: stereo
  # add mixer/resampler speakers on top as needed,
  # see waveshare-s3-audio.yaml
```
