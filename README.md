# applesmc-next
Patches to the Linux kernel that allow setting battery charge thresholds on
Intel Apple devices.

This includes a [TLP](https://linrunner.de/en/tlp/tlp.html) script for use with
that system. The file should be placed at `/usr/share/tlp/bat.d/45-apple`.
Otherwise the interface can be accessed directly via sysfs:
```
/sys/class/power_supply/BAT0/charge_control_end_threshold
/sys/class/power_supply/BAT0/charge_control_full_threshold
/sys/class/power_supply/BAT0/charge_control_start_threshold
```

Note `charge_control_start_threshold` is not currently implemented.

Example:
```
# echo 80 > /sys/class/power_supply/BAT0/charge_control_end_threshold
```

This will cause charging to stop at 80% charge. Note this change is persistent
across power cycles and operating systems unless the SMC is reset.

Setting `charge_control_full_threshold` controls when the charging LED turns
green and is cosmetic only. Generally it doesn't need to be set.

Note that for `charge_control_full_threshold` to work it needs to be set to a
value less than than the end threshold because the battery will tend to float
slightly under the maximum. Subtract at least 2% or more from the end
threshold.

### TLP 1.10 and later

`45-apple` targets the battery plugin API of older TLP releases. For TLP 1.10+
use `tlp/62-applesmc-next`, installed as `/usr/share/tlp/bat.d/62-applesmc-next`.
It only claims the battery when the applesmc-next `charge_control_end_threshold`
attribute is present (otherwise TLP falls back to its generic plugin), applies
`STOP_CHARGE_THRESH_BAT0` (10..100, the range the driver accepts) and ignores `START_CHARGE_THRESH_BAT0`,
since the SMC has no start threshold and the driver's start attribute is a
read-only placeholder. Offline behaviour test: `sh tlp/test-62-applesmc-next.sh`.

### Linux 7 status

On the stock Debian/Kali 7.1.5 kernel (verified against upstream v7.1.5
`drivers/acpi/sbshc.c` and a running system) `sbshc` and `sbs` bind as platform
drivers and `sbshc` stores its opaque `struct acpi_smb_hc` with
`platform_set_drvdata()`. Other kernels and distribution configs may differ. The
`sbs` probe therefore looks up the host controller in the parent's platform
device drvdata, falls back to the ACPI driver data, and defers instead of
dereferencing NULL.

Dependencies on kernel interfaces that can change between versions:

1. how `sbshc` publishes its host-controller pointer (platform drvdata vs ACPI driver data);
2. the exported `sbshc` API (`acpi_smbus_read/write`, callback registration) declared in `sbs/sbshc.h`;
3. the `acpi_battery_hook` callback signature (changed in 6.2) used by `sbs_hook_register()`;
4. the SMC `BCLM` key and its accepted range; the driver accepts 10..100.

| Item | Status |
| --- | --- |
| Both modules build (DKMS) against Kali 7.1.5, MacBookPro12,1, no compiler warnings | tested |
| Module load and stop-threshold write/readback on Kali 7.1.5 / MacBookPro12,1 | tested |
| TLP 1.10.2 plugin, offline test with simulated sysfs (10..100, rejects 0/1/9/101 without writing) | tested |
| TLP 1.10.2 detects the live driver and applies the stop threshold | tested |
| Charger-connected test at 89%: saved 100% limit resumes charging (~1.54 A); 80% stops charging (0 mA) | tested |
| One reboot on Kali 7.1.5 / MacBookPro12,1: custom modules and TLP adapter load automatically; 80% limit retained; AC connected at 89%, charging stopped (0 mA) | tested |
| Suspend/resume with replacement modules | **untested** |

Contributors and provenance:

- `sbs` host-controller lookup and NULL guard: Graham, upstream PR #15.
- Mutex around the SMC read in `applesmc_percent_show()` and returning the
  real `device_create_file()` error in `applesmc_battery_add()`: Michael
  Moreira Melo, upstream PR #12 (only these lines; its range and formatting changes
  are not included).

[Arch Linux AUR package](https://aur.archlinux.org/packages/applesmc-next-dkms)

To build on other distributions install the `dkms` package. The DKMS version
is the static `PACKAGE_VERSION` in `dkms.conf` (currently `0.1.6`), and the source
directory name must match it, so read it from the clone rather than from a tag:
```
#!/bin/sh
set -e
git clone --depth 1 https://github.com/c---/applesmc-next /tmp/applesmc-next
VER="$(sed -n 's/^PACKAGE_VERSION="\(.*\)"/\1/p' /tmp/applesmc-next/dkms.conf)"
mv /tmp/applesmc-next "/usr/src/applesmc-next-$VER"
dkms install "applesmc-next/$VER"
```
Older release tags still have the `@_PKGBASE@`/`@PKGVER@` placeholders. To build
one of those untouched releases, set `PACKAGE_NAME="applesmc-next"` and
`PACKAGE_VERSION="<tag>"` in its `dkms.conf`, place it in
`/usr/src/applesmc-next-<tag>` and run `dkms install "applesmc-next/<tag>"`.

### Reverting a test installation

Keep the original driver files and TLP configuration before installing. Before
changing the stop threshold, record its original value through the new driver.
The SMC setting persists across reboots and operating systems; removing the
module alone does not restore it.

To revert, restore the recorded stop threshold while the custom driver is still
loaded, verify readback, remove the added TLP adapter, then remove this test's
DKMS registration so a later header installation cannot reinstall it. Reload the
stock modules only when their reference counts permit; never force-unload them.
Retain the saved source and original module files until stock operation is
verified. Loading an external module can set out-of-tree or signature taint flags.
