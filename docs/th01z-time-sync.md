# TH01Z native time synchronization review

This review branch uses the built-in TH01Z definition from
[the converter fork PR](https://github.com/grespyrad/zigbee-herdsman-converters/pull/1).
`package.json` pins that fork to commit
`13d0b00296f2c4ce56c845e3cdc76debc3901628`; the lockfile pins its archive integrity.
The pnpm build permission is restricted to that exact GitHub archive, which needs
its existing `prepack` script to compile the TypeScript sources.

For `TS0601` / `_TZE200_locansqn`, the existing Nous SZ-T04 / Tuya TH01Z
modern extend automatically disables the ZCL default response for `mcuSyncTime`.
It applies to any matching device, with no IEEE address restriction, external
converter, or user option. Other manufacturer variants retain their current
response policy. UTC/local time payloads, the 1970/2000 epochs, the hourly update
interval, and transport-error logging are preserved.

This follows the [device contribution guide](https://www.zigbee2mqtt.io/advanced/support-new-devices/01_support_new_devices.html):
device behavior belongs in `zigbee-herdsman-converters`. Zigbee2MQTT itself only
needs this temporary dependency pin to exercise the change before a converter
release. Once the fix is accepted upstream and released, replace the pin with
the released version. Both review PRs target personal forks.

## Local development

The local checkouts are sibling directories under `Home Assistant`:
`zigbee2mqtt` and `zigbee-herdsman-converters`. Both have an `upstream` remote
pointing to Koenkk and an `origin` remote pointing to grespyrad. The Zigbee2MQTT
branch is based on `dev`; the converter branch is based on `master`.

Node.js 22 is supported by both current manifests. Use the pnpm version from
each repository's `packageManager` field; some version examples in upstream
`AGENTS.md` are older than those manifests.

From this repository:

```sh
npx --yes pnpm@11.26.0 install --frozen-lockfile
npx --yes pnpm@11.26.0 run build
npx --yes pnpm@11.26.0 run check
npx --yes pnpm@11.26.0 run test:coverage
```

The Git dependency is built during installation. This works from a standalone
clone and does not require the sibling converter checkout or `pnpm link`.
Editing the sibling converter checkout does not change this pinned dependency;
build/test/commit it, then update the pin and lockfile deliberately.

To develop the converter itself, run from its checkout:

```sh
npx --yes pnpm@12.3.4 install --frozen-lockfile
npx --yes pnpm@12.3.4 run build
npx --yes pnpm@12.3.4 run check
npx --yes pnpm@12.3.4 test
```

## Local startup

An isolated local onboarding server can be started from this repository:

```sh
ZIGBEE2MQTT_DATA="$PWD/data/th01z-local" \
Z2M_ONBOARD_URL=http://127.0.0.1:8082 \
npx --yes pnpm@11.26.0 start
```

Open <http://127.0.0.1:8082>. This uses ignored local data and listens on
loopback. It is a development startup check; a working Zigbee network requires
an MQTT broker and a dedicated coordinator. The running Home Assistant add-on
and its coordinator were not changed by these review branches.

## Validation and migration

The shared converter extension has regression checks for response policies,
UTC/local payloads and epochs, hourly forced updates, `time_start: off`,
gateway-status replies, and transport-error logging. These checks fail on the
original implementation and pass with the patch.

Local results: converter build and lint pass, with 987 tests in 30 files passing.
Zigbee2MQTT build and lint pass, with 846 tests in 25 files passing and 100%
statement, branch, function, and line coverage. The installed Git dependency
was also inspected directly: all eight SZ-T04 manufacturer fingerprints resolve
without external definitions, and only `_TZE200_locansqn` changes the response
policy for both requested and forced updates.

Earlier hardware validation used an equivalent external converter: after a
short button press, the display applied a deliberate +5 minute offset, then
returned to the correct time after the offset was removed. The native review
implementation is verified locally and has not been installed on that live
network. Successful Zigbee delivery alone does not prove display accuracy.

After installing a release containing the native fix, remove the old TH01Z
external converter so it cannot override the built-in definition. Wake the
clock with a short button press and verify its display and the logs. The patch
does not fix sleeping-device delivery failures or radio connectivity. To revert
this review setup, check out `dev` and reinstall with its frozen lockfile.
