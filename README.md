# stack-nfcreader

Runs [nfc-mqtt-bridge](https://github.com/scottfridwin/docker-nfc-mqtt-bridge) on the dining room kiosk
(`kiosk-diningroom`), not on fridwin. It reads tags from the kiosk's USB NFC reader (ACS ACR1552) through the host's
`pcscd` and publishes them to Home Assistant's MQTT broker, with discovery, so the reader appears in Home Assistant
as a tag scanner (**Settings > Tags**) named "Dining Room NFC Reader".

## Services

| Container | Image | Role | Runs as | Networks | Reached at |
|---|---|---|---|---|---|
| `nfc-mqtt-bridge` (service `nfcmqttbridge`) | `ghcr.io/scottfridwin/nfc-mqtt-bridge` (2.x) | Polls the reader over PC/SC, publishes scans to MQTT | `UID_NFCREADER` (1102) : `GID_SERVICES` (1001) | `public` (egress) | no UI or port; outbound only |

- MQTT: broker at `${MQTT_HOST}:${MQTT_PORT}` = `LOCALIP_HOME_ASSISTANT`:`HOMEASSISTANT_MQTT_PORT` (1883), user
  `HOMEASSISTANT_MQTT_USERNAME`, without TLS. `DEVICE_ID=nfc_reader_diningroom`, `DEVICE_NAME="Dining Room NFC Reader"`.
- Topics (`<id>` = `nfc_reader_diningroom`): `homeassistant/event/<id>/tag_scanned`, `homeassistant/sensor/<id>/uid/state`,
  `homeassistant/sensor/<id>/availability` (retained `online` / `offline`), plus the discovery configs.
- The bridge talks to the reader only through the host's `pcscd` socket (`/run/pcscd`); it gets no USB device.
- Logs use the kiosk's Docker default (`local` driver, capped), not fridwin's syslog.

## Deploy settings

Deployed by the shared scripts, see [stack-komodo](https://github.com/scottfridwin/stack-komodo#stack-deploy-scripts);
on the kiosk they come from the `deploy-scripts-kiosk-diningroom` clone of stack-komodo.
Komodo `environment` (in `komodo-resources/fridwin/stacks.toml`, `server = "kiosk-diningroom"`):

| Setting | Value / effect |
|---|---|
| `SERVICE_UID` / `SERVICE_GID` / `SERVICE_NAME` | `UID_NFCREADER` / `GID_ADMINS` / `nfcreader` |
| `SHARED_NETWORKS` | `public` (created on the kiosk if missing) |
| `MQTT_HOST` / `MQTT_PORT` / `MQTT_USERNAME` | `[[LOCALIP_HOME_ASSISTANT]]` / `[[HOMEASSISTANT_MQTT_PORT]]` / `[[HOMEASSISTANT_MQTT_USERNAME]]` |
| `DEVICE_NAME` / `DEVICE_ID` | `Dining Room NFC Reader` / `nfc_reader_diningroom` (Home Assistant device and entity IDs derive from `DEVICE_ID`) |

No Komodo tags. `auto_update = true`, but the image tag is pinned, so new versions arrive as Renovate PRs that are
merged by hand (no automerge).

## Kiosk prerequisites

Set up by the kiosk bootstrap in [stack-komodo `nodes/kiosk/`](https://github.com/scottfridwin/stack-komodo/blob/main/nodes/kiosk/README.md),
from `nodes/kiosk/hosts/kiosk-diningroom.env`:

- **Service user:** `SERVICE_USERS` contains `nfcreader:1102`, so the bootstrap creates `service_nfcreader` (UID 1102,
  primary group `fridwin_services`). There is no other account script for kiosk stacks.
- **pcscd:** `NFC_READER=yes` installs `pcscd` and `libccid`, enables `pcscd.socket`, and adds the polkit rule
  `/etc/polkit-1/rules.d/50-pcscd-fridwin-services.rules`, which lets members of `fridwin_services` use the reader.
  pcscd 2.x checks polkit, and the container's user has no login session, so without this rule the bridge gets
  *Access denied*.
- **Matching PC/SC generation:** the container's PC/SC client and the host `pcscd` must be the same generation.
  The kiosk runs Raspberry Pi OS trixie (`pcscd` 2.x), which needs bridge **2.x**; bridge 1.x only works with
  `pcscd` 1.9 (bullseye/bookworm). A mismatch logs *Service was stopped (0x8010001E)*.

## Storage

The stack mounts no `/config`, `/work` or `/data` folders (the deploy scripts still create empty
`/config/nfcreader`, `/work/nfcreader`, `/data/nfcreader`). Nothing needs backing up; tags and automations live in
Home Assistant.

| Path | Contents |
|---|---|
| `/run/pcscd` (host) | The host `pcscd` socket folder, mounted at `/run/pcscd`. |
| `/tmp` (tmpfs) | Heartbeat file for the container health check. |

## Secrets

| File in `/config/secrets` (on the kiosk) | Use |
|---|---|
| `nfcmqttbridge_mqtt_password` | Password of the Home Assistant MQTT user (Compose secret, read via `MQTT_PASSWORD_FILE`). Owned by `service_nfcreader`, mode `0600`. Restored by hand when the kiosk is rebuilt (see the kiosk README). |

## Security

Non-root (`UID_NFCREADER`), `cap_drop: ALL`, `no-new-privileges`, read-only root filesystem with a tmpfs `/tmp`.
It has no device access; reader access is granted by the host's polkit rule for `fridwin_services`.
MQTT is unencrypted (no `MQTT_TLS`), so the password and tag UIDs cross the LAN in clear text.

## Operations / Notes

- **Verify:** scan a tag; it appears in Home Assistant **Settings > Tags**, and the device's **Tag UID** sensor shows
  it while the tag is on the reader. On the kiosk, `pcsc_scan` (package `pcsc-tools`, not installed by the bootstrap)
  checks the reader without the bridge.
- **Unhealthy container:** the bridge has not finished a reader poll for 30 seconds, usually because it is waiting
  for `pcscd` (*Waiting for pcscd socket*). Check `systemctl status pcscd.socket` and `journalctl -u pcscd` on the kiosk.
- **Rollback to an old (bullseye/bookworm) kiosk card:** pin the image back to the last 1.x tag so it matches
  `pcscd` 1.9.
- Troubleshooting table: [bridge README](https://github.com/scottfridwin/docker-nfc-mqtt-bridge#troubleshooting).
