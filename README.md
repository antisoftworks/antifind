<p align="center"><img src="docs/icon.svg" width="96" height="96" alt=""></p>

<h1 align="center">AntiFind</h1>

<p align="center"><b>Instant file search, finally on Linux.</b><br>
Every file name on your computer in memory, found in milliseconds, kept current as files change.</p>

<p align="center"><a href="https://antisoftworks.com/antifind">Product page</a> · <a href="#install">Install</a> · <a href="BENCHMARKS.md">Benchmarks</a> · <a href="#licence-and-price">Price</a> · <a href="https://github.com/antisoftworks/antifind/issues">Report a problem</a></p>

---

On Windows, people have had instant file search for years with Everything. Linux never had
anything like it: `find` walks the disk every time, `locate` answers from a database that went
stale last night, and desktop search indexes contents slowly. AntiFind reads the filesystems
directly, keeps every name in memory and follows every change as it happens, so a search over
millions of files takes milliseconds and is never out of date. It works on Windows too.

This repository holds the releases, the documentation and the issue tracker. The source code
is not public.

## How fast

Searching for a name that contains `readme`, warm cache, median of 30 runs, every match printed:

| data set | entries | AntiFind | plocate | fd | GNU find |
|---|---|---|---|---|---|
| 10 M synthetic files, btrfs | 10 066 328 | **177 ms** | 1.90 s | 1.15 s | 17.2 s |
| /home, btrfs | 4 515 190 | **27 ms** | 151 ms | 522 ms | 8.15 s |
| / (root), btrfs | 1 844 767 | **7.9 ms** | 25 ms | 145 ms | 2.22 s |
| NTFS volume | 1 067 954 | **5.4 ms** | 25 ms | 147 ms | 2.48 s |
| ext4 SSD | 160 071 | **1.8 ms** | 5.7 ms | 26 ms | 255 ms |

A new file can be found 8 to 14 ms after it is made; plocate finds it only after its next
`updatedb` (17.6 s for 10 million files). All tools returned the same files. Where another
tool is better, [BENCHMARKS.md](BENCHMARKS.md) says so, along with the method, the machine and
every query.

![Building the index](docs/charts/build.svg)

![A new file: time until it can be found](docs/charts/fresh.svg)

## A taste

```
$ antifind report
$ antifind '*.pdf' .
$ antifind -e jpg -e png ~/Pictures
$ antifind -t d node_modules ~
$ antifind -S +1G -a
$ antifind --changed 10m ~
$ antifind --watch ~/repos
$ antifind du ~ -d 2
```

## Supported filesystems

Searches answer from memory, so they take milliseconds on every filesystem. What differs is how
the names are read the first time and how they stay current:

| where | filesystem | how the names are read | live updates |
|---|---|---|---|
| Linux | btrfs | straight from the filesystem trees, on all cores | yes, from the kernel |
| Linux | ext2, ext3, ext4 | straight from the inode tables and folders | yes |
| Linux | XFS | straight from the inode trees and folders | yes |
| Linux | F2FS | straight from the node blocks and folders | yes |
| Linux | NTFS (the ntfs3 driver) | straight from the `$MFT` | yes |
| Linux | FAT12, FAT16, FAT32 | straight from the allocation tables | yes |
| Linux | exFAT | straight from the allocation tables | the kernel reports no changes on exFAT, so it is read again every two minutes |
| Linux | any other local filesystem | folder by folder, in parallel | where the kernel reports changes |
| Linux | FUSE, NFS, SMB, sshfs, rclone | not indexed (slow drives) | search them live with `--slow` |
| Windows | NTFS | straight from the `$MFT` | yes, from the change journal |
| Windows | FAT32, exFAT | straight from the allocation tables | yes |
| Windows | network drives | refused unless `--slow`, which walks them live | not watched |
| Not yet | HFS+, APFS and ZFS readers; btrfs and ZFS snapshots | | |

A direct read is compared with the mounted volume's own view, and what changed during the read
is read again through the kernel; a disk is never frozen. With a cold cache a folder scan is
limited by the disk, which is why the direct readers matter most on spinning disks. You can
search while the first index is built: the filesystems that are done answer at once, and the
command line tells you the index is still being built.

### How it works

1. A small service, `antifindd`, reads every file name once and keeps them in memory: about
   80 bytes per name, so 11 million names take under 1 GB.
2. It asks the kernel to report every create, delete, rename and change
   ([fanotify](https://man7.org/linux/man-pages/man7/fanotify.7.html)), so the index stays
   current without rescanning.
3. A search scans all names at once with SIMD instructions on every core. This is limited by
   memory speed, so a search over 11 million names takes about 10 to 30 ms.
4. The index is saved on shutdown and every ten minutes when it changed. After a restart the
   service answers from the saved copy within about 2 seconds while it checks for changes in
   the background.
5. Each user sees only what they could list themselves (see Privacy below).

## Using it

`antifind PATTERN [FOLDER...]`: a pattern without `*` or `?` matches names that contain it,
ignoring case; with wildcards it must match the whole name. Folders limit the search to what
lies below them, and `.` is the current folder.

| option | meaning |
|---|---|
| `-e pdf` | extension (repeat for more) |
| `-t f`, `-t d`, `-t l` | files, folders, links |
| `-S +1G`, `-S -10k` | at least 1 GB, at most 10 KB |
| `--changed 10m`, `--changed-before 2d` | changed in the last ten minutes; longer ago than two days |
| `-d 2` | at most two levels below the folders |
| `-s` | match case (ignoring case is the default) |
| `-x` | stay off filesystems mounted below the folders |
| `-a` | every indexed filesystem |

With a filter the pattern may be left out: `antifind -e pdf .`

**Where it looks.** Without a folder, AntiFind searches the system disk: `/` and the mounts on
the same filesystem or physical disk, such as `/home` and `/boot`. Other disks, USB sticks
and tmpfs are left out unless you add `-a` or name a folder on them. `antifind --status` lists
both groups. On Windows the default is the system drive.

**Slow drives.** Cloud drives (rclone and other FUSE mounts of a cloud service), network drives
(NFS, SMB, sshfs) and other FUSE filesystems are not indexed, and AntiFind never reads them
unless you ask. Naming a folder on one says why and stops:

```
$ antifind report ~/OneDrive
~/OneDrive is a cloud drive (rclone, over the network). Searching it reads it over the network, which is slow and may download files. Add --slow to search it anyway.
```

With `--slow` the folder is walked live and gently, with your own rights, after the matches
from the index; nothing is kept. `--status` lists these drives as "not indexed: slow drive".

**Shells.** The shell turns `*.pdf` into the names of the files in the current folder before
AntiFind sees it, so quote patterns, or add the shell integration to `~/.zshrc` or `~/.bashrc`:

```
source /usr/share/antifind/antifind.zsh     # bash: antifind.bash
```

Then `antifind *.pdf .` works as typed, and `af` is a short name for `antifind` (when nothing
else is called `af`). fish has no switch for this, so quote patterns there. Without the
integration, AntiFind notices when a pattern was already expanded and says so instead of
searching for the wrong names.

The integration (also for fish: `antifind.fish`) adds three helpers:

- `afcd PATTERN [FOLDER]` goes to a matching folder,
- `afe PATTERN [FOLDER]` opens a matching file in `$EDITOR`,
- Ctrl+X f turns the word before the cursor into a matching path.

With several matches, [fzf](https://github.com/junegunn/fzf) picks one when it is installed;
otherwise you get a numbered list. Versions for Oh My Zsh, Oh My Bash, Oh My Fish and Fisher
come with the packages.

### Advanced queries

Every option above is a short form of a query word, and the words can be written directly:
`antifind --print-query -e pdf .` shows them. Words must all match (AND). `|` is OR, `!` is NOT,
`< >` groups, `"..."` is a phrase. Filters: `regex:`, `path:`, `ext:`, `size:`, `dm:`, `changed:`,
`created:`, `accessed:`, `type:`, `owner:`, `group:`, `perm:`, `depth:`, `empty:`, `mount:`,
`fs:`, `under:`, `skipmount:`, `scope:`, `case:` and `wholeword:`. Saved searches are used as
`@name`. Output can be paths, `--long`, `--json`, `-0` for `xargs -0`, `--count` or your own
`--format`. See `man antifind`.

```
$ antifind 'ext:png;jpg' 'size:>5mb' dm:last7days
$ antifind 'regex:^lib.*\.so\.[0-9]+$' --json
$ antifind --watch 'changed:<1h' ~/repos
```

### Saved searches and templates

```
$ antifind --save pics -e jpg -e png ~/Pictures     # keep a search
$ antifind @pics holiday                            # run it, with one more term
$ antifind --saved                                  # the built-in and your own searches
$ antifind @big size=500M folder=~/Videos           # a template with parameters
$ antifind @changed since=2d                        # files changed in the last two days
$ antifind @duplicates                              # same name and size, in groups
$ antifind @large-folders                           # the du view
$ antifind --save mybig 'type:file size:>={size} {folder}' --param size=size:1G --param folder=folder:~
```

Values are checked by type (size, duration, folder, extension list) and nothing runs through a
shell; a search cannot use itself. Built in: big, changed, recent, downloads, documents, pictures,
videos, audio, archives, code, empty-files, empty-folders, duplicates, old-downloads, logs,
executables, private (Linux), large-folders. `antifind --copy NAME NEW` copies one to edit. The
searches are one plain file per user.

### Live list, watch searches and marks

```
$ antifind @recent --live                   # the newest changes, refreshed in place
$ antifind @system-changes                  # hosts, startup items, services, cron, drivers...
$ antifind --mark before-install            # note the time
$ antifind --since-mark before-install      # what changed since (7 days at most)
$ antifind changes 10:00..10:30 -e conf /etc   # what changed in that half hour, from the history (30 days)
$ antifind changes yesterday "*.log" -S +10M   # grouped by folder; a file made then deleted shows once
```

The watch searches (`watch-hosts`, `watch-startup`, `watch-cron`, ...) stream what changes in
the places that matter on this system. As always, you see only what you may list, and never
content.


## Privacy

The service runs as a systemd service with only `CAP_SYS_ADMIN` and `CAP_DAC_READ_SEARCH`.
The kernel tells it who is asking (`SO_PEERCRED`), and a path is only returned if that user could
list every folder on it. Other people's private folders do not appear, not in results, not in
counts, not in watch mode and not in folder sizes. Control characters in file names are escaped in
the terminal. Tests check this at every release.

`CAP_SYS_ADMIN` is broad; the unit file confines the service with the systemd sandbox options in
its systemd unit (`antifindd.service`). POSIX ACLs are not evaluated; the mode bits decide.

On Windows the service runs as LocalSystem, because only it may read a volume's `$MFT` and
change journal; every result is checked against the asking user's own access rights. During
the 14-day trial only, AntiFind checks the trial with antisoftworks.com (an anonymous machine
hash, the platform and the version). With a licence it works fully offline. File names, the
index and your searches never leave the computer.

## Install

Linux 5.17 or newer, or Windows 10 and 11. The packages come with the first release:

| system | how |
|---|---|
| Arch, CachyOS, Manjaro | `yay -S antifind-bin` (AUR) |
| Debian, Ubuntu | our apt repository, set up as shown on the [product page](https://antisoftworks.com/antifind#install) |
| Windows | Microsoft Store |
| any | the archives on the [Releases](https://github.com/antisoftworks/antifind/releases) page here and at [antisoftworks.com/antifind/download](https://antisoftworks.com/antifind/download/), with checksums and signatures |

After installing on Linux: `sudo systemctl enable --now antifindd`, then `antifind --status`.

## Licence and price

AntiFind is made by one developer. One licence, €19 once (€14 at launch), unlocks the service,
the command line and the desktop app, on Windows and Linux. Every feature works for 14 days
from the first install, with no registration. Buy a licence at
[antisoftworks.com/antifind](https://antisoftworks.com/antifind) and enter it with
`antifind --licence KEY`. The programs are proprietary; see [LICENSE](LICENSE).

## Problems and ideas

Open an [issue](https://github.com/antisoftworks/antifind/issues) with your distribution,
filesystem and the output of `antifind --status`. Security problems: see the product page's
contact address rather than a public issue.

## Also from antisoftworks.com

[AntiPDF](https://antisoftworks.com/), a fast PDF reader and writer.
