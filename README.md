# Zephyr Upgrade via MCUMgr

## Setup

```bash
uv sync
```

```bash
# need to do only once at the time of repo setup
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

> Note even though the doc/cmdline suggest that path to key files can be relative, it does not work; safer to provide full path!

```bash
uv run west build -b esp32_devkitc/esp32/procpu -p always example-app --sysbuild -- '-DSB_CONFIG_BOOT_SIGNATURE_KEY_FILE="/Users/kapil.sachdeva/Desktop/Dev/myoss/zephyr-upgrade-mcumgr/assets/bootloader-key.pem"' '-DCONFIG_MCUBOOT_SIGNATURE_KEY_FILE="/Users/kapil.sachdeva/Desktop/Dev/myoss/zephyr-upgrade-mcumgr/assets/app-key.pem"' '-DEXTRA_CONF_FILE=transport-serial.conf'
```

```bash
# use this command to make sure that config from
# transport-serial.conf made into the final config
grep CONFIG_BOOT_SIGNATURE_KEY_FILE build/mcuboot/zephyr/.config
grep CONFIG_MCUMGR_TRANSPORT_UART build/example-app/zephyr/.config
grep CONFIG_MCUBOOT_SIGNATURE_KEY_FILE build/example-app/zephyr/.config
```

**See the signature dump**

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
uv run west build -b esp32_devkitc/esp32/procpu -p always example-app --sysbuild -- '-DSB_CONFIG_BOOT_SIGNATURE_KEY_FILE="/Users/kapil.sachdeva/Desktop/Dev/myoss/zephyr-upgrade-mcumgr/assets/bootloader-key.pem"' '-DEXTRA_CONF_FILE=transport-serial.conf' '-DCONFIG_MCUBOOT_IMGTOOL_SIGN_VERSION="0.2.0+3"' '-DCONFIG_MCUBOOT_SIGNATURE_KEY_FILE="/Users/kapil.sachdeva/Desktop/Dev/myoss/zephyr-upgrade-mcumgr/assets/app-key.pem"'
```

```bash
# verify
grep CONFIG_MCUBOOT_IMGTOOL_SIGN_VERSION build/example-app/zephyr/.config
grep CONFIG_MCUMGR_TRANSPORT_UART build/example-app/zephyr/.config
```

### Upgrade using `smpmgr`

```bash
# Let's read the state of the image
# You should only see slot0
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
# upload the new image
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
# Note import to put "" around the hash (learned the hard way :())
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

## Important Config items


```bash
# Defined in prj.conf

# Direct dependencies of MCUMGR
CONFIG_NET_BUF=y
CONFIG_ZCBOR=y

CONFIG_MCUMGR=y

# This enables image management commands
CONFIG_FLASH=y
CONFIG_FLASH_MAP=y
CONFIG_STREAM_FLASH=y
CONFIG_IMG_MANAGER=y
CONFIG_MCUMGR_GRP_IMG=y

# we need to turn this ON otherwise smpmgr fails on state-write
# confirmation
CONFIG_MCUMGR_GRP_OS=y

# Bootloader specific settings
CONFIG_BOOTLOADER_MCUBOOT=y
CONFIG_MCUBOOT_IMGTOOL_SIGN_VERSION="0.1.0+2"
```

```bash
# Defined in transport-serial.conf

# Deps of CONFIG_MCUMGR_TRANSPORT_UART
CONFIG_CONSOLE=y
CONFIG_BASE64=y
CONFIG_CRC=y

CONFIG_MCUMGR_TRANSPORT_UART=y
```
