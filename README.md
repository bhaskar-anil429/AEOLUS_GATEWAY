# AEOLUS gateway firmware

Release channel for the NCD AEOLUS LTE gateway (nRF54L15 + Quectel EG916). The
gateway collects recordings from PLP AEOLUS strain sensors over Bluetooth and
uploads them to PLP Insight (HTTPS) or MQTT.

| File | What it is |
|---|---|
| `aeolus/latest.json` | Version file the gateways check for updates |
| `AEOLUS_V*.bin` | Signed firmware update images (MCUboot) |
| `AEOLUS_CONFIG_TOOL.html` | Config page: open in Chrome or Edge and connect to the gateway over Bluetooth |

## Automatic updates

Each gateway downloads `aeolus/latest.json` on its first cycle after a restart,
then every 24 hours by default:

```json
{ "version": "V1.73", "url": "https://raw.githubusercontent.com/bhaskar-anil429/AEOLUS_GATEWAY/main/AEOLUS_V1.73.bin" }
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
{"cmd":"ota_ready","url":"https://raw.githubusercontent.com/bhaskar-anil429/AEOLUS_GATEWAY/main/AEOLUS_V1.73.bin"}
```

The config page has the same settings under **Firmware updates (LTE, automatic)**.

## Images here contain no credentials

Update images are built with `CONFIG_AEOLUS_EMBED_SECRETS=n`: the Insight token
and MQTT user/password are left out. A gateway keeps those in its own settings.
They are saved there the first time it boots a factory (J-Link) image, or set
through the config page or MQTT. **Never upload an image built with the
secrets embedded.**

## Releasing a new version

1. Bump `AEOLUS_FW_VERSION` in `src/devconfig.c`.
2. Build the secret-free image (`-DCONFIG_DEVICE_TYPE_AEOLUS=y
   -DEXTRA_CONF_FILE=aeolus.conf -DCONFIG_AEOLUS_EMBED_SECRETS=n`, sysbuild with
   MCUboot). Take `zephyr/zephyr.signed.bin`.
3. Add it here as `AEOLUS_V<x.yy>.bin`.
4. Update `aeolus/latest.json` (`version` and `url`) in the same commit.
