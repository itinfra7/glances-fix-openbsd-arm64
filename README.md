# Glances Fixes for OpenBSD arm64

Apply OpenBSD-specific compatibility fixes to an installed Glances package on OpenBSD arm64/aarch64.

## Overview

This repository contains a small patch set for Glances 4.5.1 and psutil 7.2.2 on OpenBSD 7.9 arm64.

The patches target package-installed files and do not vendor Glances or psutil source code. The wrapper preserves the normal curses interface on real terminals, while the Python patches keep Glances process and sensor handling available on OpenBSD.

## Included Fixes

### Non-TTY launcher

`glances-nontty-wrapper` preserves the package launcher for real terminals and explicit non-UI modes. A plain non-TTY invocation returns one useful snapshot instead of attempting to initialize curses.

### Battery capability detection

`glances-battery.patch` treats the unavailable OpenBSD `psutil.sensors_battery()` API as an unsupported optional sensor. Other Glances sensors remain enabled.

### OpenBSD process command lines

`psutil-openbsd-cmdline.patch` makes the OpenBSD psutil backend reuse its existing safe `EINVAL` handling for process command-line queries. An unreadable process returns an empty command line instead of terminating the process list update; the process and process-count plugins remain enabled.

## Requirements

- OpenBSD 7.9 on arm64/aarch64
- Root privileges
- Glances 4.5.1 installed from the OpenBSD package
- Python 3.13.13 and psutil 7.2.2
- The OpenBSD `patch`, `install`, and `cp` utilities

## Default Paths

```sh
GLANCES=/usr/local/bin/glances
PACKAGE_LAUNCHER=/usr/local/libexec/glances-package
PYTHON_SITE_PACKAGES=/usr/local/lib/python3.13/site-packages
BACKUP_ROOT=/var/backups/glances-backup
```

Adjust the Python site-packages path when using a different Python package layout.

## Backup

Run this as root before applying the patches and keep the resulting directory for rollback.

```sh
stamp=$(date +%Y-%m-%d_%H%M%S)
backup_dir=/var/backups/glances-backup/$stamp
glances_launcher=/usr/local/bin/glances
package_launcher=/usr/local/libexec/glances-package
battery_file=/usr/local/lib/python3.13/site-packages/glances/plugins/sensors/sensor/glances_batpercent.py
psutil_file=/usr/local/lib/python3.13/site-packages/psutil/_psbsd.py

install -d -m 0700 "$backup_dir"
cp -p "$glances_launcher" "$backup_dir/glances-package-launcher"
cp -p "$battery_file" "$backup_dir/glances_batpercent.py"
cp -p "$psutil_file" "$backup_dir/_psbsd.py"

if [ -e "$package_launcher" ]; then
    cp -p "$package_launcher" "$backup_dir/glances-package-launcher-existing"
fi
```

The backup contains the original launcher, Glances battery plugin, psutil BSD backend, and any pre-existing preserved launcher.

## Install

Run as root from this repository directory:

```sh
glances_launcher=/usr/local/bin/glances
package_launcher=/usr/local/libexec/glances-package
battery_dir=/usr/local/lib/python3.13/site-packages/glances/plugins/sensors/sensor
psutil_dir=/usr/local/lib/python3.13/site-packages/psutil

install -d -m 0755 /usr/local/libexec
if [ ! -e "$package_launcher" ]; then
    cp -p "$glances_launcher" "$package_launcher"
fi
install -m 0755 glances-nontty-wrapper "$glances_launcher"
patch -d "$battery_dir" -p0 < glances-battery.patch
patch -d "$psutil_dir" -p0 < psutil-openbsd-cmdline.patch
```

## Verification

Check the version and exercise both the non-TTY and real-TTY paths.

```sh
glances --version
glances
```

A real terminal keeps the normal curses interface. A non-TTY invocation returns one snapshot and exits successfully.

The process backend can be checked independently:

```sh
python3.13 - <<'PY'
import psutil

processes = list(psutil.process_iter(attrs=["pid", "name", "cmdline"], ad_value=None))
print(f"processes={len(processes)}")
PY
```

## Restore

Set `backup_dir` to the directory created by the backup step, then run as root.

```sh
backup_dir=/var/backups/glances-backup/YYYY-MM-DD_HHMMSS
glances_launcher=/usr/local/bin/glances
package_launcher=/usr/local/libexec/glances-package
battery_file=/usr/local/lib/python3.13/site-packages/glances/plugins/sensors/sensor/glances_batpercent.py
psutil_file=/usr/local/lib/python3.13/site-packages/psutil/_psbsd.py

cp -p "$backup_dir/glances-package-launcher" "$glances_launcher"
cp -p "$backup_dir/glances_batpercent.py" "$battery_file"
cp -p "$backup_dir/_psbsd.py" "$psutil_file"

if [ -f "$backup_dir/glances-package-launcher-existing" ]; then
    cp -p "$backup_dir/glances-package-launcher-existing" "$package_launcher"
else
    rm -f "$package_launcher"
fi
```

The restore returns the launcher and package-owned Python files to their pre-application state.

## Package Upgrades

Glances or psutil package upgrades may replace the patched files. Recreate a backup and reapply the patches after such an upgrade.

## License And Credits

- Glances: https://github.com/nicolargo/glances
- psutil: https://github.com/giampaolo/psutil
- OpenBSD compatibility changes: itinfra7
