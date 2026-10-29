# RL78/F24 MCU CAN FD device App

App firmware for a compatible `RL78_F24_Boot_loader_CAN_FD` Boot.
The App owns its board/CAN/clock/metadata sources and SC output.

## Runtime

Nominal500K,data2M,BRS; standard request0x700,response0x701.
App INFO reports live identity and APP_READY after initialization.
CAN reception uses RXMB0 interrupt -> eight-entry RAM queue -> main loop.
`app_irq_handlers.h` maps0x4C to its global,no-vect `Device_CanRxIsr`;
physical vectors remain Boot-owned. Driver ISR generation is disabled to
avoid competing physical vectors. UART identifies the receiver and reports
nonzero sticky RX faults/discard counts beside the periodic runtime counter.
ENTER acknowledges before reset into Boot.
Diagnostic UART115200 prints Boot handoff/update provenance and `log(timer)-V1: <counter>` once per second.
Change the observation tag in `app_runtime_log.h`; image release identity remains in `app_image_config.h`.

## Editing and build

Edit `app_image_config.h` for release identity/version policy and `app_irq_handlers.h` for App-local ISR mappings.
Use global,no-vect interrupt handlers; Boot and App are separately linked images.
Generated `ftable.inc`,`app_identity.h`,`app_irq_table.inc` and ASM are not manual inputs.
Engineering contracts are local configuration headers.

The App builds using its own source tree and PowerShell helpers.

From the Boot directory:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .uild.ps1
```

Use CS+ CC-RL with Windows PowerShell5.1/.NET.
Pre-build generates metadata/vectors; post-build validates actual linked bytes and emits the packaged image/report.
App code starts0x2200,table0x2000,metadata0x2100..0x21FF.
Capacity is0x3DC00; final1KiB is protected.

## Artifacts and operation

| Artifact | Purpose |
| --- | --- |
| `DefaultBuild/RL78_F24_Boot_loader_CAN_FD_app_update.bin` | Tool CAN FD App update |
| `DefaultBuild/flash_app_variable_CRC.hex` | Programmer App installation |
| `DefaultBuild/flash2000_3FBFF.hex` | Fixed-range programmer App image |
| `boot_app.hex` / `boot_app.bin` | Combined programmer image; raw BIN starts0 |
| `DefaultBuild/app_build_report.html` | Browser-readable actual build summary |

Install Boot first using the programmer.
Tool uses a verified host with FD burst capability bit13,CAN FD /500K /2M,Discover,Enter Boot as applicable and the update BIN.
The App/Boot metadata requires FD mode; Classical update images are rejected.
See [CAN FD operation](https://released.github.io/slide_simple_renesas_isp_tool/#can-fd-update) and
[metadata/vector editing](https://released.github.io/slide_boot_loader_for_RL78/#device-metadata-vector-workshop).
Clean-build/model results do not establish physical FD update acceptance.
Normal500K/2M/BRS App->ENTER->update->APP_READY operation has also been verified
on the board,including the UART counter and zero recorded Host error-counter
increases during the checked update. Sustained stress and power/fault recovery
require separate evidence.
Receive configuration,queue limits and Flash/IRQ ownership are described in
the [FD engineering guide](https://released.github.io/slide_boot_loader_for_RL78/#can-fd-device-rx).

## App timing test images

A successful App build also produces FF-filled test BINs under `test_bins/auto/`.
Open `test_bins/auto/index.html` to select an image by actual size, CRC and hash.
The set includes supported Flash-budget tiers and the declared App capacity;
tiers smaller than the executable or larger than physical Flash are skipped.
The packaged `DefaultBuild/*_update.bin`, code, IRQ tables and release settings
remain unchanged. Only test-image length, FF padding and final CRC are updated.
Automatic rebuilds refresh this one owned set; changed or unknown files stop
replacement. Manual `generate_test_bins.bat` creates a separate batch.
Use the matching ISP workflow and keep its JSON/HTML operation report for timing.
See the [test-image guide](https://released.github.io/slide_boot_loader_for_RL78/#device-test-images).

## User code entry

`custom_func.c` provides `hardware_init()` for initialization and `loop()` for
foreground service. Add application initialization and foreground work through
these callbacks, retaining the existing communication service calls.
The selected main/startup adapter owns entry and handoff. Generated peripheral
files remain under `src/`; retain the project's MTPJ source selection when using
Code Generator or Smart Configurator, then rebuild to validate the image and vectors.

## Project layout and standalone build

This project includes its own source files, configuration headers and build tools.
A sibling Boot, App or Transfer checkout is not required to compile it.
Keep the project directory name aligned with its `.mtpj` filename; remove a ZIP download suffix when necessary.
Build with CS+ Rebuild, or run the local PowerShell entry:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\build.ps1
```

| Location | Responsibility |
| --- | --- |
| `custom_func.c` | Project initialization in `hardware_init()` and recurring work in `loop()`; runtime orchestration stays here |
| `hal/can` | CAN service, controller driver and CAN adapter, when used by this project |
| `hal/device` | Device, clock, peripheral and CPU handoff APIs |
| `hal/image` | Metadata parser/profile, image validation and software CRC |
| `hal/flash` | Flash access/geometry APIs, when required by the selected image |
| `hal/isp` | Update/Transfer protocol and transport processing APIs |
| Root configuration headers | Local MCU/Flash, handoff and communication contract |
| `tools` and root PowerShell helpers | Local metadata/IRQ generation and linked-image checks |
| `src` | Code Generator or Smart Configurator peripheral sources |

Only the sources registered in this project's CS+ configuration are linked.
Generated main/vector implementations must not become a second active entry.
Retain the `custom_func.c` callbacks and the selected startup adapter after code generation.

Boot and App each carry a local contract. Keep their MCU, Flash layout, image format,
IRQ ABI, handoff and communication settings compatible when installing a pair.
App releases are edited in `app_image_config.h` and `app_irq_handlers.h` in the App project.
Rebuild the selected App to regenerate its metadata, vector table, update BIN and timing BINs.
The App build does not read or rewrite a Boot checkout.
