# Vectrx2020 ESP32 firmware: build and flash

This directory is the main ESP32 firmware source project for Vectrx2020
(`network_adapter`). It also uses shared code from `../common`. The STM32 firmware
is a separate project in `Vectrx2020_Slave`. The commands below describe the
Windows workspace used for testing:

- ESP-IDF: `D:\Espressif\frameworks\esp-idf-v5.2.6`
- Target: classic `esp32` (not ESP32-S3 or ESP32-C3).
- Serial port: `COM10` in this setup; substitute your adapter's actual port.
- Programming baud: `460800`.
- Normal configuration: this project's `sdkconfig`, selecting SDIO with packet
  statistics disabled at the time these instructions were checked.
- Build artifacts are kept under
  `Vectrx2020_Slave\build\diagnostics\esp32-forwarding\build`.

These commands build the current source, including the `ESP_FORWARD` diagnostic
instrumentation when enabled by the selected configuration. They do not restore
the separately saved original ESP32 application.

## Build in PowerShell

```powershell
$env:IDF_TOOLS_PATH = 'D:\Espressif'
$env:IDF_PYTHON_ENV_PATH = 'D:\Espressif\python_env\idf5.2_py3.11_env'
$env:PATH = "$env:IDF_PYTHON_ENV_PATH\Scripts;$env:PATH"

. 'D:\Espressif\frameworks\esp-idf-v5.2.6\export.ps1'

python --version
idf.py --version

Set-Location 'D:\source\vectrx2020_framework\esp-hosted\slave'

$espBuild = 'D:\source\vectrx2020_framework\Vectrx2020_Slave\build\diagnostics\esp32-forwarding\build'
$espConfig = 'D:\source\vectrx2020_framework\esp-hosted\slave\sdkconfig'

idf.py -B $espBuild "-DSDKCONFIG=$espConfig" reconfigure
idf.py -B $espBuild "-DSDKCONFIG=$espConfig" build
```

Use a fresh PowerShell window. The environment settings above apply only to that
session and select the existing installation under `D:\Espressif`. Check that
the version commands report Python **3.11.x** and ESP-IDF **5.2.6** before
building. `export.ps1` invokes `python` from `PATH`; an inherited
`IDF_TOOLS_PATH=C:\Espressif` or a system Python 3.13 can make it look for the
wrong tools and Python environment. If it reports that installed tools are
missing, check these paths before running any installation commands.

The application output is `$espBuild\network_adapter.bin`. These commands use
the main project's `slave\sdkconfig` explicitly, including when reusing an
existing build directory. They build current source, not an archived firmware
snapshot.

## Configuration selection and diagnostics

The word `diagnostics` in the output-directory path does **not** enable
diagnostics. ESP32 packet counters are controlled by `CONFIG_ESP_PKT_STATS` in
the selected configuration. STM32 network diagnostics are separate and do not
select the ESP32 transport.

An optional local configuration used during the investigation is:

```text
D:\source\vectrx2020_framework\Vectrx2020_Slave\build\diagnostics\esp32-forwarding\sdkconfig
```

That copy selects SDIO with packet statistics enabled. To use it intentionally,
set `$espConfig` to that file and rerun both `reconfigure` and `build` with the
explicit `-DSDKCONFIG` argument. The local copy is under an ignored build
directory and may not exist in another checkout.

If errors mention missing `CONFIG_ESP_SDIO_RX_Q_SIZE`, invalid SDIO timing/speed,
or a missing `sdio_buf_handle`, check for an unintended SPI configuration:

```powershell
Select-String -Path "$espBuild\CMakeCache.txt" -Pattern '^SDKCONFIG:'
Select-String -Path "$espBuild\config\sdkconfig.h" `
    -Pattern 'CONFIG_ESP_SDIO_HOST_INTERFACE|CONFIG_ESP_SPI_HOST_INTERFACE'
```

For this hardware the generated header must select
`CONFIG_ESP_SDIO_HOST_INTERFACE`, not SPI. One failed build cached
`Vectrx2020_Slave\build\sdkconfig` instead of the intended file. The current
component list also includes `sdio_slave_api.c` unconditionally, so selecting
SPI produces SDIO compile errors. Use the explicit configuration/reconfigure
commands above rather than defining the missing macros manually.

## Check physical flash capacity

### Replacement module on COM14 (4 MB)

`sdkconfig.esp32-com14-4mb` is a separate configuration for the tested
ESP32-D0WD-V3 replacement (MAC `48:e7:29:8a:0f:28`). It differs from the main
configuration only in selecting 4 MB instead of 8 MB flash; SDIO and its timing
settings remain the same, with packet statistics disabled. COM numbers can
change when reconnecting devices.

After initializing the ESP-IDF environment above, use:

```powershell
$espBuild = 'D:\source\vectrx2020_framework\Vectrx2020_Slave\build\diagnostics\esp32-com14-4mb\build'
$espConfig = 'D:\source\vectrx2020_framework\esp-hosted\slave\sdkconfig.esp32-com14-4mb'

idf.py -B $espBuild "-DSDKCONFIG=$espConfig" build
# Full flash of the replacement, including bootloader and partition table:
idf.py -B $espBuild "-DSDKCONFIG=$espConfig" -p COM14 -b 460800 flash
```

The initial full-flash backup is under
`D:\source\vectrx2020_firmware_backups\esp32-com14-before-20261119-103348`.

### Read the chip identification

Close serial monitors before accessing the port. Run from the initialized
ESP-IDF shell:

```powershell
python -m esptool --chip esp32 --port COM10 --baud 460800 flash_id
```

This reads the flash-chip identification without writing flash, but normally
resets the ESP32 into its downloader and interrupts normal operation. Record
the manufacturer, device ID, and detected flash size.

The build's configured flash size and the physical chip capacity are separate.
This workspace previously used an 8 MB configuration while physical detection
reported 4 MB. Resolve that discrepancy for the actual module before a full
flash. A 4 MiB backup covers only the lower half of an 8 MiB device.

## Update only the application on the existing device

The existing device's factory application starts at `0x10000`. Confirm the
partition layout before using this address on another device.

```powershell
python -m esptool --chip esp32 --port COM10 --baud 460800 `
    write_flash --flash_mode keep --flash_freq keep --flash_size keep `
    0x10000 "$espBuild\network_adapter.bin"
```

This writes only the application range, preserving the existing bootloader,
partition table, and settings outside that range. It does not change the
generated application's configured flash size. The application must fit its
existing partition. Do not write this application binary at address zero.

During previous successful programming, the STM32 was held halted through
ST-Link so it could not access SDIO while the ESP32 was being programmed.
Coordinate this before flashing and release the STM32 afterward. Power-cycle
both boards after programming and check startup before running games.

## Flash a new or blank module

A blank module also needs the bootloader and partition table. Once the module's
flash capacity, configuration, and partition layout have been checked:

```powershell
idf.py -B $espBuild "-DSDKCONFIG=$espConfig" -p COM10 -b 460800 flash
```

Unlike the application-only command, this writes the generated bootloader,
partition table, application, and any other images specified by the build.
Back up an existing device before using it; do not use it as a routine
application-only update.

A bare WROOM module requires suitable 3.3 V power, boot/reset wiring and a
3.3 V logic UART adapter. Resolve GPIO2/GPIO12 SDIO boot-strapping conflicts for
the exact module before programming or connecting it to the STM32.

## If COM10 reports access denied

Close monitors and check for earlier capture, debugger, or programming processes.
Do not repeatedly reset or reflash the STM32 to solve a Windows serial-port
access problem. Reconnecting the USB-UART adapter can help; in this investigation
a Windows restart was needed to release COM10. Recheck the COM number afterward.

## Saved firmware

Firmware snapshots are separate from source builds. One backup bundle is:

```text
D:\source\vectrx2020_firmware_backups\stm32-dat1_esp32-original_20261118-143913
```

Its manifest records hashes and flash addresses. That bundle contains the STM32
DAT1 image and the **original** ESP32 application. The celebrated smooth-running
session instead used the ESP32 diagnostic application. Neither pairing has been
established as reliable across all subsequent startup attempts.

Committing source does not automatically include binaries under ignored build
directories or record what is currently flashed onto either board.
