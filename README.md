# Zephyr Upgrade via MCUMgr

## Setup

```bash
uv sync
```

```bash
# needed only once
uv run pre-commit install
uv run west update
uv run west sdk install
```

## Generate key pair

```bash
uv run poe key-gen
```

This would create RSA 2048 key pairs for MCUBoot & App Firmware

See `poe.toml` for exact commands

## Multi stage build using `sysbuild` [Version 0.1.0+2]

```bash
uv run west build -b esp32_devkitc/esp32/procpu -p always example-app --sysbuild -- -DEXTRA_CONF_FILE="transport-serial.conf"
```

> I have found passing arguments to cmake to be flaky; if more than one it seems to ignore

```bash
# use this command to make sure that config from transport-serial.conf made into the final config
grep CONFIG_MCUMGR_TRANSPORT_UART build/example-app/zephyr/.config
```

**See the signature dump***

```bash
uv run poe sign-dump
```

### Flash & Monitor

```bash
uv run west flash
```

```bash
uv run west espressif monitor
```

## Build a new image with version 0.2.0+3

```bash
uv run west build -b esp32_devkitc/esp32/procpu -p always example-app --sysbuild -- '-DEXTRA_CONF_FILE=transport-serial.conf' '-DCONFIG_MCUBOOT_IMGTOOL_SIGN_VERSION="0.2.0+3"'
```

> Passing cmake extra args is flaky, a good to verify if our intended configs made it or not

```bash
# verify
grep CONFIG_MCUBOOT_IMGTOOL_SIGN_VERSION build/example-app/zephyr/.config
grep CONFIG_MCUMGR_TRANSPORT_UART build/example-app/zephyr/.config
```

### Upgrade using `smpmgr`

```bash
# Let's read the state of the image
# You should only one slot0
uv run smpmgr --port /dev/tty.usbserial-0001 --baudrate 115200 image state-read
```

```
# example output
ImageState(
    slot=0,
    version='0.1.0.2',
    image=None,
    hash=HashBytes('3037020FA153516370291611B7EB03D4E268151D9D43736CEDEE4DAD9DA79050'),
    bootable=True,
    pending=False,
    confirmed=True,
    active=True,
    permanent=False
)
splitStatus: 0
```

```bash
# upload the new image from upgrade-build directory
uv run smpmgr --port /dev/tty.usbserial-0001 --baudrate 115200 upgrade build/example-app/zephyr/zephyr.signed.bin
```


```bash
# Let's read the state of the image
# Now we should have slot0 and slot1
uv run smpmgr --port /dev/tty.usbserial-0001 --baudrate 115200 image state-read
```

```
# example output
ImageState(
    slot=0,
    version='0.1.0.2',
    image=None,
    hash=HashBytes('3037020FA153516370291611B7EB03D4E268151D9D43736CEDEE4DAD9DA79050'),
    bootable=True,
    pending=False,
    confirmed=True,
    active=True,
    permanent=False
)
ImageState(
    slot=1,
    version='0.2.0.3',
    image=None,
    hash=HashBytes('62828ECB3E52387A7D54C2733413C8438B3B0F9FAA67B991819E5341055333F9'),
    bootable=True,
    pending=False,
    confirmed=False,
    active=False,
    permanent=False
)
splitStatus: 0
```

Now we make the image slot1 active

```bash
# Change the hash to what you see in slot1
uv run smpmgr --port /dev/tty.usbserial-0001 --baudrate 115200 image state-write "62828ECB3E52387A7D54C2733413C8438B3B0F9FAA67B991819E5341055333F9"
```

```bash
# use monitor to see the new version
uv run west espressif monitor
```

```bash
# or state-read to see that new image must be active
uv run smpmgr --port /dev/tty.usbserial-0001 --baudrate 115200 image state-read
```
