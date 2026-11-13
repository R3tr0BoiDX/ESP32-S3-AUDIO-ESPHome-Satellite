ESPHome configuration for enabeling WAVESHARE-S3-AUDIO-BOARD (https://www.waveshare.com/esp32-s3-audio-board.htm)
to be used as a HomeAssistant Voice Satellite. Features such as simultanious music/and announcements and
continious on-board wake-word detection.

The device's DAC and ADC share pins for the i2s bus so to be able to configure two i2s_audio points for 
simultanious audio in/out I had to path the DAC (es8311) component, adding a setting to force it to become 
i2s master while still configuring the ESP and ADC to act as i2s slaves. I also had to fix MCLK/BLCK calculations.

example code features:
* complete Voice Assistant setup
* Onboard Wake-Word detection
* Led animations and event sounds
* working control buttons
* exposed announcement and music media_players
* built in alarm and timer

Steps to use the custom ES8311 component:
``` yaml
substitutions:
  i2c_id: internal_i2c
  i2s_mclk_multiple: 256
  i2s_bps_spk: 16bit
  i2s_bps_mic: 16bit
  i2s_sample_rate_spk: 16000
  i2s_sample_rate_mic: 16000 # must be 16000 for voice_assistant to work(?)
  i2s_use_apll: true
  
external_components:
  - source:
      type: git
      url: https://github.com/sw3Dan/waveshare-s2-audio_esphome_voice
      ref: main
    components: [ es8311 ]
    refresh: 0s

i2c:
  - id: $i2c_id
    sda: GPIO11
    scl: GPIO10
    scan: true

i2s_audio:
  - id: i2s_output
    i2s_lrclk_pin: 
      number: GPIO14
      allow_other_uses: true
    i2s_bclk_pin:  
      number: GPIO13
      allow_other_uses: true
    i2s_mclk_pin: # <-- for ESP to know what port to configure MCLK output
      number: GPIO12

  - id: i2s_input
    i2s_lrclk_pin:  
      number: GPIO14
      allow_other_uses: true
    i2s_bclk_pin:  
      number: GPIO13
      allow_other_uses: true

audio_dac:
  - platform: es8311
    id: es8311_dac
    i2c_id: $i2c_id
    use_mclk: true
    sample_rate: $i2s_sample_rate
    bits_per_sample: $i2s_bps_spk
    force_master: true # <-- New: to configure device as i2s master
    mclk_multiple: i2s_mclk_multiple # <-- New: for the i2s bus MCLK/BLCK calculations

audio_adc:
  - platform: es7210
    id: adc_mic
    i2c_id: $i2c_id
    sample_rate: $i2s_sample_rate
    bits_per_sample: $i2s_bps_mic

microphone:
  - platform: i2s_audio
    id: i2s_mics
    i2s_din_pin: GPIO15
    adc_type: external
    pdm: false
    i2s_audio_id: i2s_input
    i2s_mode: secondary # <-- so that ESP wont generate LRCLK/BLCK
    mclk_multiple: $i2s_mclk_multiple
    sample_rate: $i2s_sample_rate_mic # must be 16000 for VA (?)
    bits_per_sample: $i2s_bps_mic
    
speaker:
  - platform: i2s_audio
    id: i2s_audio_speaker
    i2s_dout_pin: GPIO16
    i2s_audio_id: i2s_output
    i2s_mode: secondary # component patch will force ES8311 to take command
    dac_type: external
    timeout: never
    buffer_duration: 100ms
    audio_dac: es8311_dac
    sample_rate: $i2s_sample_rate_spk
    bits_per_sample: $i2s_bps_spk
    use_apll: $i2s_use_apll # dont know if this is enforced when in 'secondary' i2s mode
    mclk_multiple: $i2s_mclk_multiple
    channel: stereo

  - platform: mixer
    id: mixing_speaker
    output_speaker: i2s_audio_speaker
    num_channels: 2
    source_speakers:

** Then create speaker: speaker/resamplers as needed ** 
```

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
| Wake word detected | Solid blue `#03A9F4` |
| Listening / replying | HA blue bands, slow rotation |
| Thinking | HA blue bands, fast rotation |
| Error | Red fast pulse |
| Alarm/timer ringing | Purple strobe |
| Timer running | Purple slow pulse |
| Volume change | Green bar, one LED per 1/7 of the volume |

## Time

The on-board PCF85063 RTC is read at boot and updated on every Home Assistant time sync, so alarms keep working without HA.

## Secrets

`secrets.yaml` (not in git) needs: `wifi_ssid`, `wifi_password`, `api_encryption_key`, `ota_password`, `device_name_web_username`, `device_name_web_password`.

## Lower room speakers during a voice request

The device ducks its own media. To also turn down other speakers in the room, snapshot them when the satellite leaves `idle` and restore them when it goes back to `idle`. Replace the entity ids with your own:

``` yaml
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

## TODO

* Filter speaker output from the mic input (echo cancellation with the IDF-SR AFE). Until then, ducking is the stopgap.
