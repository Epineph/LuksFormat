# Arch Linux root on ZFS over LUKS2, alongside Windows                                                                                                         **Original runbook date:** 2026-08-04                **Live setup, keyboard, recovery, and Windows revision:** 2026-10-15                                      **Target:** x86-64 Arch Linux, UEFI/GPT, one physical drive, Windows dual boot                            **Encryption:** LUKS2 below ZFS
**Boot manager:** rEFInd                             **Arch kernel:** `linux-lts`, pinned to the version required by ArchZFS                                                                                        This is a manual installation runbook, not a paste-all-at-once script. It has                             deliberate inspection gates before every destructive operation. If the output                             at a gate differs from the stated expectation, stop; do not improvise with a                              different disk or partition name.                                                                         **Where to start:** already at an Alpine prompt: section 6.1; missing                                     symbol keys: section 6.3.1; reopening an existing pool: section 28;                                       Windows error `0x8031005A`: section 4.3. Keep existing installation data:                                 never repeat formatting or pool creation just to resume after a reboot.

This revision expands live setup and recovery. Its shell examples were
checked for syntax; the installation has not been executed on your laptop.

The runbook ends at a bootable, networked Arch console. Install a graphical
environment only after this storage and boot foundation has passed first-boot
validation.

The resulting storage stack is:

```text
UEFI firmware
  -> rEFInd and Arch kernel/initramfs on an unencrypted FAT32 ESP
  -> mkinitcpio encrypt hook
  -> LUKS2 partition opened as /dev/mapper/cryptzfs
  -> ZFS pool rpool
  -> rpool/ROOT/arch mounted as /
```

Windows remains on its existing partitions. Nothing in this guide formats,
moves, or mounts the Windows NTFS partition.

## 1. Important limitations

1. **This procedure can destroy Windows if the wrong partition is selected.**
   Make a tested backup and Windows recovery USB first.
2. **A single-disk ZFS pool has no data redundancy.** ZFS can detect checksum
   corruption but generally cannot repair damaged file data without a mirror,
   RAIDZ parity, or an additional copy. Snapshots are not backups.
3. **LUKS protects the ZFS labels and metadata as well as file data.** The EFI
   System Partition (ESP), kernel, initramfs, and rEFInd remain unencrypted so
   that UEFI can start the system.
4. **Encryption without Secure Boot does not prevent offline modification of
   the ESP.** This guide deliberately leaves Secure Boot disabled for the first
   successful boot. Secure Boot may be added afterward with controlled key
   enrollment and signed EFI binaries.
5. **OpenZFS is an out-of-tree Linux module.** Kernel and ZFS module versions
   must match. The guide therefore installs the exact `linux-lts` package
   required by `zfs-linux-lts`; it does not assume that the newest Arch kernel
   is usable.
6. The current ArchZFS repository is officially hosted as a GitHub release but
   is still labelled `experimental` by its maintainers. They explicitly advise
   caution on non-critical systems.

## 2. Intended partition layout

The Windows partition numbers shown below are examples only.

| GPT partition | Type | Purpose | Action in this guide |
|---|---|---|---|
| Existing ESP | FAT32 / EF00 | Windows boot files | Preserve |
| Existing MSR | Microsoft reserved | Windows | Preserve |
| Existing NTFS | NTFS | Windows | Preserve |
| Existing recovery | Microsoft recovery | Windows recovery | Preserve |
| `ARCH_EFI` | FAT32 / EF00 | Recommended dedicated Arch ESP, 1 GiB | Create |
| `ARCH_CRYPT` | Linux LUKS / 8309 | LUKS2 container for ZFS | Create |

A dedicated 1 GiB Arch ESP is recommended because many Windows ESPs are too
small for a Linux kernel and initramfs. If the existing Windows ESP is at least
512 MiB and has at least 250 MiB free, it may instead be shared. The shared-ESP
branch is explicitly marked below. Never format an existing Windows ESP.

## 3. Conventions

Linux commands before section 14 run as root in Alpine Extended, normally
under BusyBox `ash`. Section 5 is run on the Linux system preparing the USB;
section 4 is run in Windows. From section 14, the indicated commands run
inside the new Arch chroot until `exit` returns to Alpine. A chroot changes
the filesystem view, not the running kernel.

Do not assume GNU options work in the minimal live shell. Section 6 installs
the utilities needed for later steps. Paths and variables marked EXAMPLE
must be replaced with the observed values for this machine.

These names are fixed throughout:

```text
GPT label for new Arch ESP:       ARCH_EFI
GPT label for encrypted root:     ARCH_CRYPT
LUKS mapper name:                 cryptzfs
ZFS pool:                         rpool
Root dataset:                     rpool/ROOT/arch
Mount point during installation:  /mnt
```

## 4. Prepare Windows before booting Linux

Perform this section from an elevated PowerShell or Command Prompt in Windows.

### 4.1 Back up and record recovery information

- Make an image or otherwise tested backup of irreplaceable data.
- Create Windows recovery/install media.
- If BitLocker is enabled, save its recovery key somewhere off the laptop.
- Record the current Windows partition layout with Disk Management or:

```powershell
Get-Disk
Get-Partition
Get-Volume
manage-bde.exe -status
```

### 4.2 Disable hibernation and Fast Startup

Fast Startup uses the Windows hibernation mechanism. Disable it before a
dual-boot installation:

```powershell
powercfg.exe /hibernate off
```

### 4.3 Check encryption and handle Windows Home correctly

Open Windows Terminal or PowerShell with **Run as administrator**. First
inspect the edition and actual drive status; these commands do not change it:

```powershell
Get-CimInstance Win32_OperatingSystem | Select-Object Caption, Version
manage-bde.exe -status C:
```

| Status of C: | Action before changing the boot configuration |
|---|---|
| Fully Decrypted, 0% encrypted | Nothing to suspend; continue. |
| Encrypted with Protection On | Save and verify the recovery key, then suspend if supported. |
| Encrypted with Protection Off | Protection is currently disabled; this is not the same as decryption. Verify status before the maintenance reboot. |
| Encryption/decryption in progress | Let the operation finish before partition work. |
| Status cannot be determined | Resolve that before assuming encryption is absent. |

Keep the recovery key somewhere accessible without booting this Windows
installation. A personal account's keys may be at
[Microsoft's recovery-key page](https://account.microsoft.com/devices/recoverykey).
A work/school-managed device may require its administrator. Check that the key
identifier matches the device; do not publish the 48-digit recovery password.

On an edition that supports full BitLocker management, suspend protection:

```powershell
Suspend-BitLocker -MountPoint 'C:' -RebootCount 0
manage-bde.exe -status C:
```

`-RebootCount 0` keeps protection suspended until you explicitly resume it.
The data remains encrypted, but a clear key is made available on the drive.

**If the command returns `0x8031005A`:** Microsoft identifies this as
`FVE_E_NO_FEATURE_LICENSE`. It is an edition/feature restriction, not a
PowerShell quoting problem. Windows Home can still use **Device Encryption**
and have an encrypted C: even though a full BitLocker operation is unavailable.
Do not infer "unencrypted" from this error, and do not upgrade Windows merely
to satisfy a line in this runbook.

After saving the recovery key, you may try the alternative supported command
interface; it is not guaranteed to bypass the edition restriction:

```powershell
manage-bde.exe -protectors -disable C: -RebootCount 0
manage-bde.exe -status C:
```

Proceed as suspended only if the command succeeds and status reports
`Protection Off`. If it also returns the edition error, choose deliberately:

1. **Keep encryption enabled:** have the verified recovery key available and
   expect that changing Secure Boot or the boot chain may request it. Boot
   Windows once after the firmware changes, before continuing partition work,
   and confirm that you can unlock it. Avoid unnecessary firmware changes.
2. **Temporarily decrypt through Device Encryption:** in Windows 11, open
   Settings > Privacy & security > Device encryption and turn it off. In
   Windows 10, look under Settings > Update & Security > Device encryption.
   This decrypts the drive, takes time, and removes its protection at rest.
   It is not a quick substitute for suspension. Keep the laptop powered and
   wait for `manage-bde.exe -status C:` to report Fully Decrypted and 0% before
   resizing or continuing. Re-enable Device Encryption afterward and verify
   the saved recovery key again.

Open the Settings page directly if available:

```powershell
Start-Process 'ms-settings:deviceencryption'
```

If the page is absent, inspect the edition/status and administrator access
rather than assuming a particular encryption state. Do not delete TPM or
recovery protectors, clear the TPM, or run `manage-bde -off` merely to silence
the suspension error. For a managed device, follow its administrator's policy.


### 4.4 Use UTC for the hardware clock

Arch keeps the hardware clock in UTC. Configure Windows to do the same so that
the two systems do not move the clock when alternating between them:

```powershell
reg.exe add `
  'HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation' `
  /v RealTimeIsUniversal `
  /t REG_DWORD `
  /d 1 `
  /f
```

### 4.5 Shrink Windows from Windows

Use Windows Disk Management to shrink `C:` and leave the desired space as
**unallocated**. Do not create a filesystem in that space.

Allow at least:

- 1 GiB for a dedicated Arch ESP; and
- 40 GiB for a minimal usable Arch/ZFS root, preferably 80 GiB or more.

Then perform a full Windows shutdown:

```powershell
shutdown.exe /s /t 0
```

## 5. Prepare the installation medium

Use **Alpine Extended, x86_64** for this runbook. Its live kernel has ZFS
support available. Alpine is the temporary installation environment; the
installed system will still be Arch. Another ZFS-capable live environment can
work, but its setup commands differ.

The image below is Alpine 3.24.2, listed on the official download page at this
revision. Use its release-specific directory rather than combining a fixed
filename with a moving `latest-stable` directory.

From an existing Linux system:

```bash
ALPINE_VERSION='3.24.2'
ALPINE_ISO="alpine-extended-${ALPINE_VERSION}-x86_64.iso"
ALPINE_URL='https://dl-cdn.alpinelinux.org/alpine/v3.24/releases/x86_64'

curl --fail --location --remote-name "${ALPINE_URL}/${ALPINE_ISO}"
curl --fail --location --remote-name "${ALPINE_URL}/${ALPINE_ISO}.sha256"
curl --fail --location --remote-name "${ALPINE_URL}/${ALPINE_ISO}.asc"

sha256sum --check "${ALPINE_ISO}.sha256"
gpg --auto-key-retrieve \
  --keyserver hkps://keyserver.ubuntu.com \
  --verify "${ALPINE_ISO}.asc" "${ALPINE_ISO}"
```

Stop if a download or verification fails. Check the signer's full fingerprint
against Alpine's official downloads/keys page; retrieving a key alone does not
establish its owner.

For a direct image write, identify the USB carefully:

```bash
lsblk --output NAME,PATH,SIZE,MODEL,TRAN,FSTYPE,MOUNTPOINTS
```

The hybrid image already supplies the boot and partition structures. If the
USB is `/dev/sdb`, the destination is `/dev/sdb`, **not** `/dev/sdb1` or
`/dev/sdb2`. Its previous GPT/DOS table does not need preparation. Do not
create an ESP, format it, or convert its partition table after writing.

Unmount each mounted partition on the verified target, for example:

```bash
sudo umount /dev/sdb1
```

Repeat only for other partitions that are actually mounted. If unmounting
fails with "busy", close the programs using that partition and retry. Do not
suppress the failure and proceed. Do not overwrite the medium supplying your
current live session or the device holding the downloaded source ISO.

**The next command destroys the selected USB's existing layout and files.**
Use `/dev/sdb` here only after verifying that it is the intended USB:

```bash
sudo dd if="${ALPINE_ISO}" of=/dev/sdb bs=4M status=progress conv=fsync
sync
```

If the resulting USB does not boot reliably, compare the written image bytes
before investigating the installer:

```bash
sudo cmp --bytes="$(stat --format=%s "${ALPINE_ISO}")" \
  "${ALPINE_ISO}" /dev/sdb
```

No output and a zero exit status mean that the compared bytes match. `dd`
copies only the image's length; it is not a secure erase of the entire USB.

**Ventoy alternative:** keep Ventoy's existing partitions and copy the verified
`.iso` to its mounted data partition as an ordinary file. Do not run the `dd`
command on a Ventoy partition. Boot the Alpine ISO from Ventoy's menu.

Keep this Markdown guide on a second data USB or the Ventoy data partition;
a directly written ISO9660 filesystem is read-only. Sections 6.3.1-6.3.2 also
provide console symbol bindings and a reusable offline keyboard kit.


## 6. Firmware, Alpine shell, keyboard, networking, and tools

Complete this section in the **Alpine live environment**, before mounting the
Arch root at `/mnt` or entering its chroot. Work through one subsection at a
time. Commands here work in Alpine's default `ash`; Bash is optional.

### 6.1 Boot the correct environment and identify your shell

1. Enter the firmware boot menu and select the USB's **UEFI** entry.
2. Disable Secure Boot for this installation procedure. Leave the existing
   AHCI/RST/storage-controller setting unchanged.
3. Select the normal Alpine Linux entry and let startup finish.
4. At `localhost login:` or a similar login prompt, type `root` and press
   Enter. A fresh official live image has no root password.
5. You should reach a prompt resembling `localhost:~#`. Commands below run as
   root, so do not prepend `sudo`; it need not exist in Alpine.

The shell is normally BusyBox `ash`, reached through `/bin/sh`. That is
expected. `help` shows **shell built-ins**, such as `exec`, `eval`, and `read`;
it does not list every installed command. Try these individually:

```sh
cat /etc/alpine-release
echo "$0"
command -v apk
command -v setup-keymap
command -v mount
busybox --list
```

`apk` is Alpine's package manager. `busybox --list` lists the applets compiled
into BusyBox; other installed programs exist separately. For usage, try
`mount --help`, `ip --help`, and `apk --help`.

| What you observe | Meaning and next step |
|---|---|
| Alpine version and paths for `apk` and `setup-keymap` | Normal live environment; continue below. |
| `help` lists only built-ins, but the checks above work | Normal shell behaviour; no repair is needed. |
| `initramfs emergency recovery shell launched` | Early boot failed; use the diagnostic branch below. |
| `/etc/os-release` identifies Arch and `pacman` exists | You are probably in Arch already; live setup commands belong outside the chroot. |

If ordinary commands are missing, first restore a normal executable search
path and retry the checks:

```sh
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

**If this is the initramfs emergency shell:** photograph the error immediately
above the prompt. These commands collect useful evidence without changing the
disk layout:

```sh
cat /proc/cmdline
cat /proc/partitions
mount
dmesg | tail -n 60
```

If a command name is unavailable, try its applet explicitly, for example
`/bin/busybox mount` or `/bin/busybox cat /proc/partitions`.

A failure to find the boot medium is not fixed by installing Bash. Check that
the ISO was written to the whole USB, that verification succeeded, and that
you selected its normal boot entry. Try another USB port without a hub. If
using Ventoy and the ISO fails to start, try a direct ISO write to a spare USB.
Do not format the internal disk to fix a live-USB startup error. Normal Alpine
login must work before continuing; the exact early-boot message determines
any further repair.

### 6.2 Console essentials and what survives a reboot

- Type the commands, not a displayed `$` or `#` prompt.
- `Tab` completes paths and command names; Up recalls previous commands.
- `Ctrl+C` cancels the current command; `q` exits `less` or `more`.
- A trailing backslash continues a command onto the next line. It must be the
  final character on that line, without trailing spaces.
- For `cat <<'EOF'`, enter the following content literally, then put `EOF`
  alone at the beginning of a line. Do not type the Markdown fence markers.
- Use `vi FILE` before installing a friendlier editor: `i` starts editing,
  Escape returns to command mode, `:wq` saves/exits, and `:q!` abandons edits.
- `Alt+F2` usually opens another virtual console; log in there as root.
  `Alt+F1` returns to the first one. Shell variables are separate per console.
- Alpine live settings and packages normally live in RAM and disappear after
  reboot. Changes to mounted target filesystems persist. Do not run
  `setup-disk` or complete `setup-alpine`'s disk-installation stage: the target
  operating system in this runbook is Arch.

### 6.3 Set the Danish console keyboard before entering passwords

Start with this command; it needs no shell punctuation:

```sh
setup-keymap dk dk
```

Alternatively, run `setup-keymap` with no arguments and choose layout `dk`,
then variant `dk`. This configures the Linux text console, independently of
whether the shell is `ash` or Bash.

Test punctuation before entering Wi-Fi or LUKS credentials. Run `cat`, type a
harmless sample containing the symbols you need, then press `Ctrl+C`:

```sh
cat
```

Do not test an actual password this way. Over SSH, character entry comes from
the client terminal; configure the keyboard on that client instead.

If the setup command exists but cannot obtain keymaps, try the package already
carried on the live medium:

```sh
apk --no-network add kbd-bkeymaps
setup-keymap dk dk
```

If that reports a missing package, follow the local-repository instructions
in section 6.5. Do not assume that internet access is required for this step.

If the setup script fails but the map is present, locate it:

```sh
find /etc/keymap /usr/share/bkeymaps -name 'dk.bmap*' 2>/dev/null
```

Use the exact path printed. For a compressed map, for example:

```sh
zcat /usr/share/bkeymaps/dk/dk.bmap.gz | loadkmap
```

For an uncompressed `.bmap`, use `loadkmap < /actual/path/dk.bmap` instead.
The direct fallback changes the current console layout; `setup-keymap` also
records it in the live system's configuration. Section 17 separately sets
`KEYMAP=dk` for installed Arch and its future unlock prompt.

### 6.3.1 Add reliable keys for less-than, greater-than, and backslash

These are **Linux virtual-console bindings**. They work in `ash`, Bash, an
editor, and console password prompts. They do not depend on Hyprland or Zsh.
They do not configure Windows, graphical terminals, SSH clients, or a firmware
password prompt; those have their own keyboard handling.

Use three adjacent letter keys as an additional way to enter the symbols:

| Combination | Character | Keymap symbol name |
|---|---|---|
| Right Alt / AltGr + J | `<` | `less` |
| Right Alt / AltGr + K | `>` | `greater` |
| Right Alt / AltGr + L | `\` | `backslash` |

The extra Control + left Alt + J/K/L bindings below provide the same characters
if the right Alt key is awkward. Ordinary J, K, and L remain letters. Only
these modifier/key combinations are replaced.

First obtain the `loadkeys` tool, supplied by `kbd`:

```sh
command -v loadkeys
apk --no-network add kbd
```

If `loadkeys` already exists, skip installation. If the ISO does not carry the
`kbd` package, that offline installation can fail: use section 6.7's Ethernet
or USB tethering, then `apk add kbd` after enabling repositories. BusyBox's
`loadkmap` alone cannot parse the text below. Section 6.3.2 provides an offline
kit for later boots that needs only BusyBox.

At the laptop's actual TTY, run:

```sh
loadkeys -
```

It now reads a keymap from your keyboard. Type these six lines, pressing Enter
after each. They contain none of the three problematic characters and need
no pipe, redirection, or here-document:

```text
altgr keycode 36 = less
altgr keycode 37 = greater
altgr keycode 38 = backslash
control alt keycode 36 = less
control alt keycode 37 = greater
control alt keycode 38 = backslash
```

On the next empty line press **Ctrl+D** to finish. The bindings now apply to
the Linux virtual consoles. Run `cat` and test them with harmless text, then
press Ctrl+C. The mapping changes characters, not the contents of passwords.

On a normal PC keyboard, J/K/L have Linux keycodes 36/37/38. If this laptop
differs, run `showkey --keycodes`, press J, K, and L, then stop typing for about
ten seconds to exit. Substitute the reported codes. This command requires the
real virtual console, not an SSH pseudo-terminal.

To undo the extra bindings in Alpine, run `setup-keymap dk dk` again. Doing so
also removes the temporary custom map. On Arch, `loadkeys dk` restores Danish.
Do not replace a familiar working LUKS passphrase because the keyboard changed;
ensure that the same intended characters can be entered at unlock time.


### 6.3.2 Prepare a reusable offline keymap for other live USB sessions                                                                                        Do this once in a working Linux console with `kbd` installed, **after** loading
Danish and applying/testing the six lines above. It saves the complete active
console map, so the rest of the Danish layout is preserved:                                                                                                    ```sh                                                dumpkeys > /root/dk-heini.map
loadkeys --parse /root/dk-heini.map
loadkeys -b /root/dk-heini.map > /root/dk-heini.bmap ```                                                                                                       The text `.map` is for `loadkeys`; the binary `.bmap` is for BusyBox `loadkmap`.
The latter can be loaded even when the ISO lacks `kbd` or internet access.
This kit is intended for the x86-64 Linux console used by this guide.
                                                     Create a small loader beside the binary map:
                                                     ```sh                                                cat > /root/tty-keys.sh <<'EOF'                      #!/bin/sh                                            # Load the neighbouring Danish console map with additional symbol bindings.                               set -eu                                              case "$0" in                                           */*) key_dir=${0%/*} ;;                              *) key_dir=. ;;                                    esac
key_file="${key_dir}/dk-heini.bmap"                  test -r "${key_file}" || {                             echo 'Missing dk-heini.bmap next to this script.' >&2                                                     exit 1                                             }                                                    busybox loadkmap < "${key_file}"
echo 'Console keys loaded: AltGr+J = less; K = greater; L = backslash.'                                   EOF
```
                                                     Copy `tty-keys.sh` and `dk-heini.bmap` together onto an ordinary writable data                            USB or a Ventoy data partition. Keep the optional `.map` as the readable source.
If `/media/notes` from section 6.4 is your verified external data partition,                              make it writable for this copy only:                 
```sh                                                mount -o remount,rw /media/notes
cp /root/tty-keys.sh /root/dk-heini.bmap /media/notes/
sync                                                 mount -o remount,ro /media/notes                     ```
                                                     On the next Alpine boot, mount that data partition as in section 6.4 and run:                                                                                  ```sh                                                sh /media/notes/tty-keys.sh
```                                                  
That command contains none of `<`, `>`, `\`, or `|`, needs no executable file                             permission, and requires no download. It loads the prepared binary map into                               the running console. It is not automatically applied just because the files                               are on a USB. If no such kit was prepared and `kbd` is absent, the offline                                text-map method cannot be used; Ethernet/tethering or another prepared medium
is needed to obtain it.                              
To use the same binary map for Alpine's loadkmap service in this live session:
                                                     ```sh
mkdir -p /etc/keymap                                 cp /root/dk-heini.bmap /etc/keymap/dk-heini.bmap
printf 'KEYMAP=/etc/keymap/dk-heini.bmap\n' > /etc/conf.d/loadkmap
rc-service loadkmap restart                          ```
                                                     This service configuration is still in the live RAM filesystem and does not
survive a normal ISO reboot. Use the USB kit again next time. Section 17.1                                provides the separate persistent configuration for installed Arch and its                                 LUKS-unlock initramfs.                                                                                                                                         ### 6.4 Find devices and open a USB containing this guide                                                                                                      Before installing additional tools, use:             
```sh                                                cat /proc/partitions
blkid
mount                                                ls /media
```                                                  
`mount` with no arguments lists current mounts. The Alpine boot medium is                                 usually already mounted somewhere under `/media`; reuse that mount. Keep it
connected because its packages, modules, or firmware may still be needed.
                                                     If `lsblk` exists, its view is easier to read:

```sh                                                lsblk -o NAME,PATH,SIZE,MODEL,TRAN,FSTYPE,MOUNTPOINTS
```                                                                                                       If it is missing, that is a package difference, not evidence that disks are                               missing. Section 6.12 installs the fuller toolset. Before then, inspecting                                `cat /sys/block/sdb/device/model` can help identify a particular USB disk.
Device letters can change across boots; identify it again each time.
                                                     For an **ordinary data partition on another USB**, substitute its verified
partition path below. Here `/dev/sdY1` is deliberately a placeholder:                                     
```sh                                                mkdir -p /media/notes
mount -o ro /dev/sdY1 /media/notes                   ls -la /media/notes                                  ```                                                                                                       This is a partition mount, unlike writing the ISO to a whole device. Do not                               use `/mnt` for notes; `/mnt` is reserved for the Arch root. If automatic type                             detection fails, use the type reported by `blkid`:                                                        | Reported type | Example read-only mount; choose only the matching row |
|---|---|                                            | `vfat` | `mount -t vfat -o ro /dev/sdY1 /media/notes` |                                                 | `exfat` | `mount -t exfat -o ro /dev/sdY1 /media/notes` |                                               | `ntfs` | Try `mount -t ntfs3 -o ro /dev/sdY1 /media/notes` if that driver is available. |               | `iso9660` | The flashed ISO; normally already mounted under `/media`. |                                 | `crypto_LUKS` | Encrypted container: unlock first; do not mount it directly. |                          | `zfs_member` | Pool member: import the pool and mount a dataset; see section 28. |

An unsupported-filesystem error needs the appropriate driver/helper, not
formatting. For an external NTFS notes drive, if `ntfs3` is unavailable,                                  install `ntfs-3g` once packages are reachable and use:                                                                                                         ```sh
apk add ntfs-3g                                      mount -t ntfs-3g -o ro /dev/sdY1 /media/notes        ```                                                                                                       For an ext4 notes partition, `mount -t ext4 -o ro,noload ...` also avoids
journal replay. This branch is for the external notes drive, not Windows.                                                                                      Read the guide, using its actual filename:
                                                     ```sh                                                more '/media/notes/arch-linux-root-on-zfs-luks2-dual-boot(2).md'                                          ```                                                                                                       After section 6.12, `less FILE` is available: `/text` searches, `n` repeats a                             search, and `q` exits. A directly written ISO is read-only; do not expect to                              copy this guide into its ISO9660 filesystem afterward. Use another data drive                             or a Ventoy data partition for the Markdown file.    
### 6.5 Obtain initial tools from the ISO, before internet access
                                                     Extended includes a local package repository. A package can be on the ISO                                 without already being installed in the RAM-based live root. Inspect:
                                                     ```sh                                                cat /etc/apk/repositories                            ```                                                  
Then install Wi-Fi utilities using local sources only:
                                                     ```sh
apk --no-network add iw wpa_supplicant               ```

If this fails with a missing package or repository, locate the medium's index:                            
```sh
find /media -name APKINDEX.tar.gz                    ```
                                                     For example, if the result is                        `/media/sdb/apks/x86_64/APKINDEX.tar.gz`, the repository root is
`/media/sdb/apks`. Set the variable to **your actual result's repository root**:
                                                     ```sh
MEDIA_APKS='/media/sdb/apks'                         ls "${MEDIA_APKS}/x86_64/APKINDEX.tar.gz"
apk --no-network --repository "${MEDIA_APKS}" add \
  kbd-bkeymaps iw wpa_supplicant                     ```                                                  
The pathname above is an example, not a fixed device name. If no index is
found, inspect `mount` and the boot-medium errors from section 6.1. Do not
disable package-signature verification or use `--allow-untrusted` as a fix.
                                                     ### 6.6 Identify the network interface

```sh                                                ip link show
ls /sys/class/net                                    iw dev
```

Examples include `eth0`/`enp...` for Ethernet, `wlan0`/`wlp...` for Wi-Fi, and
`usb0`/`enx...` for USB tethering. Use the names actually shown. `lo` is the                              loopback interface and cannot connect you to the internet.                                                
Choose **one** connection method below. Once it works, skip the alternative
methods and continue at section 6.11. Do not start competing network managers                             or several DHCP clients on the same interface.
                                                     ### 6.7 Ethernet or Android USB tethering                                                                 For Ethernet, plug in the cable. For Android, connect a data-capable USB cable,                           unlock the phone, enable its internet connection, and enable **USB tethering**                            in Android's hotspot/tethering settings. Run `ip link show` again to find the                             new interface. Tethering depends on phone support and may use mobile data.                                
Start Alpine's guided configuration:
                                                     ```sh
setup-interfaces -r
```                                                  
Answer as follows; exact wording varies by release:  
| Prompt | Answer |                                  |---|---|
| Interface to initialize | Your connected Ethernet or USB interface |
| IPv4 address | `dhcp` |                            | IPv6 address, if asked | Accept `auto`, or choose `none` for an IPv4-only setup |                       | Another interface | `done` |                       | Manual network configuration | `n` |               
If the interface is still unconfigured, a temporary direct DHCP attempt is:

```sh                                                NET_IF='eth0'
ip link set dev "${NET_IF}" up                       udhcpc -i "${NET_IF}" -n -q -t 5 -T 3                ```                                                                                                       Replace `eth0` first. Use this fallback only if no DHCP client is already                                 handling that interface (`ps | grep '[u]dhcpc'` helps check). This one-shot                               command obtains a lease and exits; use the guided configuration for a longer                              session so a managed client renews the lease.                                                             ### 6.8 Wi-Fi through Alpine's guided setup                                                               After section 6.5, run:
                                                     ```sh
setup-interfaces -r                                  ```
                                                     | Prompt | Answer |
|---|---|                                            | Interface to initialize | The wireless interface shown by `iw dev` |
| Wireless network | Your exact SSID, or its listed number if offered |                                   | Pre-shared key | Your Wi-Fi password; it is not echoed |                                                | IPv4 address | `dhcp` |                            | IPv6 address, if asked | Accept `auto`, or select `none` if unused |
| Another interface | `done` |                       | Manual network configuration | `n` |
                                                     This configures `wpa_supplicant` and network addressing. Check the result:                                                                                     ```sh                                                ip addr show                                         ip route show
rc-service wpa_supplicant status                     rc-service networking status                         ```                                                  
An interface being `UP` does not prove authentication or internet access.
Look for an `inet` address on the chosen interface and a `default via` route.
If successful, skip section 6.9 and continue at section 6.11.
                                                     ### 6.9 Manual Wi-Fi fallback with wpa_supplicant                                                         Use this if the guided method fails. The recipe below is for an ordinary                                  **WPA2-Personal network with a passphrase**. WPA3-only and enterprise networks
such as eduroam require different authentication settings; use a suitable
hotspot/tether temporarily rather than guessing those credentials or policies.
                                                     Set the actual wireless interface:                                                                        ```sh
WIFI_IF='wlan0'
ip link show dev "${WIFI_IF}"                        ```
                                                     If the guided method configured this interface, bring that attempt down
before taking manual control:                        
```sh                                                ifdown "${WIFI_IF}"
rc-service wpa_supplicant stop                       ps | grep '[w]pa_supplicant'                         ps | grep '[u]dhcpc'                                 ```

An "interface not configured" or "service not started" message is acceptable                              here. If a previous manually started supplicant or DHCP client still owns this
same interface, stop that specific process before continuing. Leave unrelated
interfaces alone. Do these changes at the laptop console, not over a connection                           you are about to disconnect.
                                                     Bring up the radio and inspect visible SSIDs:
                                                     ```sh
ip link set dev "${WIFI_IF}" up
iw dev "${WIFI_IF}" scan | grep 'SSID:'              ```                                                                                                       If this reports `RF-kill`, use section 6.10. If the network is hidden, it will                            not necessarily appear in this list.
                                                     Read the SSID and password without putting the password into command history.                             `read -s` is supported by Alpine `ash` and Bash:                                                          ```sh                                                printf 'Wi-Fi network name: '                        IFS= read -r WIFI_SSID
printf 'Wi-Fi password (hidden): '                   IFS= read -r -s WIFI_PASS                            printf '\n'

umask 077                                            mkdir -p /etc/wpa_supplicant                         printf '%s\n' "${WIFI_PASS}" \
  | wpa_passphrase "${WIFI_SSID}" \                    > /etc/wpa_supplicant/live-network.tmp             unset WIFI_PASS                                      ```
                                                     If `wpa_passphrase` reports an error, stop and re-enter the credentials; do                               not continue with an empty or invalid configuration. A WPA2 passphrase normally
contains 8-63 characters. Now remove the generated plaintext-password comment                             and add the control socket and Danish radio region:                                                       ```sh
{                                                      printf '%s\n' 'ctrl_interface=/run/wpa_supplicant' 'country=DK'                                           sed '/^[[:space:]]*#psk=/d' /etc/wpa_supplicant/live-network.tmp                                        } > /etc/wpa_supplicant/live.conf                    rm /etc/wpa_supplicant/live-network.tmp
chmod 600 /etc/wpa_supplicant/live.conf              umask 022                                            ```                                                  
Use the country where the laptop is physically located if outside Denmark.                                The derived PSK in this file is still a usable credential; do not share it.                               For a hidden network, edit `live.conf` and add `scan_ssid=1` inside its                                   `network={ ... }` block.                                                                                  Start a single supplicant and check association:     
```sh                                                mkdir -p /run/wpa_supplicant                         wpa_supplicant -B -D nl80211 -i "${WIFI_IF}" \         -c /etc/wpa_supplicant/live.conf                   wpa_cli -p /run/wpa_supplicant -i "${WIFI_IF}" status```                                                                                                       Repeat the status command after a few seconds. Proceed only when it reports                               `wpa_state=COMPLETED`. `SCANNING`/`DISCONNECTED` means association is not done;                           repeated authentication failures often mean the SSID, key, or security mode is                            wrong. If it fails, stop this instance with
`wpa_cli -p /run/wpa_supplicant -i "${WIFI_IF}" terminate`, then rerun the                                same `wpa_supplicant` command **without `-B`** to see errors in the foreground.                           Press `Ctrl+C` to stop it before correcting settings and starting it again.                               
After association, request an IPv4 address:                                                               ```sh                                                udhcpc -i "${WIFI_IF}" -n -q -t 5 -T 3               ip addr show dev "${WIFI_IF}"
ip route show                                        ```                                                                                                       This deliberately separates Wi-Fi authentication from DHCP. `COMPLETED`                                   without an IP address points to addressing rather than a Wi-Fi-password issue.                            For a long installation, replace the exited one-shot DHCP client with a                                   renewing client in another console:                                                                       ```sh                                                udhcpc -i wlan0 -f                                   ```                                                                                                       Substitute your real interface again; variables do not transfer to that second
console. Leave it running there while working in the first console. Do not
also start a DHCP client managed by `setup-interfaces` for this interface.
                                                     ### 6.10 No Wi-Fi interface, blocked radio, or missing firmware
                                                     First inspect what is available without installing extra tools:
                                                     ```sh
ls /sys/class/net                                    cat /proc/net/wireless
dmesg | grep -iE 'firmware|wifi|wlan|80211|iwlwifi|rtw|ath|mt7'                                           ```

If the interface exists but cannot be brought up, check the laptop's airplane
mode key/switch. If `rfkill` is available:                                                                ```sh
rfkill list
rfkill unblock wifi                                  ```                                                                                                       A hardware block must be cleared with the physical switch or firmware setting.                            If `rfkill` is missing, try `apk --no-network add rfkill` from the medium.                                You can also inspect the radio-block state directly with this portable loop:                                                                                   ```sh
for radio in /sys/class/rfkill/rfkill*; do             test -d "${radio}" || continue                       cat "${radio}/name" "${radio}/soft" "${radio}/hard"done
```                                                  
The three lines per radio are its name, software block, and hardware block;                               `1` means blocked and `0` means unblocked.
                                                     For hardware identification, try packages on the medium:                                                  
```sh                                                apk --no-network add pciutils usbutils
lspci -nnk                                           lsusb                                                ```                                                  
The requested firmware filename and hardware ID identify the package/driver
needed. Installing `wpa_supplicant` cannot make an unsupported adapter appear.
Use Ethernet or USB tethering if necessary, then obtain the appropriate Alpine                            firmware package. Do not upgrade the live kernel with `apk upgrade` to solve                              this: the running kernel remains unchanged and the ISO's module set must match
it. A newer verified live image may be needed for newer hardware.

### 6.11 Check addressing, DNS, internet access, and clock separately                                     
```sh
ip addr show                                         ip route show
cat /etc/resolv.conf                                 ping -c 3 1.1.1.1
nslookup dl-cdn.alpinelinux.org                      date -u
```                                                  
| Result | What to inspect |                         |---|---|
| No address on the selected interface | Link/association, then DHCP |
| Address but no default route | DHCP/router settings |
| Public IP reachable, hostname lookup fails | DHCP-provided DNS in `/etc/resolv.conf` |                  | DNS works but downloads redirect or fail | Captive portal, proxy, TLS, or clock |                       | Ping fails but downloads succeed | The network may simply block ICMP |                                  
Use the DNS supplied by the network first. If it is absent and the network                                permits public DNS, a temporary live-session fallback is:                                                 
```sh                                                cp -p /etc/resolv.conf /etc/resolv.conf.before-manual-dns
printf 'nameserver 9.9.9.9\nnameserver 1.1.1.1\n' > /etc/resolv.conf                                      nslookup dl-cdn.alpinelinux.org
```                                                  
Do not repeatedly overwrite the backup. To restore it, use
`cp -p /etc/resolv.conf.before-manual-dns /etc/resolv.conf`. DHCP can subsequently
replace this file. Public DNS may not work before a captive portal is accepted.
                                                     Hospital/hotel Wi-Fi can associate successfully but still require a browser                               login. Another device's acceptance often does not authorize this laptop. For                              installation, Ethernet or permitted phone tethering is generally simpler than
bringing up a browser in the live console. Enterprise Wi-Fi also needs its                                institution-specific authentication configuration.   
Synchronize the clock once the connection is available:                                                                                                        ```sh
setup-ntp busybox
rc-service ntpd status
date -u                                              ```                                                  
Allow a short interval and check the time again. If NTP is blocked and the                                clock is far wrong, set the actual current UTC time using BusyBox's accepted                              % sed -n '500,1000p' alpine-zfs-guide.md
```text                                              altgr keycode 36 = less                              altgr keycode 37 = greater                           altgr keycode 38 = backslash                         control alt keycode 36 = less                        control alt keycode 37 = greater                     control alt keycode 38 = backslash
```

On the next empty line press **Ctrl+D** to finish. The bindings now apply to
the Linux virtual consoles. Run `cat` and test them with harmless text, then
press Ctrl+C. The mapping changes characters, not the contents of passwords.                              
On a normal PC keyboard, J/K/L have Linux keycodes 36/37/38. If this laptop
differs, run `showkey --keycodes`, press J, K, and L, then stop typing for about                          ten seconds to exit. Substitute the reported codes. This command requires the                             real virtual console, not an SSH pseudo-terminal.                                                         To undo the extra bindings in Alpine, run `setup-keymap dk dk` again. Doing so
also removes the temporary custom map. On Arch, `loadkeys dk` restores Danish.                            Do not replace a familiar working LUKS passphrase because the keyboard changed;
ensure that the same intended characters can be entered at unlock time.                                   
### 6.3.2 Prepare a reusable offline keymap for other live USB sessions                                                                                        Do this once in a working Linux console with `kbd` installed, **after** loading                           Danish and applying/testing the six lines above. It saves the complete active
console map, so the rest of the Danish layout is preserved:                                               
```sh                                                dumpkeys > /root/dk-heini.map
loadkeys --parse /root/dk-heini.map                  loadkeys -b /root/dk-heini.map > /root/dk-heini.bmap
```                                                  
The text `.map` is for `loadkeys`; the binary `.bmap` is for BusyBox `loadkmap`.                          The latter can be loaded even when the ISO lacks `kbd` or internet access.                                This kit is intended for the x86-64 Linux console used by this guide.                                                                                          Create a small loader beside the binary map:                                                              ```sh                                                cat > /root/tty-keys.sh <<'EOF'                      #!/bin/sh                                            # Load the neighbouring Danish console map with additional symbol bindings.
set -eu
case "$0" in
  */*) key_dir=${0%/*} ;;                              *) key_dir=. ;;                                    esac
key_file="${key_dir}/dk-heini.bmap"
test -r "${key_file}" || {                             echo 'Missing dk-heini.bmap next to this script.' >&2
  exit 1                                             }                                                    busybox loadkmap < "${key_file}"
echo 'Console keys loaded: AltGr+J = less; K = greater; L = backslash.'
EOF                                                  ```
                                                     Copy `tty-keys.sh` and `dk-heini.bmap` together onto an ordinary writable data                            USB or a Ventoy data partition. Keep the optional `.map` as the readable source.                          If `/media/notes` from section 6.4 is your verified external data partition,
make it writable for this copy only:

```sh
mount -o remount,rw /media/notes                     cp /root/tty-keys.sh /root/dk-heini.bmap /media/notes/                                                    sync
mount -o remount,ro /media/notes                     ```                                                  
On the next Alpine boot, mount that data partition as in section 6.4 and run:
                                                     ```sh
sh /media/notes/tty-keys.sh
```                                                  
That command contains none of `<`, `>`, `\`, or `|`, needs no executable file                             permission, and requires no download. It loads the prepared binary map into                               the running console. It is not automatically applied just because the files                               are on a USB. If no such kit was prepared and `kbd` is absent, the offline                                text-map method cannot be used; Ethernet/tethering or another prepared medium
is needed to obtain it.                                                                                   To use the same binary map for Alpine's loadkmap service in this live session:                                                                                 ```sh                                                mkdir -p /etc/keymap                                 cp /root/dk-heini.bmap /etc/keymap/dk-heini.bmap     printf 'KEYMAP=/etc/keymap/dk-heini.bmap\n' > /etc/conf.d/loadkmap
rc-service loadkmap restart                          ```                                                                                                       This service configuration is still in the live RAM filesystem and does not                               survive a normal ISO reboot. Use the USB kit again next time. Section 17.1                                provides the separate persistent configuration for installed Arch and its                                 LUKS-unlock initramfs.
                                                     
### 6.4 Find devices and open a USB containing this guide
                                                     Before installing additional tools, use:                                                                  ```sh
cat /proc/partitions                                 blkid                                                mount                                                ls /media
```                                                  
`mount` with no arguments lists current mounts. The Alpine boot medium is                                 usually already mounted somewhere under `/media`; reuse that mount. Keep it                               connected because its packages, modules, or firmware may still be needed.                                 
If `lsblk` exists, its view is easier to read:       
```sh                                                lsblk -o NAME,PATH,SIZE,MODEL,TRAN,FSTYPE,MOUNTPOINTS```                                                                                                       If it is missing, that is a package difference, not evidence that disks are                               missing. Section 6.12 installs the fuller toolset. Before then, inspecting                                `cat /sys/block/sdb/device/model` can help identify a particular USB disk.                                Device letters can change across boots; identify it again each time.
                                                     For an **ordinary data partition on another USB**, substitute its verified                                partition path below. Here `/dev/sdY1` is deliberately a placeholder:                                     
```sh                                                mkdir -p /media/notes
mount -o ro /dev/sdY1 /media/notes                   ls -la /media/notes                                  ```                                                                                                       This is a partition mount, unlike writing the ISO to a whole device. Do not                               use `/mnt` for notes; `/mnt` is reserved for the Arch root. If automatic type                             detection fails, use the type reported by `blkid`:   
| Reported type | Example read-only mount; choose only the matching row |                                 |---|---|                                            | `vfat` | `mount -t vfat -o ro /dev/sdY1 /media/notes` |                                                 | `exfat` | `mount -t exfat -o ro /dev/sdY1 /media/notes` |                                               | `ntfs` | Try `mount -t ntfs3 -o ro /dev/sdY1 /media/notes` if that driver is available. |               | `iso9660` | The flashed ISO; normally already mounted under `/media`. |
| `crypto_LUKS` | Encrypted container: unlock first; do not mount it directly. |                          | `zfs_member` | Pool member: import the pool and mount a dataset; see section 28. |

An unsupported-filesystem error needs the appropriate driver/helper, not
formatting. For an external NTFS notes drive, if `ntfs3` is unavailable,
install `ntfs-3g` once packages are reachable and use:
                                                     ```sh                                                apk add ntfs-3g
mount -t ntfs-3g -o ro /dev/sdY1 /media/notes        ```
                                                     For an ext4 notes partition, `mount -t ext4 -o ro,noload ...` also avoids
journal replay. This branch is for the external notes drive, not Windows.                                 
Read the guide, using its actual filename:           
```sh                                                more '/media/notes/arch-linux-root-on-zfs-luks2-dual-boot(2).md'                                          ```

After section 6.12, `less FILE` is available: `/text` searches, `n` repeats a                             search, and `q` exits. A directly written ISO is read-only; do not expect to                              copy this guide into its ISO9660 filesystem afterward. Use another data drive                             or a Ventoy data partition for the Markdown file.
                                                     ### 6.5 Obtain initial tools from the ISO, before internet access                                                                                              Extended includes a local package repository. A package can be on the ISO                                 without already being installed in the RAM-based live root. Inspect:

```sh                                                cat /etc/apk/repositories
```                                                  
Then install Wi-Fi utilities using local sources only:
                                                     ```sh
apk --no-network add iw wpa_supplicant               ```
                                                     If this fails with a missing package or repository, locate the medium's index:                            
```sh
find /media -name APKINDEX.tar.gz                    ```
                                                     For example, if the result is                        `/media/sdb/apks/x86_64/APKINDEX.tar.gz`, the repository root is
`/media/sdb/apks`. Set the variable to **your actual result's repository root**:
                                                     ```sh
MEDIA_APKS='/media/sdb/apks'
ls "${MEDIA_APKS}/x86_64/APKINDEX.tar.gz"            apk --no-network --repository "${MEDIA_APKS}" add \
  kbd-bkeymaps iw wpa_supplicant                     ```                                                  
The pathname above is an example, not a fixed device name. If no index is                                 found, inspect `mount` and the boot-medium errors from section 6.1. Do not                                disable package-signature verification or use `--allow-untrusted` as a fix.                                                                                    ### 6.6 Identify the network interface               
```sh                                                ip link show
ls /sys/class/net                                    iw dev                                               ```
                                                     Examples include `eth0`/`enp...` for Ethernet, `wlan0`/`wlp...` for Wi-Fi, and                            `usb0`/`enx...` for USB tethering. Use the names actually shown. `lo` is the                              loopback interface and cannot connect you to the internet.
                                                     Choose **one** connection method below. Once it works, skip the alternative                               methods and continue at section 6.11. Do not start competing network managers                             or several DHCP clients on the same interface.                                                            ### 6.7 Ethernet or Android USB tethering
                                                     For Ethernet, plug in the cable. For Android, connect a data-capable USB cable,                           unlock the phone, enable its internet connection, and enable **USB tethering**                            in Android's hotspot/tethering settings. Run `ip link show` again to find the                             new interface. Tethering depends on phone support and may use mobile data.                                                                                     Start Alpine's guided configuration:                                                                      ```sh                                                setup-interfaces -r
```                                                  
Answer as follows; exact wording varies by release:  
| Prompt | Answer |                                  |---|---|                                            | Interface to initialize | Your connected Ethernet or USB interface |                                    | IPv4 address | `dhcp` |
| IPv6 address, if asked | Accept `auto`, or choose `none` for an IPv4-only setup |
| Another interface | `done` |                       | Manual network configuration | `n` |
                                                     If the interface is still unconfigured, a temporary direct DHCP attempt is:                                                                                    ```sh                                                NET_IF='eth0'                                        ip link set dev "${NET_IF}" up                       udhcpc -i "${NET_IF}" -n -q -t 5 -T 3
```                                                  
Replace `eth0` first. Use this fallback only if no DHCP client is already
handling that interface (`ps | grep '[u]dhcpc'` helps check). This one-shot
command obtains a lease and exits; use the guided configuration for a longer                              session so a managed client renews the lease.        
### 6.8 Wi-Fi through Alpine's guided setup                                                               After section 6.5, run:                                                                                   ```sh                                                setup-interfaces -r                                  ```                                                                                                       | Prompt | Answer |
|---|---|                                            | Interface to initialize | The wireless interface shown by `iw dev` |                                    | Wireless network | Your exact SSID, or its listed number if offered |                                   | Pre-shared key | Your Wi-Fi password; it is not echoed |                                                | IPv4 address | `dhcp` |
| IPv6 address, if asked | Accept `auto`, or select `none` if unused |
| Another interface | `done` |                       | Manual network configuration | `n` |
                                                     This configures `wpa_supplicant` and network addressing. Check the result:                                
```sh
ip addr show
ip route show                                        rc-service wpa_supplicant status
rc-service networking status                         ```
                                                     An interface being `UP` does not prove authentication or internet access.
Look for an `inet` address on the chosen interface and a `default via` route.
If successful, skip section 6.9 and continue at section 6.11.
                                                     ### 6.9 Manual Wi-Fi fallback with wpa_supplicant
                                                     Use this if the guided method fails. The recipe below is for an ordinary                                  **WPA2-Personal network with a passphrase**. WPA3-only and enterprise networks                            such as eduroam require different authentication settings; use a suitable                                 hotspot/tether temporarily rather than guessing those credentials or policies.
                                                     Set the actual wireless interface:

```sh                                                WIFI_IF='wlan0'                                      ip link show dev "${WIFI_IF}"
```                                                  
If the guided method configured this interface, bring that attempt down
before taking manual control:                                                                             ```sh                                                ifdown "${WIFI_IF}"                                  rc-service wpa_supplicant stop                       ps | grep '[w]pa_supplicant'
ps | grep '[u]dhcpc'                                 ```                                                                                                       An "interface not configured" or "service not started" message is acceptable
here. If a previous manually started supplicant or DHCP client still owns this
same interface, stop that specific process before continuing. Leave unrelated
interfaces alone. Do these changes at the laptop console, not over a connection
you are about to disconnect.                         
Bring up the radio and inspect visible SSIDs:                                                             ```sh                                                ip link set dev "${WIFI_IF}" up                      iw dev "${WIFI_IF}" scan | grep 'SSID:'              ```
                                                     If this reports `RF-kill`, use section 6.10. If the network is hidden, it will                            not necessarily appear in this list.                                                                      Read the SSID and password without putting the password into command history.                             `read -s` is supported by Alpine `ash` and Bash:                                                          ```sh                                                printf 'Wi-Fi network name: '                        IFS= read -r WIFI_SSID                               printf 'Wi-Fi password (hidden): '                   IFS= read -r -s WIFI_PASS                            printf '\n'                                                                                               umask 077                                            mkdir -p /etc/wpa_supplicant                         printf '%s\n' "${WIFI_PASS}" \                         | wpa_passphrase "${WIFI_SSID}" \                    > /etc/wpa_supplicant/live-network.tmp             unset WIFI_PASS                                      ```                                                                                                       If `wpa_passphrase` reports an error, stop and re-enter the credentials; do                               not continue with an empty or invalid configuration. A WPA2 passphrase normally                           contains 8-63 characters. Now remove the generated plaintext-password comment                             and add the control socket and Danish radio region:                                                       ```sh                                                {                                                      printf '%s\n' 'ctrl_interface=/run/wpa_supplicant' 'country=DK'                                           sed '/^[[:space:]]*#psk=/d' /etc/wpa_supplicant/live-network.tmp                                        } > /etc/wpa_supplicant/live.conf                    rm /etc/wpa_supplicant/live-network.tmp              chmod 600 /etc/wpa_supplicant/live.conf              umask 022                                            ```                                                                                                       Use the country where the laptop is physically located if outside Denmark.                                The derived PSK in this file is still a usable credential; do not share it.                               For a hidden network, edit `live.conf` and add `scan_ssid=1` inside its                                   `network={ ... }` block.                                                                                  Start a single supplicant and check association:
                                                     ```sh                                                mkdir -p /run/wpa_supplicant                         wpa_supplicant -B -D nl80211 -i "${WIFI_IF}" \         -c /etc/wpa_supplicant/live.conf                   wpa_cli -p /run/wpa_supplicant -i "${WIFI_IF}" status```                                                                                                       Repeat the status command after a few seconds. Proceed only when it reports                               `wpa_state=COMPLETED`. `SCANNING`/`DISCONNECTED` means association is not done;                           repeated authentication failures often mean the SSID, key, or security mode is                            wrong. If it fails, stop this instance with          `wpa_cli -p /run/wpa_supplicant -i "${WIFI_IF}" terminate`, then rerun the                                same `wpa_supplicant` command **without `-B`** to see errors in the foreground.                           Press `Ctrl+C` to stop it before correcting settings and starting it again.                                                                                    After association, request an IPv4 address:                                                               ```sh                                                udhcpc -i "${WIFI_IF}" -n -q -t 5 -T 3               ip addr show dev "${WIFI_IF}"                        ip route show                                        ```                                                                                                       This deliberately separates Wi-Fi authentication from DHCP. `COMPLETED`                                   without an IP address points to addressing rather than a Wi-Fi-password issue.                            For a long installation, replace the exited one-shot DHCP client with a                                   renewing client in another console:

```sh                                                udhcpc -i wlan0 -f
```                                                                                                       Substitute your real interface again; variables do not transfer to that second                            console. Leave it running there while working in the first console. Do not                                also start a DHCP client managed by `setup-interfaces` for this interface.

### 6.10 No Wi-Fi interface, blocked radio, or missing firmware

First inspect what is available without installing extra tools:

```sh
ls /sys/class/net
cat /proc/net/wireless
dmesg | grep -iE 'firmware|wifi|wlan|80211|iwlwifi|rtw|ath|mt7'
```

If the interface exists but cannot be brought up, check the laptop's airplane
mode key/switch. If `rfkill` is available:

```sh
rfkill list
rfkill unblock wifi
```

A hardware block must be cleared with the physical switch or firmware setting.
If `rfkill` is missing, try `apk --no-network add rfkill` from the medium.
You can also inspect the radio-block state directly with this portable loop:

```sh
for radio in /sys/class/rfkill/rfkill*; do
  test -d "${radio}" || continue
  cat "${radio}/name" "${radio}/soft" "${radio}/hard"
done
```

The three lines per radio are its name, software block, and hardware block;
`1` means blocked and `0` means unblocked.

For hardware identification, try packages on the medium:

```sh
apk --no-network add pciutils usbutils
lspci -nnk
lsusb
```

The requested firmware filename and hardware ID identify the package/driver
needed. Installing `wpa_supplicant` cannot make an unsupported adapter appear.
Use Ethernet or USB tethering if necessary, then obtain the appropriate Alpine
firmware package. Do not upgrade the live kernel with `apk upgrade` to solve
this: the running kernel remains unchanged and the ISO's module set must match
it. A newer verified live image may be needed for newer hardware.

### 6.11 Check addressing, DNS, internet access, and clock separately

```sh
ip addr show
ip route show
cat /etc/resolv.conf
ping -c 3 1.1.1.1
nslookup dl-cdn.alpinelinux.org
date -u
```

| Result | What to inspect |
|---|---|
| No address on the selected interface | Link/association, then DHCP |
| Address but no default route | DHCP/router settings |
| Public IP reachable, hostname lookup fails | DHCP-provided DNS in `/etc/resolv.conf` |
| DNS works but downloads redirect or fail | Captive portal, proxy, TLS, or clock |
| Ping fails but downloads succeed | The network may simply block ICMP |

Use the DNS supplied by the network first. If it is absent and the network
permits public DNS, a temporary live-session fallback is:

```sh
cp -p /etc/resolv.conf /etc/resolv.conf.before-manual-dns
printf 'nameserver 9.9.9.9\nnameserver 1.1.1.1\n' > /etc/resolv.conf
nslookup dl-cdn.alpinelinux.org
```

Do not repeatedly overwrite the backup. To restore it, use
`cp -p /etc/resolv.conf.before-manual-dns /etc/resolv.conf`. DHCP can subsequently
replace this file. Public DNS may not work before a captive portal is accepted.

Hospital/hotel Wi-Fi can associate successfully but still require a browser
login. Another device's acceptance often does not authorize this laptop. For
installation, Ethernet or permitted phone tethering is generally simpler than
bringing up a browser in the live console. Enterprise Wi-Fi also needs its
institution-specific authentication configuration.

Synchronize the clock once the connection is available:

```sh
setup-ntp busybox
rc-service ntpd status
date -u
```

Allow a short interval and check the time again. If NTP is blocked and the
clock is far wrong, set the actual current UTC time using BusyBox's accepted
numeric format. For example, **replace this sample with the real time**:

```sh
busybox date -u -s '2026.10.15-02:15:00'
```

The numeric format is `YYYY.MM.DD-hh:mm:ss`. Keep certificate verification
enabled; fix the clock or connection rather than using insecure download flags.

### 6.12 Enable repositories and install the complete live toolset

Select a mirror:

```sh
setup-apkrepos
cat /etc/apk/repositories
```

Choose an official HTTPS mirror; enable `community` if needed. Keep remote
repositories on the same release branch as the live system, and retain the
local medium's repository. If the chooser is inconvenient, append a matching
official CDN pair after confirming they are not already listed:

```sh
ALPINE_BRANCH="v$(cut -d. -f1,2 /etc/alpine-release)"
ALPINE_REPO="https://dl-cdn.alpinelinux.org/alpine/${ALPINE_BRANCH}"
printf '%s\n' "${ALPINE_REPO}/main" "${ALPINE_REPO}/community" \
  >> /etc/apk/repositories
apk update
```

`apk update` refreshes package indexes; it does not upgrade the kernel. If it
fails, resolve the reported network, DNS, clock, or repository error before
continuing. Do not mix `edge` with stable repositories.

Install the utilities used later in this runbook:

```sh
apk add \
  bash \
  ca-certificates \
  coreutils \
  cryptsetup \
  curl \
  dosfstools \
  e2fsprogs \
  eudev \
  gnupg \
  gptfdisk \
  iw \
  kbd \
  less \
  nano \
  parted \
  pciutils \
  rfkill \
  tar \
  usbutils \
  util-linux \
  wpa_supplicant \
  zfs \
  zstd
```

Optional: switch this live shell to Bash now, before defining the disk and
partition variables in section 7 onward:

```sh
exec bash -l
```

This is only a shell change. It neither installs Arch nor changes the running
kernel. The later `sh` blocks also work in Bash. Do not switch shells halfway
through a procedure without exporting or re-establishing its variables.

`nano FILE` now offers a simpler editor: `Ctrl+O`, Enter saves, and `Ctrl+X`
exits. Use `less FILE` to read without editing.

### 6.13 Initialize devices and verify UEFI and ZFS

The guide uses udev-generated `/dev/disk/by-*` links. Start it now:

```sh
setup-devd udev
udevadm settle
ls /dev/disk/by-id
ip link show
```                                                                                                       Check interface names again after changing device management; if a connection                             was disrupted or an interface renamed, repeat the relevant networking step                                with its current name.                                                                                    Verify UEFI boot and access to its variables:                                                             ```sh                                                if test -d /sys/firmware/efi; then                     echo 'UEFI boot detected.'                         else                                                   echo 'STOP: Reboot the USB using its UEFI entry.'  fi                                                   ```                                                                                                       If the directory is absent, stop here and reboot correctly. If it exists,                                 ensure `efivarfs` is mounted:                                                                             ```sh                                                mkdir -p /sys/firmware/efi/efivars                   if ! mountpoint -q /sys/firmware/efi/efivars; then     mount -t efivarfs efivarfs /sys/firmware/efi/efivars                                                    fi                                                   findmnt /sys/firmware/efi/efivars                    ```                                                                                                       Load ZFS for the **running Alpine kernel**:                                                               ```sh                                                uname -r                                             modprobe zfs                                         zfs version                                          lsblk -o NAME,PATH,SIZE,MODEL,TRAN,FSTYPE,MOUNTPOINTS```                                                                                                       If `modprobe` fails, stop and inspect:                                                                    ```sh                                                ls /lib/modules                                      ls /lib/modules/"$(uname -r)"                        rc-service modloop status                            dmesg | tail -n 50                                   ```                                                                                                       If the live module-image service is stopped while the boot USB is present,                                try `rc-service modloop start`, then `modprobe zfs` again. An invalid module
format requires matching the live kernel and module; installing a random                                  current `zfs-lts` package or entering an Arch chroot cannot replace that kernel.                          Record the exact error if it persists.                                                                    Before section 7, confirm: the keyboard is correct, internet/package downloads                            work, UEFI is detected, ZFS loads, and the disks are identifiable. If an Arch                             installation or its ZFS pool already exists, **use section 28 to reopen it**;                             do not repeat partition creation, `luksFormat`, `mkfs`, or `zpool create`.                                

## 7. Inventory the disk: mandatory stop gate
                                                     List every disk, partition, filesystem, UUID, and mount point:
                                                     ```sh
lsblk --bytes \
  --output NAME,PATH,SIZE,TYPE,FSTYPE,FSVER,LABEL,PARTLABEL,PARTUUID,MOUNTPOINTS
blkid                                                find /dev/disk/by-id -maxdepth 1 -type l -print
```                                                                                                       Choose the persistent whole-disk path. Example only:

```sh
DISK='/dev/disk/by-id/nvme-Samsung_SSD_EXAMPLE_SERIAL'                                                    ```

Verify that it resolves to the intended laptop disk:

```sh
readlink -f "${DISK}"
lsblk --output NAME,PATH,SIZE,MODEL,SERIAL,FSTYPE,PARTLABEL "${DISK}"                                     sgdisk --print "${DISK}"                             ```                                                                                                       **Stop unless all of the following are true:**                                                            - `DISK` is the intended internal drive.             - The Windows EFI, MSR, NTFS, and recovery partitions are visible.                                        - The intended capacity appears as unallocated/free space.                                                - No existing partition occupies the space intended for Arch.                                                                                                  Do not run `wipefs`, `sgdisk --zap-all`, `mklabel`, `blkdiscard`, or any command                          that recreates the disk partition table. Those commands would destroy the                                 dual-boot installation.                              
## 8. Create only the new Arch partitions
                                                     Start the interactive GPT editor:

```sh
cgdisk "${DISK}"
```

### Recommended branch: dedicated Arch ESP

Within the unallocated region only:

1. Select **Free space**.                            2. Select **New**.
3. Accept the proposed first sector unless alignment is reported as invalid.
4. Enter `1G` as size (`cgdisk` uses `K`, `M`, `G`, `T`, and `P` suffixes).
5. Enter type code `EF00`.
6. Enter partition name `ARCH_EFI`.
7. Select the remaining free space and create another partition.
8. Use the remaining desired size.                   9. Enter type code `8309` (Linux LUKS).
10. Enter partition name `ARCH_CRYPT`.
11. Select **Write**, type `yes`, and then **Quit**.
                                                     ### Alternative branch: share an adequately sized Windows ESP                                             
Create only the `ARCH_CRYPT` partition in free space using type `8309`. Do not                            alter, rename, or format the Windows ESP. Later, set `ESP_PART` to that existing
ESP's `/dev/disk/by-partuuid/...` path.

Reload and inspect the table:

```sh
partprobe "${DISK}"
udevadm settle
lsblk --output NAME,PATH,SIZE,FSTYPE,PARTTYPE,PARTLABEL,PARTUUID "${DISK}"
sgdisk --print "${DISK}"
```                                                                                                       Set stable partition paths.                                                                               For the recommended dedicated-ESP branch:                                                                 ```sh                                                ESP_PART='/dev/disk/by-partlabel/ARCH_EFI'           CRYPT_PART='/dev/disk/by-partlabel/ARCH_CRYPT'       ```                                                                                                       For the shared-ESP branch, replace the first value with the existing Windows                              ESP's exact PARTUUID path:                                                                                ```sh                                                ESP_PART='/dev/disk/by-partuuid/REPLACE-WITH-WINDOWS-ESP-PARTUUID'                                        CRYPT_PART='/dev/disk/by-partlabel/ARCH_CRYPT'       ```                                                                                                       Validate both variables:                                                                                  ```sh                                                for part in "${ESP_PART}" "${CRYPT_PART}"; do          test -b "${part}" || {                                 echo "ERROR: Not a block device: ${part}" >&2        exit 1                                             }                                                    readlink -f "${part}"                                lsblk --output NAME,PATH,SIZE,FSTYPE,PARTLABEL,PARTUUID "${part}"                                       done                                                 ```                                                                                                       The output for `CRYPT_PART` must say `ARCH_CRYPT` and must not contain NTFS or                            Windows data. The ESP must be the new `ARCH_EFI` partition or the deliberately                            selected existing FAT32 ESP.                                                                              ## 9. Format or mount the ESP                                                                             Run **only one** branch.                                                                                  For a newly created `ARCH_EFI` partition:                                                                 ```sh                                                mkfs.fat -F 32 -n ARCH_EFI "${ESP_PART}"             ```                                                                                                       For a shared Windows ESP, do not format it. Verify that it is FAT32:
                                                     ```sh                                                blkid "${ESP_PART}"                                                                                       mkdir -p /tmp/esp-check                              mount -o ro "${ESP_PART}" /tmp/esp-check             df -h /tmp/esp-check                                 test -f /tmp/esp-check/EFI/Microsoft/Boot/bootmgfw.efi                                                    umount /tmp/esp-check                                rmdir /tmp/esp-check                                 ```                                                                                                       Expected filesystem type: `TYPE="vfat"`. Confirm that the filesystem is at
least 512 MiB with at least 250 MiB available. If either condition fails, use a
dedicated Arch ESP instead.                          
## 10. Create the LUKS2 container
                                                     Inspect the target one last time:

```sh
lsblk --output NAME,PATH,SIZE,FSTYPE,PARTLABEL,PARTUUID "${CRYPT_PART}"                                   wipefs --no-act "${CRYPT_PART}"
```                                                                                                       The next command destroys all pre-existing data in `CRYPT_PART`. It must not be
an NTFS, recovery, or EFI partition.

```sh                                                cryptsetup luksFormat \
  --type luks2 \
  --pbkdf argon2id \
  --label cryptzfs \
  "${CRYPT_PART}"
```

Use a strong, unique passphrase. Open and inspect it:                                                     ```sh                                                cryptsetup open "${CRYPT_PART}" cryptzfs             cryptsetup status cryptzfs                           cryptsetup luksDump "${CRYPT_PART}"                  lsblk --output NAME,PATH,SIZE,TYPE,FSTYPE,LABEL "${CRYPT_PART}"                                           ```                                                                                                       Expected stack:                                                                                           ```text                                              ARCH_CRYPT partition                                 └─cryptzfs  crypt                                    ```                                                                                                       Do not enable discard/TRIM yet. Discard reveals allocation patterns through the                           encryption layer. It can be enabled later after consciously accepting that
leakage.                                             
## 11. Create the ZFS pool

Confirm that the compatibility profile exists:

```sh
test -f /usr/share/zfs/compatibility.d/openzfs-2.3-linux || {
  echo 'Available ZFS compatibility profiles:' >&2
  ls -1 /usr/share/zfs/compatibility.d >&2
  echo 'ERROR: Required compatibility profile is absent.' >&2
  exit 1                                             }
```

Create a single-vdev pool on the LUKS mapper:

```sh
zpool create -f \
  -o ashift=12 \                                       -o autotrim=off \
  -o cachefile=none \
  -o compatibility=openzfs-2.3-linux \
  -R /mnt \                                            -O acltype=posixacl \
  -O canmount=off \                                    -O compression=lz4 \
  -O dnodesize=auto \
  -O normalization=formD \                             -O relatime=on \
  -O xattr=sa \
  -O mountpoint=none \
  rpool \
  /dev/mapper/cryptzfs
```

Why these properties matter:

- `ashift=12` aligns writes to 4 KiB sectors and cannot be reduced later.
- `compatibility=openzfs-2.3-linux` avoids accidentally enabling a new pool                                 feature that the installed Arch ZFS module cannot import.                                               - `compression=lz4` is mature, low-overhead, and generally beneficial.                                    - `xattr=sa` and `acltype=posixacl` are suitable for Linux metadata and ACLs.                             - `normalization=formD` prevents certain Unicode-equivalent duplicate names.                              - `cachefile=none` makes early boot scan `/dev/mapper` explicitly rather than                               depending on a cache created by the temporary live environment.                                                                                              Do not enable deduplication. It is not an ordinary space-saving switch and can                            create severe memory and performance costs.                                                               ## 12. Create datasets and mount them                                                                     The root operating system is kept separate from home and volatile data so that                            root snapshots do not also roll back personal data, logs, or package cache.                               `/var/lib`, including the pacman database, deliberately remains in the root                               dataset so package state and system files roll back together.                                                                                                  ```sh                                                zfs create -o canmount=off -o mountpoint=none rpool/ROOT                                                  zfs create \                                           -o canmount=noauto \                                 -o mountpoint=legacy \                               rpool/ROOT/arch                                                                                         zfs create -o canmount=off -o mountpoint=none rpool/DATA                                                  zfs create -o mountpoint=legacy rpool/DATA/home      zfs create -o mountpoint=legacy rpool/DATA/var-log   zfs create -o mountpoint=legacy rpool/DATA/var-cache zfs create -o mountpoint=legacy rpool/DATA/var-tmp                                                        zpool set bootfs=rpool/ROOT/arch rpool               ```                                                                                                       Mount the datasets:                                                                                       ```sh                                                mkdir -p /mnt                                        mount -t zfs rpool/ROOT/arch /mnt

mkdir -p \                                             /mnt/home \                                          /mnt/var/log \                                       /mnt/var/cache \                                     /mnt/var/tmp \                                       /mnt/boot                                                                                               mount -t zfs rpool/DATA/home /mnt/home               mount -t zfs rpool/DATA/var-log /mnt/var/log         mount -t zfs rpool/DATA/var-cache /mnt/var/cache     mount -t zfs rpool/DATA/var-tmp /mnt/var/tmp         mount -t vfat -o umask=0077 "${ESP_PART}" /mnt/boot  chmod 1777 /mnt/var/tmp                              ```                                                  
Verify every mount:

```sh                                                findmnt --real
zpool status -v
zpool get ashift,autotrim,compatibility,bootfs rpool zfs list -o name,used,avail,refer,mountpoint,canmount
```

## 13. Download and verify the Arch bootstrap tarball
                                                     The release pinned when this runbook was written is Arch 2026.08.01.                                                                                           ```sh
ARCH_RELEASE='2026.08.01'
ARCH_ARCHIVE="https://archive.archlinux.org/iso/${ARCH_RELEASE}"
ARCH_TARBALL="archlinux-bootstrap-${ARCH_RELEASE}-x86_64.tar.zst"

cd /tmp
curl --fail --location \
  "${ARCH_ARCHIVE}/${ARCH_TARBALL}" \
  --output "${ARCH_TARBALL}"
curl --fail --location \
  "${ARCH_ARCHIVE}/${ARCH_TARBALL}.sig" \              --output "${ARCH_TARBALL}.sig"                                                                          gpg --auto-key-locate clear,wkd \                      --locate-external-key pierre@archlinux.org         gpg --verify "${ARCH_TARBALL}.sig" "${ARCH_TARBALL}" ```                                                                                                       Do not continue unless GnuPG reports a good signature from the Arch release                               signing key. Confirm that `/mnt` is the mounted Arch root dataset. Using the                              GNU `tar` installed in section 6, strip the archive's `root.x86_64/` prefix                               and extract directly there. Run this only for a new, unpopulated installation:                            do not re-extract the bootstrap over an existing Arch system.                                                                                                  ```sh
findmnt --mountpoint /mnt
tar --zstd --extract \                                 --file="${ARCH_TARBALL}" \
  --directory=/mnt \
  --strip-components=1 \
  --numeric-owner
```

Confirm the target resembles an Arch root filesystem:

```sh
test -x /mnt/usr/bin/pacman
test -x /mnt/usr/bin/bash                            ls -la /mnt
```                                                  
## 14. Prepare and enter the Arch chroot

Before binding anything, check `findmnt -R /mnt`: `/mnt` must be the root
dataset and `/mnt/boot` the intended ESP. Use this once for a fresh install.
For an already installed system or a later recovery boot, use section 28.

The `--rbind`/`--make-rslave` options below need the full mount utility from                              section 6. `mount --version` should identify util-linux.                                                  
```sh
cp -L /etc/resolv.conf /mnt/etc/resolv.conf          
for path in dev proc sys run; do
  mount --rbind "/${path}" "/mnt/${path}"              mount --make-rslave "/mnt/${path}"                 done
                                                     chroot /mnt /usr/bin/env -i \                          ESP_PART="${ESP_PART}" \
  CRYPT_PART="${CRYPT_PART}" \                         HOME=/root \                                         TERM="${TERM:-linux}" \                              PATH=/usr/local/sbin:/usr/local/bin:/usr/bin \       /bin/bash --login
```                                                  
The prompt is now inside the new Arch installation and its Bash shell.
`cat /etc/os-release` identifies Arch; `uname -r` still identifies the live                               Alpine kernel. Use `pacman` for Arch packages here, not `apk`.                                                                                                 Verify the two partition                             variables survived the chroot boundary:                                                                   ```bash                                              printf 'ESP_PART=%s\nCRYPT_PART=%s\n' "${ESP_PART}" "${CRYPT_PART}"                                       test -b "${ESP_PART}"                                test -b "${CRYPT_PART}"                              ```                                                                                                       ## 15. Initialize pacman and add the signed ArchZFS repository                                            
Configure Arch's official geographic mirror, initialize the keyring, and fully
upgrade the bootstrap environment. The full `-Syu` is intentional: Arch does
not support partial upgrades.

```bash
tee /etc/pacman.d/mirrorlist >/dev/null <<'EOF'
Server = https://geo.mirror.pkgbuild.com/$repo/os/$arch
EOF

pacman-key --init
pacman-key --populate archlinux
pacman -Syu --noconfirm archlinux-keyring
pacman-key --populate archlinux
```
                                                     Import the current ArchZFS signing key and locally certify the exact
fingerprint published by ArchZFS:
                                                     ```bash
ARCHZFS_KEY='3A9917BF0DED5C13F69AC68FABEC0A1208037BE9'

pacman-key --recv-keys "${ARCHZFS_KEY}"
pacman-key --finger "${ARCHZFS_KEY}"                 pacman-key --lsign-key "${ARCHZFS_KEY}"
```
                                                     The displayed fingerprint must exactly equal the value assigned above. Append                             the repository using required package signatures:
                                                     ```bash
tee -a /etc/pacman.conf >/dev/null <<'EOF'

[archzfs]                                            SigLevel = Required
Server = https://github.com/archzfs/archzfs/releases/download/experimental
EOF                                                  
pacman -Syy                                          ```

Do not weaken this to `SigLevel = Never`.            
## 16. Install the base system and a matching LTS kernel/ZFS pair

First install packages that do not require the kernel pair:
                                                     ```bash
pacman -S --noconfirm --needed \
  base \
  base-devel \
  cryptsetup \
  efibootmgr \
  git \
  kbd \                                                linux-firmware \
  man-db \
  man-pages \
  mkinitcpio \
  networkmanager \
  refind \
  sudo \
  texinfo \                                            vim \
  wireless-regdb \
  zram-generator
```

Install the correct CPU microcode package:           
```bash                                              if grep -q AuthenticAMD /proc/cpuinfo; then
  pacman -S --noconfirm --needed amd-ucode           elif grep -q GenuineIntel /proc/cpuinfo; then
  pacman -S --noconfirm --needed intel-ucode         else
  echo 'WARNING: CPU vendor not recognized; install microcode manually.' >&2
fi                                                   ```
                                                     Query the exact kernel version required by the current precompiled ZFS module:
                                                     ```bash
ZFS_KERNEL_PACKAGE='zfs-linux-lts'                   LINUX_LTS_VERSION="$(
  LC_ALL=C pacman -Si "${ZFS_KERNEL_PACKAGE}" \          | sed -n \
      's/^Depends On[[:space:]]*:.*linux-lts=\([^ ]*\).*/\1/p'                                            )"
                                                     test -n "${LINUX_LTS_VERSION}" || {
  echo 'ERROR: Could not derive the required linux-lts version.' >&2
  pacman -Si "${ZFS_KERNEL_PACKAGE}" >&2
  exit 1                                             }
                                                     printf 'ArchZFS requires linux-lts %s\n' "${LINUX_LTS_VERSION}"
```

Install that exact kernel from the Arch Linux Archive, then the matching ZFS                              module and utilities:

```bash                                              LINUX_LTS_URL='https://archive.archlinux.org/packages/l/linux-lts'
LINUX_LTS_URL="${LINUX_LTS_URL}/linux-lts-${LINUX_LTS_VERSION}"
LINUX_LTS_URL="${LINUX_LTS_URL}-x86_64.pkg.tar.zst"  
pacman -U --noconfirm "${LINUX_LTS_URL}"
pacman -S --noconfirm --needed \                       "${ZFS_KERNEL_PACKAGE}" \
  zfs-utils                                          ```
                                                     Verify package relationships and that a ZFS kernel module exists:
                                                     ```bash
pacman -Q linux-lts zfs-linux-lts zfs-utils          pacman -Qi zfs-linux-lts | sed -n '/^Depends On/p'
                                                     KERNEL_RELEASE="$(
  find /usr/lib/modules \                                -mindepth 1 \
    -maxdepth 1 \
    -type d \
    -name '*-lts' \
    -printf '%f\n' \                                     | sort -V \
    | tail -n 1                                      )"
                                                     test -n "${KERNEL_RELEASE}"
find "/usr/lib/modules/${KERNEL_RELEASE}" -name 'zfs.ko*' -print
```

At least one `zfs.ko` or compressed equivalent must be printed.
                                                     ## 17. Configure timezone, locale, console, hostname, and users                                           
The following uses Danish keyboard layout and Copenhagen time while retaining
an English Denmark locale:

```bash
ln -sf /usr/share/zoneinfo/Europe/Copenhagen /etc/localtime
hwclock --systohc

sed -i 's/^#en_DK.UTF-8 UTF-8/en_DK.UTF-8 UTF-8/' /etc/locale.gen
sed -i 's/^#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
                                                     printf 'LANG=en_DK.UTF-8\n' > /etc/locale.conf
printf 'KEYMAP=dk\n' > /etc/vconsole.conf
printf 'archzfs\n' > /etc/hostname

tee /etc/hosts >/dev/null <<'EOF'                    127.0.0.1 localhost
::1       localhost
127.0.1.1 archzfs.localdomain archzfs                EOF
```                                                  
Set the root password, create the normal user, and configure `sudo`:

```bash
passwd

useradd --create-home --groups wheel --shell /bin/bash heini
passwd heini

install -m 0440 /dev/null /etc/sudoers.d/10-wheel    printf '%%wheel ALL=(ALL:ALL) ALL\n' > /etc/sudoers.d/10-wheel
visudo --check --file=/etc/sudoers.d/10-wheel
```

Replace `archzfs` or `heini` before running the commands if different names are
desired.                                             
### 17.1 Preserve the symbol bindings in Arch and at LUKS unlock                                          
Run this **inside the Arch chroot**, after its `kbd` package is installed.
The following text keymap includes the normal Danish layout and overrides the
same six combinations from section 6.3.1:            
```bash
pacman -S --needed kbd
mkdir -p /etc/kbd
cat > /etc/kbd/dk-heini.map <<'EOF'                  include "dk.map"
altgr keycode 36 = less
altgr keycode 37 = greater                           altgr keycode 38 = backslash
control alt keycode 36 = less                        control alt keycode 37 = greater
control alt keycode 38 = backslash                   EOF                                                  
loadkeys --parse /etc/kbd/dk-heini.map               ```
                                                     Do not continue if parsing fails. Use the corrected keycodes here too if
`showkey` reported different values on this keyboard. Record the custom map:

```bash                                              printf 'KEYMAP=/etc/kbd/dk-heini.map\n' > /etc/vconsole.conf                                              ```
                                                     When using the laptop's local virtual console, you may immediately test it:
                                                     ```bash
loadkeys /etc/kbd/dk-heini.map
```

Loading a keymap from the chroot still changes the running machine's console
map. Skip the immediate load over SSH; parsing and initramfs generation can                               still be performed without an interactive console.
                                                     Section 20's `keyboard` and `keymap` hooks must precede `encrypt`. Its
`mkinitcpio -P` run compiles the configured map into the initramfs for the LUKS
prompt, before the encrypted root can be read. If you change the map after
section 20, rebuild the initramfs again while the correct ESP is mounted:
                                                     ```bash
findmnt /boot                                        mkinitcpio -P
lsinitcpio /boot/initramfs-linux-lts.img | grep keymap
```                                                  
For this BusyBox hook arrangement, expect a compiled `keymap.bin`. A generated
file is not a substitute for testing the real keyboard at the first boot.
Keep the recovery medium and intended password characters available.

To revert permanently to ordinary Danish, set `KEYMAP=dk` in                                              `/etc/vconsole.conf`, load `dk` at the local console, and run `mkinitcpio -P`.                            These mappings apply to Linux TTYs. Configure Hyprland/XKB separately for a
desktop session; adding a `.zshrc` binding does not fix an early LUKS prompt.
                                                     
## 18. Write and verify `/etc/fstab`                 
Obtain the actual FAT UUID and LUKS UUID:            
```bash                                              ESP_UUID="$(blkid -s UUID -o value "${ESP_PART}")"
LUKS_UUID="$(blkid -s UUID -o value "${CRYPT_PART}")"
test -n "${ESP_UUID}"                                test -n "${LUKS_UUID}"
printf 'ESP UUID:  %s\nLUKS UUID: %s\n' "${ESP_UUID}" "${LUKS_UUID}"
```                                                  
Create the mount table. The root entry is required because the `zfs`
mkinitcpio hook consults it when the root dataset uses `mountpoint=legacy`.

```bash
tee /etc/fstab >/dev/null <<EOF
# <source>                    <mount point> <type> <options>       <dump> <pass>
rpool/ROOT/arch               /             zfs    defaults       0      0
rpool/DATA/home               /home         zfs    defaults       0      0                                rpool/DATA/var-log            /var/log      zfs    defaults       0      0                                rpool/DATA/var-cache          /var/cache    zfs    defaults       0      0                                rpool/DATA/var-tmp            /var/tmp      zfs    defaults       0      0                                UUID=${ESP_UUID}              /boot         vfat   umask=0077     0      2
EOF

findmnt --verify --verbose --tab-file /etc/fstab
```                                                  
`findmnt` may note that the ZFS sources are not conventional block devices; it
must not report syntax errors, duplicate mount points, or an absent ESP UUID.

## 19. Configure ZFS identity, services, and zram    
Create the stable host ID required for reliable pool import:

```bash
zgenhostid -f -o /etc/hostid
od -An -tx1 /etc/hostid
```

Enable the ZFS target/event daemon and ordinary system services:
                                                     ```bash
systemctl enable \
  NetworkManager.service \
  systemd-timesyncd.service \
  zfs.target \
  zfs-zed.service                                    ```

Use compressed RAM swap by default; this avoids the complications of a swap
ZVOL and deliberately provides no hibernation:

```bash
tee /etc/systemd/zram-generator.conf >/dev/null <<'EOF'
[zram0]
zram-size = min(ram / 2, 8192)                       compression-algorithm = zstd
swap-priority = 100                                  EOF
```                                                  
## 20. Build an initramfs that unlocks LUKS before importing ZFS
                                                     Use the busybox/udev hook family. Do not mix it with `systemd`/`sd-encrypt`; the                          ArchZFS `zfs` hook used here is designed for the former sequence.

Create a mkinitcpio drop-in:

```bash
mkdir -p /etc/mkinitcpio.conf.d
                                                     tee /etc/mkinitcpio.conf.d/10-zfs-root.conf >/dev/null <<'EOF'                                            MODULES=(zfs)
BINARIES=()
FILES=(/etc/hostid)                                  HOOKS=(
  base                                                 udev
  autodetect                                           microcode
  modconf
  kms
  keyboard
  keymap
  block
  encrypt
  zfs
  filesystems                                        )
EOF                                                  
mkinitcpio -P
```

The essential order is:

```text                                              block -> encrypt -> zfs -> filesystems
```

Inspect the generated image:                         
```bash                                              test -s /boot/initramfs-linux-lts.img
lsinitcpio /boot/initramfs-linux-lts.img \             | grep -E '(^|/)(cryptsetup|hostid|zfs\.ko|zpool|mount\.zfs)'                                           ```

The image must contain cryptsetup components, `/etc/hostid`, the ZFS module,
and ZFS userland required by the hook.

## 21. Install and configure rEFInd

With the ESP mounted at `/boot`, install rEFInd and create the UEFI NVRAM entry:                          
```bash                                              findmnt /boot
refind-install                                       ```
                                                     The ArchZFS mkinitcpio hook currently understands both `root=ZFS=...` and
`zfs=...`. Use the explicit root dataset and scan only the mapper directory:
                                                     ```bash
tee /boot/refind_linux.conf >/dev/null <<EOF
"Boot Arch Linux on encrypted ZFS"  "cryptdevice=UUID=${LUKS_UUID}:cryptzfs root=ZFS=rpool/ROOT/arch rw zfs_import_dir=/dev/mapper initrd=\\initramfs-linux-lts.img"
"Boot Arch Linux verbose"           "cryptdevice=UUID=${LUKS_UUID}:cryptzfs root=ZFS=rpool/ROOT/arch rw zfs_import_dir=/dev/mapper loglevel=7 initrd=\\initramfs-linux-lts.img"
EOF
```

Because the mkinitcpio `microcode` hook embeds early microcode into the
initramfs, no separate `amd-ucode.img` or `intel-ucode.img` argument is needed.

Optionally set a visible timeout:                    
```bash                                              REFIND_CONF='/boot/EFI/refind/refind.conf'
test -f "${REFIND_CONF}"
sed -i 's/^timeout .*/timeout 10/' "${REFIND_CONF}"  ```
                                                     Verify the boot files and NVRAM entries:
                                                     ```bash
ls -lh \
  /boot/vmlinuz-linux-lts \                            /boot/initramfs-linux-lts.img \
  /boot/refind_linux.conf \                            /boot/EFI/refind/refind_x64.efi
                                                     efibootmgr --verbose
```

The firmware list should retain `Windows Boot Manager` and add rEFInd. rEFInd
normally discovers `EFI/Microsoft/Boot/bootmgfw.efi` automatically, including                             when it resides on another ESP. Even if it does not appear in rEFInd initially,                           Windows remains bootable from the firmware's own boot menu.                                               
Do not delete or overwrite the `EFI/Microsoft` directory.                                                 
## 22. Final pre-reboot validation
                                                     Run every check below inside the chroot:
                                                     ```bash
set -o pipefail
                                                     pacman -Q linux-lts zfs-linux-lts zfs-utils cryptsetup refind                                             findmnt --verify --verbose --tab-file /etc/fstab
zpool status -v                                      zpool get ashift,autotrim,compatibility,bootfs rpool
zfs list -o name,mountpoint,canmount                 test -s /etc/hostid
test -s /boot/vmlinuz-linux-lts                      test -s /boot/initramfs-linux-lts.img
test -s /boot/EFI/refind/refind_x64.efi
test -s /boot/refind_linux.conf                      grep -F "cryptdevice=UUID=${LUKS_UUID}:cryptzfs" \
  /boot/refind_linux.conf                            grep -F 'root=ZFS=rpool/ROOT/arch' /boot/refind_linux.conf                                                systemctl --root=/ is-enabled NetworkManager.service
systemctl --root=/ is-enabled zfs.target
```

If using section 17.1's custom console map, also check that
`/etc/vconsole.conf` selects `/etc/kbd/dk-heini.map` and that
`lsinitcpio /boot/initramfs-linux-lts.img` lists `keymap.bin`.

If any required check fails, do not reboot.          
Create the first recursive snapshot while the installed state is known:

```bash
zfs snapshot -r rpool@fresh-install
zfs list -t snapshot
sync                                                 exit
```                                                  
The prompt is now back in Alpine.                    
## 23. Cleanly unmount, export, and reboot                                                                ```sh
sync
umount -R /mnt                                       zpool export rpool
cryptsetup close cryptzfs

zpool list
ls /dev/mapper                                       ```

`zpool list` should report no imported pools and `cryptzfs` should be absent.                             Then reboot and remove the USB medium:
                                                     ```sh
reboot
```

## 24. First boot

1. Select Arch Linux in rEFInd.
2. The initramfs should request the LUKS passphrase.
3. After successful decryption, the ZFS pool should import and Arch should
   reach the login prompt.                           4. Log in as the normal user.
                                                     Verify the live system:

```bash                                              findmnt /
for target in /boot /home /var/log /var/cache /var/tmp; do
  findmnt "${target}"                                done
                                                     lsblk --fs
zpool status -v                                      zpool get ashift,autotrim,compatibility,bootfs rpool
zfs list -o name,used,avail,refer,mountpoint,canmountsystemctl --failed
swapon --show
journalctl --boot --priority=warning
```

For Wi-Fi, create the connection with `sudo nmtui` if it was not configured
already; NetworkManager stores it for later boots.   
Expected root source: `rpool/ROOT/arch`. Expected underlying ZFS device:
`/dev/mapper/cryptzfs`. Expected swap: `/dev/zram0` after the generator starts.
                                                     Boot Windows from rEFInd or the firmware menu. Once both systems boot,                                    restore Windows encryption protection according to the branch used in                                     section 4.3. If you suspended protectors, use an administrator terminal:                                  
```powershell                                        manage-bde.exe -protectors -enable C:
manage-bde.exe -status C:                            ```
                                                     Expect Protection On. If you instead turned Device Encryption off and                                     decrypted the volume, turn Device Encryption on again in Settings; enabling                               protectors alone does not re-encrypt a decrypted drive. Verify the recovery                               key afterward. If C: was originally unencrypted, there is nothing to resume.

## 25. Back up the LUKS header and add a recovery passphrase

Do this after the first successful boot, with an external encrypted drive
mounted. Replace the output path with that external location:

```bash
sudo cryptsetup luksAddKey /dev/disk/by-partlabel/ARCH_CRYPT                                              
sudo cryptsetup luksHeaderBackup \                     /dev/disk/by-partlabel/ARCH_CRYPT \
  --header-backup-file \
  /path/on/encrypted-external-drive/ARCH_CRYPT-luks2-header.img                                           
sudo cryptsetup luksDump /dev/disk/by-partlabel/ARCH_CRYPT
```

The header backup does not itself contain the plaintext passphrase, but it is
security-sensitive: an old header preserves the key slots that existed when it                            was copied and permits offline passphrase attacks. Store it encrypted and                                 offline. A passphrase cannot recover a destroyed LUKS header without a usable
header backup.                                       
## 26. Optional SSD discard/TRIM

The privacy-conservative default is no discard through LUKS and
`autotrim=off`. If SSD space reclamation is more important than hiding block
allocation patterns:
                                                     1. Add `:allow-discards` to the `cryptdevice` option on both lines in                                        `/boot/refind_linux.conf`:

   ```text                                              cryptdevice=UUID=<UUID>:cryptzfs:allow-discards
   ```

2. Enable ZFS automatic TRIM:                        
   ```bash
   sudo zpool set autotrim=on rpool
   sudo zpool get autotrim rpool                        ```

For lower-end SSDs, leaving `autotrim=off` and periodically running
`sudo zpool trim rpool` may reduce continuous discard activity. Discards still
require the LUKS `allow-discards` option to reach the physical SSD.
                                                     ## 27. Routine Arch/ZFS maintenance
                                                     ### 27.1 Before every full upgrade
                                                     Create a root snapshot:

```bash                                              sudo zfs snapshot rpool/ROOT/arch@pre-upgrade-$(date +%F)
```                                                  
Refresh all packages together; partial upgrades are unsupported on Arch:
                                                     ```bash
sudo pacman -Syu                                     ```

If pacman reports a conflict between `linux-lts` and `zfs-linux-lts`, do not
force or ignore it. Wait for a matching ArchZFS build, or install the exact
compatible pair intentionally.

Before rebooting after an upgrade:

```bash
pacman -Q linux-lts zfs-linux-lts zfs-utils          find /usr/lib/modules -name 'zfs.ko*' -print
sudo mkinitcpio -P                                   test -s /boot/vmlinuz-linux-lts
test -s /boot/initramfs-linux-lts.img
```

Do not run `zpool upgrade -a` merely because `zpool status` reports available
features. Pool feature activation is one-way and can make recovery media with
older ZFS versions unable to import the pool. The compatibility profile is
intentional.                                         
### 27.2 Scrub and inspect                           
For an ordinary laptop, a monthly scrub is reasonable:
                                                     ```bash
sudo zpool scrub rpool
zpool status rpool
```

Inspect completion later with `zpool status`. Do not suspend or power off the
laptop unnecessarily during the first full scrub.

### 27.3 Keep real backups                           
At minimum, keep versioned backups of home and research data on another
physical device. Use ordinary incremental `zfs send` streams into an encrypted
destination where appropriate. Because this pool uses LUKS rather than native
ZFS encryption, its send stream is plaintext unless the transport or destination
encrypts it. A snapshot on `rpool` fails with the SSD and is not a backup.

## 28. Reopen an existing installation from Alpine Extended
                                                     Use this after rebooting the live medium, after an interrupted installation,                              or when repairing boot. **Do not repeat sections 8-11 on existing data.** This                            section unlocks and mounts; it does not create partitions, filesystems, or a
new pool. It assumes the LUKS/pool/dataset names from this runbook.
                                                     ### 28.1 Prepare the live session and identify existing devices                                           
Repeat section 6: normal Alpine root login, Danish/custom console map, network
if needed, utilities, udev, UEFI verification, and `modprobe zfs`. A minimal
tool installation from the ISO can be attempted without internet:
                                                     ```sh
apk --no-network add cryptsetup eudev util-linux zfs setup-devd udev
udevadm settle
modprobe zfs                                         zfs version
lsblk -o NAME,PATH,SIZE,FSTYPE,PARTLABEL,PARTUUID,MOUNTPOINTS                                             blkid
```                                                  
If a package is not available locally, use section 6.5 or configure networking
and install it from the matching release repositories. Do not proceed after
a failed command that supplies a required tool or kernel module.

Re-establish variables in this shell; they do not survive a reboot:
                                                     ```sh
CRYPT_PART='/dev/disk/by-partlabel/ARCH_CRYPT'       ESP_PART='/dev/disk/by-partlabel/ARCH_EFI'
ls -l "${CRYPT_PART}" "${ESP_PART}"                  blkid "${CRYPT_PART}" "${ESP_PART}"
```

For the shared Windows ESP branch, set `ESP_PART` to its recorded
`/dev/disk/by-partuuid/...` path instead. If a label is missing or duplicated,
identify the correct partition from the disk inventory and use its PARTUUID;
do not create a replacement partition to restore a missing link.

Expected types: `crypto_LUKS` for `CRYPT_PART` and `vfat` for `ESP_PART`.

### 28.2 Unlock the container and inspect pool state
                                                     ```sh
cryptsetup status cryptzfs
```

If it is inactive, open the verified container:      
```sh
cryptsetup open "${CRYPT_PART}" cryptzfs
cryptsetup status cryptzfs
```                                                  
If it is already active, verify that its `device:` is the intended partition
and reuse it. A wrong-passphrase message may be a keyboard-layout problem;
check the harmless symbol test from section 6.3 before retrying. Never run
`luksFormat` to fix an unlock error.

Check both imported pools and pools available to import:

```sh
zpool list
zpool import -d /dev/mapper
```                                                  
`zpool import` without a pool argument lists candidates. An already imported
pool will appear in `zpool list`, not necessarily in the candidate list.

If `rpool` is not yet imported, import it for repair without automatically
mounting its datasets:                               
```sh                                                mkdir -p /mnt
zpool import -N -R /mnt -d /dev/mapper rpool         ```
                                                     This is a read/write repair import. For inspection only, choose this command
**instead**, and do not attempt package installation or boot repairs:                                     
```sh
zpool import -N -R /mnt -o readonly=on -d /dev/mapper rpool
```
                                                     For either branch, if the pool may be active elsewhere, first establish that
only this recovery environment has access to it. Only then may the same
import command be retried with lowercase `-f`. Uppercase `-F` is a different
recovery operation that can discard transactions; it is not a routine retry.
                                                     If the pool is already imported, inspect its state before mounting:

```sh
zpool status -v rpool                                zpool get altroot,readonly rpool
zfs list -r -o name,mountpoint,canmount,mounted rpool
findmnt -R /mnt
```

Do not import a second copy or change permanent dataset mountpoints merely
because a path is missing. If the import mode/alternate root needs changing,
unmount the relevant mounts and cleanly export it first, then reimport. An
empty `findmnt -R /mnt` result just means nothing is mounted there yet.
                                                     ### 28.3 Mount the root dataset before making subdirectories                                              
This runbook uses `mountpoint=legacy`; mount those datasets with `mount -t zfs`.
Do not try to mount the outer LUKS partition or `/dev/mapper/cryptzfs` as ext4.

```sh
mkdir -p /mnt
mount -t zfs rpool/ROOT/arch /mnt                    findmnt --mountpoint /mnt
ls -la /mnt                                          ```
                                                     Stop unless `findmnt` shows `rpool/ROOT/arch` at `/mnt`. If it is already                                 mounted there with the correct source, skip that mount command. If it is
mounted somewhere else, resolve that explicitly; do not stack another mount
on top or continue writing into an ordinary RAM directory.

For the inspection-only import, inspect the root's existing files now and
stop before the following directory creation and repair steps. If you also
need the ESP for inspection and `/mnt/boot` exists, mount it with
`mount -t vfat -o ro "${ESP_PART}" /mnt/boot`. Then use section 28.5 to close.
                                                     Inspect which child datasets actually exist, particularly after an interrupted
installation:
                                                     ```sh
zfs list -r rpool
mkdir -p /mnt/home /mnt/var/log /mnt/var/cache /mnt/var/tmp /mnt/boot
```                                                  
For a completed layout, mount the remaining datasets and the verified ESP.
Skip a mount already present with the correct source. If a dataset was never
created, stop and establish where the installation was interrupted rather than
running a new pool-creation sequence:                
```sh
mount -t zfs rpool/DATA/home /mnt/home               mount -t zfs rpool/DATA/var-log /mnt/var/log
mount -t zfs rpool/DATA/var-cache /mnt/var/cache     mount -t zfs rpool/DATA/var-tmp /mnt/var/tmp
mount -t vfat -o umask=0077 "${ESP_PART}" /mnt/boot  findmnt -R /mnt
```

Do not format the ESP when a mount fails. Inspect `blkid`,
the selected device, the filesystem type, and `dmesg | tail -n 30` instead.
                                                     ### 28.4 Enter the existing Arch system
                                                     First verify that the root has actually been populated with Arch:

```sh
cat /mnt/etc/os-release
test -x /mnt/usr/bin/bash && echo 'Arch Bash exists.'test -x /mnt/usr/bin/pacman && echo 'Arch pacman exists.'
```
                                                     If these files are absent, an interrupted installation may not have reached                               section 13. Do not try to repair a bootloader before installing the base system.                          
For read/write repair, provide virtual filesystems. Perform these binds only
once per recovery session:                           
```sh
mkdir -p /mnt/dev /mnt/proc /mnt/sys /mnt/run        for path in dev proc sys run; do
  mount --rbind "/${path}" "/mnt/${path}"              mount --make-rslave "/mnt/${path}"
done
```

If repair needs internet access, temporarily supply the live resolver file.
Preserve the installed resolver file or symlink itself, rather than writing
through an absolute symlink into the wrong environment. First inspect it:

```sh
ls -l /mnt/etc/resolv.conf
ls -l /mnt/etc/resolv.conf.alpine-save               ```

The second path should be absent. If it already exists, an earlier repair may
be unfinished: inspect and restore it before creating another backup. If the
original `/mnt/etc/resolv.conf` is absent, skip the next move and create a                                temporary resolver file with `cp /etc/resolv.conf /mnt/etc/resolv.conf`; remove                           that temporary file before leaving recovery. Otherwise, preserve and replace:

```sh
if test -e /mnt/etc/resolv.conf.alpine-save || \
   test -L /mnt/etc/resolv.conf.alpine-save; then
  echo 'STOP: An earlier resolver backup already exists.'
else
  mv /mnt/etc/resolv.conf /mnt/etc/resolv.conf.alpine-save &&
    cp /etc/resolv.conf /mnt/etc/resolv.conf         fi
```                                                  
Stop if the copy failed. Do not enter a repair chroot expecting internet until
the resolver file is usable. If packages are not needed, omit this DNS change.
Now enter Arch:

```sh
chroot /mnt /usr/bin/env -i \
  ESP_PART="${ESP_PART}" \
  CRYPT_PART="${CRYPT_PART}" \                         HOME=/root \
  TERM="${TERM:-linux}" \                              PATH=/usr/local/sbin:/usr/local/bin:/usr/bin \
  /bin/bash --login                                  ```

Inside this chroot, `/` is the installed Arch filesystem, while `uname -r`
still reports the live Alpine kernel. Do not use it to select a new Arch kernel
package or attempt to load Arch's ZFS module into Alpine.

After diagnosing the failure, common repair commands are:                                                 
```bash                                              pacman -Q linux-lts zfs-linux-lts zfs-utils
findmnt /boot                                        mkinitcpio -P
refind-install                                       efibootmgr --verbose
```

Run `refind-install` only if bootloader installation actually needs repair.
For kernel/ZFS package mismatch, use section 16's matching-pair procedure.
For custom console keys, check section 17.1 before rebuilding the initramfs.

### 28.5 Leave recovery cleanly

Exit the chroot, then restore the resolver if you temporarily replaced it:

```bash                                              exit
```

```sh
cd /                                                 if test -e /mnt/etc/resolv.conf.alpine-save || \
   test -L /mnt/etc/resolv.conf.alpine-save; then
  mv -f /mnt/etc/resolv.conf.alpine-save /mnt/etc/resolv.conf
fi
```

If the resolver was originally absent and you created a temporary regular                                 file, remove that temporary `/mnt/etc/resolv.conf` now. Skip resolver changes                             for an inspection-only import. Close the mounts and storage:                                              
```sh
sync
umount -R /mnt
zpool export rpool
cryptsetup close cryptzfs
```

If unmount/export fails with `busy`, leave the disk open and inspect
`findmnt -R /mnt`. Exit other shells, editors, and chroots that use it before
retrying. Do not use forced or lazy unmount as a routine shortcut.                                        
Verify closure before rebooting:                     
```sh                                                zpool list
cryptsetup status cryptzfs
```                                                  
No imported `rpool` and an inactive `cryptzfs` are expected. Then run `reboot`.

                                                     ## 29. Early-boot troubleshooting

This section concerns the **installed Arch system**. An emergency shell
while starting the Alpine USB belongs to section 6.1 instead.

### No LUKS passphrase prompt

From recovery, inspect `/boot/refind_linux.conf` and the initramfs:
                                                     ```bash
grep cryptdevice /boot/refind_linux.conf             grep '^HOOKS' /etc/mkinitcpio.conf.d/10-zfs-root.conf
lsinitcpio /boot/initramfs-linux-lts.img | grep cryptsetup
```
                                                     Confirm that the LUKS UUID in rEFInd equals:
                                                     ```bash
blkid -s UUID -o value /dev/disk/by-partlabel/ARCH_CRYPT
```                                                  
### LUKS unlocks, but ZFS pool is not found

At the initramfs emergency shell:

```sh
ls -l /dev/mapper
zpool import -d /dev/mapper
zpool import -N -d /dev/mapper rpool
zfs list                                             ```

If the manual import works, check that the boot command line contains:

```text
root=ZFS=rpool/ROOT/arch rw zfs_import_dir=/dev/mapper
```

and that `encrypt` precedes `zfs` in `HOOKS`.        
### ZFS module missing or invalid module format      
This almost always indicates a kernel/ZFS mismatch. From recovery and chroot:
                                                     ```bash
pacman -Q linux-lts zfs-linux-lts zfs-utils
pacman -Qi zfs-linux-lts | sed -n '/^Depends On/p'
find /usr/lib/modules -name 'zfs.ko*' -print
mkinitcpio -P
```

Install a compatible pair; do not force-load a module built for another kernel.

### rEFInd does not show Windows                     
First use the firmware boot menu and select `Windows Boot Manager`. Verify the
NVRAM entry:                                         
```bash
efibootmgr --verbose
```

If the Microsoft entry and `EFI/Microsoft/Boot/bootmgfw.efi` still exist,
Windows has not been erased. Repair rEFInd scanning separately; do not reinstall
or format Windows partitions as a boot-menu troubleshooting step.

## 30. Primary references

- [OpenZFS: Arch Linux Root on ZFS](https://openzfs.github.io/openzfs-docs/Getting%20Started/Arch%20Linux/Root%20on%20ZFS.html)
- [ArchWiki: Install Arch Linux on ZFS](https://wiki.archlinux.org/title/Install_Arch_Linux_on_ZFS)       - [ArchWiki: ZFS](https://wiki.archlinux.org/title/ZFS)                                                   - [ArchWiki: dm-crypt system configuration](https://wiki.archlinux.org/title/Dm-crypt/System_configuration)
- [ArchWiki: Dual boot with Windows](https://wiki.archlinux.org/title/Dual_boot_with_Windows)
- [ArchWiki: rEFInd](https://wiki.archlinux.org/title/REFInd)
- [ArchZFS current repository and signing instructions](https://github.com/archzfs/archzfs/releases/tag/experimental)                                          - [Current ArchZFS mkinitcpio ZFS hook](https://github.com/archzfs/archzfs/blob/master/src/zfs-utils/zfs-utils.initcpio.hook)
- [OpenZFS pool properties](https://openzfs.github.io/openzfs-docs/man/master/7/zpoolprops.7.html)
- [rEFInd: Methods of Booting Linux](https://www.rodsbooks.com/refind/linux.html)
- [Microsoft: disable and re-enable hibernation](https://learn.microsoft.com/en-us/troubleshoot/windows-client/setup-upgrade-and-drivers/disable-and-re-enable-hibernation)
- [Microsoft: BitLocker operations guide](https://learn.microsoft.com/en-us/windows/security/operating-system-security/data-protection/bitlocker/operations-guide)

### Sources for the 2026-10-15 revision

- [Alpine downloads](https://alpinelinux.org/downloads/)
- [Alpine live-medium preparation](https://docs.alpinelinux.org/user-handbook/0.1a/Installing/medium.html)
- [Alpine setup tools](https://docs.alpinelinux.org/user-handbook/0.1a/Installing/manual.html)
- [Alpine apk and repositories](https://docs.alpinelinux.org/user-handbook/0.1a/Working/apk.html)
- [Alpine Wi-Fi](https://wiki.alpinelinux.org/wiki/Wi-Fi)
- [Alpine setup-interfaces source](https://github.com/alpinelinux/alpine-conf/blob/master/setup-interfaces.in)
- [Alpine setup-keymap source](https://github.com/alpinelinux/alpine-conf/blob/master/setup-keymap.in)
- [Alpine Standard/Extended image profiles](https://github.com/alpinelinux/aports/blob/master/scripts/mkimg.standard.sh)                                       - [BusyBox applet reference](https://busybox.net/downloads/BusyBox.html)
- [wpa_passphrase manual](https://man.archlinux.org/man/wpa_passphrase.8.en)
- [wpa_supplicant manual](https://man.archlinux.org/man/wpa_supplicant.8.en)
- [loadkeys manual](https://man.archlinux.org/man/loadkeys.1.en)
- [Console keymap format](https://man.archlinux.org/man/keymaps.5.en)
- [Arch console keyboard configuration](https://wiki.archlinux.org/title/Linux_console/Keyboard_configuration)                                                 - [OpenZFS pool import](https://openzfs.github.io/openzfs-docs/man/master/8/zpool-import.8.html)          - [Microsoft Device Encryption](https://support.microsoft.com/en-us/windows/security/encryption/device-encryption-in-windows)
- [Microsoft BitLocker error codes](https://learn.microsoft.com/en-us/windows/win32/com/com-error-codes-6)
- [Microsoft drive encryption status](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-status)                      - [Microsoft protector management](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-protectors)