# LCOS Software Center

A software store for LCOS and other Devuan-based systems, in the spirit of Linux Mint's
Software Manager. Browse by category, search, read details, install, remove and update,
for both **APT** (the Devuan repositories) and **Flatpak** (Flathub).

Python 3 and GTK 3, so it fits XFCE. It does not use or require systemd, PackageKit or aptdaemon.

![The main window: category sidebar, search box, and a list of APT and Flatpak apps](docs/screenshots/main-window.png)

## Features

- Categories, search, Featured, Installed and Updates views; filter by source (APT or Flatpak).
- Detail page with description, version, website, and download and installed sizes.
- Install, Remove, Update and **Open** (launches the app) buttons.
- Before removing something, shows any extra packages that would go with it. Warns if a change
  would touch init or session packages (`systemd-sysv`, `sysvinit-core`, `openrc`, `elogind`, ...).
- Checks your setup and tells you about problems (see [Health checks](#health-checks)).
- An **About** button in the header (next to Refresh; some icon themes draw it as a star) shows the version.

## Install

Download the `.deb` and install it with apt, which pulls in the dependencies:

```
sudo apt install ./lcos-software-center_<version>_all.deb
```

The app appears in the menu under System, or run `lcos-software-center`. Remove it with
`sudo apt remove lcos-software-center`. If you ever used `sudo make install`, run
`sudo make uninstall` from the source folder first.

apt may print a notice that the download was done "unsandboxed as root". That is harmless; copy
the `.deb` to `/tmp` first to avoid it.

You also need a polkit authentication agent running in your XFCE session, or no password prompt
will appear. If you never get one, install one (for example `xfce-polkit` or `policykit-1-gnome`;
the name varies by release).

### Dependencies

The `.deb` depends on `python3` (3.9 or newer), `python3-gi`, `gir1.2-gtk-3.0`, `python3-apt` and
`pkexec` (or `policykit-1` on older releases), and recommends `flatpak`.

### Run from the source folder

```
sudo apt install python3-gi gir1.2-gtk-3.0 python3-apt flatpak pkexec
./bin/lcos-software-center
```

This works for browsing and installing, but pkexec shows a generic "run program as root" prompt,
and the sources fix described below is disabled. Use the `.deb` or `sudo make install` for the full set.


## Troubleshooting

**A Flatpak app isn't in the XFCE menu.** The detail page's **Open** button works regardless. For
the menu entry, Flatpak's launcher folders must be in `XDG_DATA_DIRS`, which is set at login, so log
out and back in once after Flatpak is first installed. Check with
`echo $XDG_DATA_DIRS | tr ':' '\n' | grep flatpak`. You can always run an app by hand:
`flatpak list --app`, then `flatpak run <App ID>` (the full ID, such as `com.github.dahenson.agenda`).

**A Flatpak app installs but won't start**, with `bwrap: Creating new namespace failed ... (ENOSPC)`.
Flatpak's sandbox needs unprivileged user namespaces, and some systems switch them off
(`user.max_user_namespaces = 0`). To fix it:

```
grep -H . /proc/sys/user/*                       # look for a 0
sudo sysctl -w user.max_user_namespaces=15000
echo 'user.max_user_namespaces=15000' | sudo tee /etc/sysctl.d/99-userns.conf
```

Restart the store afterwards to clear the warning. This loosens a hardening setting, but Flatpak
cannot run without it.

**Flatpak sizes show "unknown".** Run
`LC_ALL=C.UTF-8 flatpak remote-ls --app --columns=application,download-size,installed-size | head -3 | cat -A`.
If it prints nothing, your Flatpak is too old to report sizes.

**No password prompt appears.** Start a polkit authentication agent in your session (see Install).

## Build and test

```
make test          # unit tests; no GTK or root needed
make deb           # writes dist/lcos-software-center_<version>_all.deb (needs dpkg-deb, make, python3)
sudo make install  # PREFIX=/usr by default; sudo make uninstall to remove
```

## Licence

Copyright © 2026 BDM. Licensed under the GNU General Public License, version 3 or any later version (GPL-3.0-or-later). 


