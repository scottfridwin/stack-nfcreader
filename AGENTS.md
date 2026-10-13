# Notes for agents and maintainers

This repo is a Komodo stack deployed by the shared scripts in
[stack-komodo](https://github.com/scottfridwin/stack-komodo) (see its README and AGENTS.md).
Its settings are the `[[stack]]` entry for `nfcreader` in
[komodo-resources](https://github.com/scottfridwin/komodo-resources) `fridwin/stacks.toml`.
It runs on the kiosk server `kiosk-diningroom`.

- Files in `config/`, `work/`, `data/` are copied over the live copies on every deploy and
  rendered with envsubst: any `$VAR` must be set or the deploy fails; use `$$VAR` for a literal.
- Merging a change to `compose.yaml` redeploys automatically; other changes need a manual Deploy.
- Don't commit files the app rewrites at runtime; use `name.seed` for create-if-missing files.
- Check the live state read-only (`docker inspect`, `docker logs`); never print secrets.

## This stack
- It is not on fridwin: `docker inspect` / `docker logs` there will not find `nfc-mqtt-bridge`. Use Komodo
  (server `kiosk-diningroom`) or SSH to the kiosk as `scott`; host setup is in stack-komodo `nodes/kiosk/`.
- The image major version must match the kiosk's `pcscd` generation: 2.x for `pcscd` 2.x (Raspberry Pi OS trixie,
  the current kiosk), 1.x for `pcscd` 1.9 (bullseye/bookworm). Do not merge a Renovate PR that changes the major
  without checking the kiosk OS; a mismatch fails with *Service was stopped (0x8010001E)*.
- Reader access depends on the host polkit rule for group `fridwin_services`, so keep the container's group
  `GID_SERVICES` and keep `service_nfcreader` (UID 1102, created by `nodes/kiosk/bootstrap-kiosk.sh` from
  `SERVICE_USERS` in `nodes/kiosk/hosts/kiosk-diningroom.env`) in that group.
- Changing `DEVICE_ID` creates a new device in Home Assistant and orphans the old one's entities; tag automations
  trigger on tag IDs, not on the reader.
- The bridge talks to Home Assistant only over MQTT (no REST token); the MQTT user name comes from the
  `HOMEASSISTANT_MQTT_USERNAME` Komodo variable and its password from the `nfcmqttbridge_mqtt_password` secret.
