# About this fork

This is a fork of **Polo File Manager**, originally created by
**[Tony George](https://github.com/teejee2008)**. The original project is now
archived, and this fork aims to keep it alive — though it is expected to
**diverge significantly** from the original over time, following the
maintainer's plans.

It is maintained by **Stefano Balzarotti (Orbintsoft)**, with
[Claude](https://www.anthropic.com) as an AI support tool under human review.
The project stays free software under **[GPL-3.0-or-later](LICENSE)**; author
and contributor credits are kept in [AUTHORS](AUTHORS).

**Contributions are welcome** and will be credited. The long-term hope is to
hand the project back to the community, should someone one day wish to maintain
it.

# Polo File Manager

Polo is a modern, light-weight file manager for Linux with support for multiple
panes and tabs; support for archives, and much more.

## **Features**

- **Multiple Panes** - Single-pane, dual-pane (vertical or horizontal split) and
  quad-pane layouts. Supports multiple tabs in each pane. Each tab has an
  embedded terminal which can be toggled with the F4 key.
- **Multiple Views** - List view, Icon view, Tiled view and Media view.
- **Device Manager** - Devices popup displays list of connected devices with
  options to mount and unmount. Supports locking/unlocking LUKS encrypted
  devices.
- **Archive Support** - Support for browsing archives as normal folders.
  Supports creation of archives in multiple formats with advanced compression
  settings.
- **PDF Actions** - Actions in right-click menu: Split and Merge pages, Add or
  Remove Password, Rotate, etc.
- **ISO Actions** - Actions in right-click menu: Mount, Boot in VM, Write to USB
- **Image Actions** - Actions in right-click menu: Rotate, Resize, Reduce
  Quality, Optimize PNG, Convert to other formats, Boot or Reduce Colors, etc.
- **Checksum & Hashing** - Actions in right-click menu to generate MD5, SHA1,
  SHA2-256 ad SHA2-512 checksums for file and folders. Double-click on files
  with **.md5, .sha1, .sha2, .sha256, .sha512** extensions to verify.
- **Video Downloads** - Paste URLs from YouTube and other video websites in a
  folder to download video files. Integrates with youtube-dl utility.

## Screenshots

_Single-pane layout with Sidebar and IconView_

![](src/share/polo/images/polo_layout_single_icons.png)

_Single-pane layout with Sidebar and ListView_

![](src/share/polo/images/polo_layout_single_list.png)

_Dual-pane layout with IconView_

![](src/share/polo/images/polo_layout_dual_icons.png)

_Dual-pane layout with ListView_

![](src/share/polo/images/polo_layout_dual_list.png)

_Options for creating archives - Supports all encoder options, encryption, and
split archives_

![](images/polo_compress.png)![](images/polo_compress_expanded.png)
![](images/polo_compress_progress.png)

_Support for browsing cloud storage accounts_

![](images/cloud_storage.png)

## Installation

PPA and DEB files are available for Ubuntu and Ubuntu-based distributions.
Binary installers are available for all Linux distributions.

**[Installation](https://github.com/RogueScholar/polo/wiki/Installation)**

## License

Polo is free software, released under the
**[GNU General Public License v3.0 or later](LICENSE)** (GPL-3.0-or-later).

The project was originally created by **Tony George**
(`teejeetech@gmail.com`) and distributed under GPL-2.0-or-later. As permitted
by the "or later" clause of that license, this fork is now distributed under
GPL-3.0-or-later. The original copyright notices are retained in every source
file, and all credit for the original work belongs to Tony George and the
Polo community of contributors.
