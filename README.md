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
- A **Colors** button to customize the interface (see [Customizing colors](#customizing-colors)).
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

## Customizing colors

Click **Colors** in the header. For each of the **Header**, **Side panel** and **Main panel**, tick
*Use custom*, pick a background and, optionally, a text color. With text set to *Automatic* the app
chooses black or white, whichever is easier to read on your background. Changes preview live; **Save**
keeps them, **Cancel** undoes them, and **Reset all** returns everything to your system theme.
Unticked areas always follow the system theme.

Your choices are stored in `~/.config/lcos-software-center/theme.json` (only `#rrggbb` values are
accepted). Delete that file to start over. Buttons, the search box and the selection highlight keep
your system theme so they stay legible.

## How it works

| Piece | What it does |
|---|---|
| `lcos_software_center/ui.py` | The GTK 3 window. |
| `lcos_software_center/theme.py` | Loads, validates and saves the color settings and turns them into CSS. |
| `lcos_software_center/apt_backend.py` | Lists packages with `python-apt`, dry-runs changes with `apt-get -s`, guards init packages. |
| `lcos_software_center/flatpak_backend.py` | Wraps the `flatpak` command: listing, sizes, install, uninstall, update. |
| `lcos_software_center/appstream.py` | Reads AppStream XML (system and Flatpak) for names, summaries, categories, icons. |
| `lcos_software_center/localmeta.py` | Reads the `.desktop` and MetaInfo files of installed packages for real names, icons and categories. |
| `lcos_software_center/launcher.py` | The Open button: `flatpak run`, or the package's `.desktop` launcher for APT apps. |
| `lcos_software_center/repocheck.py`, `sourcesfix.py` | Source checks and the "Fix it" button. |
| `lcos_software_center/sandbox_check.py` | Detects Flatpak's user-namespace problem. |
| `helper/lcos-sc-helper` | The root helper, run through `pkexec`. |
| `data/featured.json` | The curated Featured list. Edit it for LCOS. |

**Privileges.** The GUI never runs as root. APT changes go through `pkexec` to the helper, which
accepts only a fixed set of commands (`install`, `remove`, `upgrade`, `upgrade-all`, `update`,
`fix-sources`) with strictly validated arguments, at most 20 package names per call, and never a
`dist-upgrade`. Flatpak does its own polkit authorization for system-wide installs. Set
`LCOS_SC_FLATPAK_SCOPE=user` to install Flatpaks into `~/.local/share/flatpak` with no prompt.

**Metadata.** Devuan ships no AppStream catalog, so metadata comes from three places: Flatpak apps
use Flathub's catalog; installed APT apps use their own `.desktop` and MetaInfo files; uninstalled
APT apps are classified from Debian debtags (the `Tag:` field), then Debian sections, then a
"depends on a GTK/Qt/wx/FLTK library" guess. Uninstalled APT apps therefore often show a generic icon,
and the category guess is imperfect. `data/featured.json` is the reliable way to put specific apps up front.

## Health checks

The store looks at your system at startup and shows a bar when something needs attention.

**Non-Devuan repositories.** Warns if Debian, Ubuntu or Mint repositories are configured. Devuan
advises against this, since they can pull in systemd. See <https://www.devuan.org/os/packages>.

**Suite names instead of codenames.** If a Devuan source line says `stable` (or `testing`, ...),
the store names your real release and, when it is safe, offers **Fix it…**. It shows the exact lines
that will change, and on confirmation (one password prompt):

1. backs up each file to `/var/backups/lcos-software-center/<timestamp>/`,
2. changes only the suite word on Devuan lines (comments, options and vendor repositories such as
   Chrome or Brave are untouched),
3. runs `apt-get update`.

The codename is never guessed. It is the `Codename` in the repository's own `Release` file, which
`apt update` already downloaded for that suite. It refuses, and says why, when there is no release
information yet (run a refresh first), when the repository returns something unusable, when
`/etc/os-release` names a different known Devuan release than the suite points to (you may be
part-way through an upgrade, so edit by hand), or when the files changed after the preview. The
helper rebuilds the plan itself as root and checks it against what you approved, and loads its code
only from a root-owned install. Installing the package never edits `/etc/apt`. To undo, copy the file
back from the backup folder.

**https on deb.devuan.org.** Devuan recommends plain http there; packages are verified with
OpenPGP signatures either way.

**Flatpak sandbox.** If user namespaces are disabled, Flatpak apps install but won't start. The
store warns and asks before you install a Flatpak. See Troubleshooting.

## First things to check on a new machine

1. Open the store, search `gimp`, click it and Install. Expect a password prompt, a progress bar,
   and the button turning into Remove.
2. Click the refresh button (it runs `apt-get update` through the helper).
3. If Flathub isn't set up, use the bar's **Enable Flathub** button, then install a Flatpak.
4. Try removing something with dependents and confirm the extra-packages dialog appears.

`apt-cache show geany | grep -i '^Tag'` shows whether your repositories provide debtags. With
tags, categories are decent; without, the store falls back to the noisier library-dependency guess.

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
MAINTAINER="Your Name <you@example.org>" ./packaging/build-deb.sh
```

The version comes from `lcos_software_center/__init__.py`. The package maintainer defaults to
"BDM <noreply@lcos.invalid>"; set a real address with the `MAINTAINER` variable when you publish.

## Licence

Copyright © 2026 BDM. Licensed under the GNU General Public License, version 3 or (at your option)
any later version (GPL-3.0-or-later). The full text is in `LICENSE`; the `.deb` points to
`/usr/share/common-licenses/GPL-3`.

## Known gaps

- No screenshots, ratings or reviews.
- One operation at a time, deliberately, because of the dpkg lock.
- Flatpak sizes cover the app itself; a first install may also download a runtime that isn't counted.
- AppStream is read from XML only. If Devuan starts shipping a catalog as XML under
  `/usr/share/swcatalog` it is picked up automatically; DEP-11 YAML would need a reader.
- Search covers name, package id and summary, not full descriptions.
- Update lists cover installed packages only.
- Packaging uses plain `dpkg-deb`. A full `debian/` source package (debhelper, `dh-python`) would be
  the next step to host it in an LCOS apt repository.
