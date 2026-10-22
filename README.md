# RL78/F24 MCU CAN FD device App

Independent App paired with sibling `RL78_F24_Boot_loader_CAN_FD`.
The App uses its sibling's board/CAN/clock/metadata sources and its own SC output.

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
Shared engineering contracts are in the sibling Boot headers.

The required sibling layout is:

```text
workspace/
  RL78_F24_Boot_loader_CAN_FD/       # shared sources and build helpers
  RL78_F24_Boot_loader_CAN_FD_app/   # App-local image/IRQ settings
```

Keep these directory names when relocating checkouts. The App project alone
does not include its required Boot-owned common sources and PowerShell helpers.

From the Boot directory:

```powershell
.\build.ps1 -Kind app
.\test.ps1
```

Use CS+ CC-RL with Windows PowerShell5.1/.NET.
Pre-build generates metadata/vectors; post-build validates actual linked bytes and emits the packaged image/report.
App code starts0x2200,table0x2000,metadata0x2100..0x21FF.
Capacity is0x3DC00; final1KiB is protected.

## FF-filled App test images

After a successful App build, double-click `generate_test_bins.bat`, or run
`powershell -NoProfile -ExecutionPolicy Bypass -File .\generate_test_bins.ps1`
from this App directory. The matching sibling Boot supplies
`tools/app_config.ps1`; no Python, SRecord or MCU compiler invocation
is needed.

The script reads the packaged `DefaultBuild/*_update.bin` and creates a new
`test_bins/<batch>/` with test BINs, `manifest.csv` and `skipped.csv`.
Default Flash budgets are 16/32/64/128/256/512 KiB, including Boot; the actual
BIN subtracts the Boot reserve and respects metadata capacity. With the current
App, lengths are 8/24/56/120/247 KiB; the last 1 KiB remains protected. An App
that no longer fits a budget is skipped rather than truncated. One KiB is 1024 bytes.

App code and vectors remain unchanged. FF padding starts at the original CRC
location; metadata image_size at BIN `0x118..0x11B` is updated, and the new final
four bytes contain little-endian CRC32. Version/build/policy remain unchanged.
The original BIN is never overwritten. Test outputs are ignored by Git and
can be regenerated after each App build; both generator scripts are shared.
Use the existing Tool CAN FD Upgrade workflow and keep its JSON operation report.
See the [test-image guide](https://released.github.io/slide_boot_loader_for_RL78/#device-test-images).

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
See [CAN FD operation](https://released.github.io/slide_isp_tool_for_RL78/#can-fd-update) and
[metadata/vector editing](https://released.github.io/slide_boot_loader_for_RL78/#device-metadata-vector-workshop).
Clean-build/model results do not establish physical FD update acceptance.
Normal500K/2M/BRS App->ENTER->update->APP_READY operation has also been verified
on the board,including the UART counter and zero recorded Host error-counter
increases during the checked update. Sustained stress and power/fault recovery
require separate evidence.
Receive configuration,queue limits and Flash/IRQ ownership are described in
the [FD engineering guide](https://released.github.io/slide_boot_loader_for_RL78/#can-fd-device-rx).
