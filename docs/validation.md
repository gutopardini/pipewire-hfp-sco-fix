# Validation checklist

Use this checklist before reporting a device as working.

## Record the environment

```bash
uname -r
pipewire --version
wireplumber --version
bluetoothctl info <MAC>
pactl list cards
```

Record the headset model, Fedora/distribution version, Bluetooth adapter and active PipeWire profile.

## Automatic profile switch

Start with the headset in A2DP, then capture from the Bluetooth source:

```bash
pactl set-card-profile bluez_card.<MAC_UNDERSCORED> a2dp-sink
timeout 12s pw-record \
  --target bluez_input.<MAC_COLON> \
  --rate 16000 --channels 1 --format s16 \
  /tmp/hfp-sco-fix-test.wav
```

The exact card/source names can be obtained with `pactl list cards` and `pactl list short sources`.

Inspect the capture and logs:

```bash
ffmpeg -hide_banner -i /tmp/hfp-sco-fix-test.wav \
  -af volumedetect -f null - 2>&1 | grep -E 'mean_volume|max_volume'

journalctl --user -u wireplumber.service --since '2 minutes ago' --no-pager \
  | grep -Ei 'sco|transport|BCS|BCC|decode|Too many|Failure in Bluetooth'
```

Expected signals include `reusing pending transport`, `sco_ready`, `acquire complete` and `decode_data`. The failure signature is `Too many links (-31)` or `SCO socket error`.

## Consecutive captures

Run at least two captures without restarting PipeWire. The second capture is important: it checks that an already active transport is not incorrectly treated as a pending connection.

## Browser test

Fully exit and reopen the browser after restarting PipeWire. On WhatsApp Web, start a voice message, speak for several seconds, stop and send it. Repeat once. Confirm that the profile switches to HFP and returns to A2DP after capture ends.

## Report template

Include:

- headset model and Bluetooth address with the address partially redacted;
- distribution, kernel, PipeWire, WirePlumber and BlueZ versions;
- Bluetooth adapter model;
- active profiles before/during/after capture;
- whether the first and second captures contain audio;
- the relevant log lines, especially `Too many links`, `SCO socket error`, `sco_ready` and `decode_data`.
