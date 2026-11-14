# AEOLUS gateway firmware

Release channel for the NCD AEOLUS LTE gateway (nRF54L15 + Quectel EG916). The
gateway collects recordings from PLP AEOLUS strain sensors over Bluetooth and
uploads them to PLP Insight (HTTPS) or MQTT.

| File | What it is |
|---|---|
| `aeolus/latest.json` | Version file the gateways check for updates |
| `AEOLUS_V*.bin` | Signed firmware update images (MCUboot) |
| `AEOLUS_CONFIG_TOOL.html` | Config page: open in Chrome or Edge and connect to the gateway over Bluetooth |
| `sensors/*.ncd` | PLP strain sensor updates (the same files the PLP phone app sends) |

## Automatic updates

Each gateway downloads `aeolus/latest.json` on its first cycle after a restart,
then every 24 hours by default:

```json
{ "version": "V1.77", "url": "https://raw.githubusercontent.com/bhaskar-anil429/AEOLUS_GATEWAY/main/AEOLUS_V1.77.bin" }
```

If `version` is newer than the running firmware, the gateway:

1. downloads `url` over LTE into the spare flash slot,
2. restarts into it on a trial basis,
3. keeps it once it reaches the cloud (MQTT connects, Insight accepts an upload,
   or the version file loads).

If the new image doesn't reach the cloud in 3 cycles, the gateway restarts and
the bootloader goes back to the previous firmware. Each image URL gets at most 3
download attempts. The bootloader only installs images signed with the NCD key.

### Changing the update settings

MQTT, on `aeolus/<gateway-id>/cmd`:

```json
{"cmd":"auto_ota","val":1,"days":1,"url":"https://.../latest.json"}
```

- `val`: 1 = on, 0 = off
- `days`: check interval, 1 to 365
- `url`: version file, `""` = the firmware default above
- every field is optional

Install one specific image now:

```json
{"cmd":"ota_ready","url":"https://raw.githubusercontent.com/bhaskar-anil429/AEOLUS_GATEWAY/main/AEOLUS_V1.77.bin"}
```

The config page has the same settings under **Firmware updates (LTE, automatic)**.

## Sensor firmware updates (gateway V1.75 and later)

The gateway can update the PLP strain sensors it reads. It downloads a `.ncd`
file over LTE, keeps it in its spare firmware slot, and sends it over Bluetooth
to each sensor after reading that sensor's recordings. The sensor restarts and
its bootloader checks the file and installs it; a bad file is rejected and the
old firmware stays. On the next visit the gateway reads the sensor's version and
reports the result.

Send a file to the sensors, over MQTT on `aeolus/<gateway-id>/cmd`:

```json
{"cmd":"sensor_fw","url":"https://raw.githubusercontent.com/bhaskar-anil429/AEOLUS_GATEWAY/main/sensors/PLP_SG_V10.ncd","mac":["EB:0E:AD:F9:55:6F","FC:FB:07:9E:63:ED"]}
```

- `url`: any HTTPS location of a `.ncd` file, so the files can move to another
  host without a firmware change
- `mac`: optional list of up to 16 sensors (a single MAC string also works);
  leave it out to send the file to every sensor of the matching type and
  hardware. The listed sensors are updated one by one as the gateway visits them.
- the file is downloaded **once** and kept on the gateway (also across
  restarts); every sensor gets it from there, so 10 sensors cost one download
- a file from this command goes to sensors on any other version, so an older
  file rolls a sensor back
- `{"cmd":"sensor_fw","cancel":1}` stops sending and forgets the file
- `{"cmd":"sensor_fw","auto":0}` / `1` ignores / follows the version file's
  `sensor` entry

Automatic: add a `sensor` entry to `aeolus/latest.json`. Gateways then send it
to sensors on an **older** version:

```json
{ "version": "V1.77", "url": "https://.../AEOLUS_V1.77.bin",
  "sensor": { "version": 10, "url": "https://.../sensors/PLP_SG_V10.ncd" } }
```

The location of `latest.json` itself is set with
`{"cmd":"auto_ota","url":"https://.../latest.json"}`, so both the gateway and the
sensor updates follow it if the files move.

Results come back on `aeolus/<gateway-id>/<SENSOR-MAC>/ack` with `"param":"firmware"`
and `status` `sent`, `installed`, `failed`, `transfer_failed`, `refused`,
`no_update_service` or `wrong_sensor_type`, and on `aeolus/<gateway-id>/ack` for
the download (`staged`, `download_failed`, `invalid_file`). A sensor gets 3 tries
per file. A sensor whose type or hardware ID doesn't match the file is never sent it.

| File | Sensor firmware |
|---|---|
| `sensors/PLP_SG_V10.ncd` | PLP strain sensor FW 10 (type 0x6B, hardware 50-58-0C) |
| `sensors/PLP_SG_V11_TEST.ncd` | Test only: FW 10 code labelled version 11, to check the update path |

## Images here contain no credentials

Update images are built with `CONFIG_AEOLUS_EMBED_SECRETS=n`: the Insight token
and MQTT user/password are left out. A gateway keeps those in its own settings.
They are saved there the first time it boots a factory (J-Link) image, or set
through the config page or MQTT. **Never upload an image built with the
secrets embedded.**

## Releasing a new version

1. Bump `FW_VER` in `CMakeLists.txt`.
2. Build the secret-free image with `build_aeolus_ota.sh` (sysbuild with MCUboot,
   `CONFIG_AEOLUS_EMBED_SECRETS=n`). It fails if the token or MQTT password is
   in the image and writes `build_gw_ota/AEOLUS_V<x.yy>.bin`.
3. Add it here as `AEOLUS_V<x.yy>.bin`.
4. Update `aeolus/latest.json` (`version` and `url`) in the same commit.
