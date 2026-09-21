# Contributing

This is an experimental reproduction and validation repository, not an official PipeWire fork.

When opening an issue, include the validation checklist from `docs/validation.md`. Do not post full Bluetooth addresses, private message content or unrestricted journal logs. Redact addresses and unrelated audio/application data.

Patches should remain narrow and explain which transport state they protect. Please test both the first HFP acquisition and a subsequent acquisition after returning to A2DP.

Before proposing an upstream change, compare the patch with the current PipeWire source and test against the exact release being reported.
