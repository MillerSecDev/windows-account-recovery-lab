# windows-account-recovery-lab
Authorized Windows security lab documenting offline account recovery and credential-protection lessons.
# Windows 7 Local Account Recovery Lab

> **Authorized-use statement:** This lab was performed only on a personally
> owned computer for the purpose of restoring access to the owner's local
> account. Credential values and identifying account information are omitted.
> The techniques described here must not be used without explicit authorization.

## Overview

This project documents the recovery of a forgotten local-account password on
an owner-controlled Toshiba laptop running Windows 7 Home Premium. Preserving
the existing user profile and files was the primary requirement.

Instead of resetting the password or reinstalling Windows, I:

1. Booted a compact Linux-based recovery environment.
2. Copied the offline `SAM` and `SYSTEM` registry hives without committing a
   registry change.
3. Extracted the account's NTLM hash on a separate Kali Linux virtual machine.
4. Performed a targeted, owner-informed password audit with Hashcat.
5. Recovered the original password and verified access to the unchanged
   Windows profile.

**Result:** The original password was recovered, the account login was
verified, and the user's files remained intact.

## Skills Demonstrated

- Legacy BIOS boot-media preparation
- Linux disk and partition discovery
- Manual creation of missing Linux block-device nodes
- Evidence-conscious collection of offline Windows registry hives
- NTLM hash extraction with Impacket
- Targeted GPU-assisted password auditing with Hashcat
- PowerShell and shell command-line use
- Troubleshooting across Windows, Linux, USB, and virtualized environments
- Security analysis and defensive-control recommendations

## Lab Environment

| Component | Role |
| --- | --- |
| Toshiba laptop | Owner-controlled Windows 7 Home Premium target |
| 515 MB PNY USB 2.0 drive | Boot media and temporary evidence transfer |
| Windows workstation | Rufus, Hashcat, and file handling |
| Kali Linux virtual machine | Offline hive processing with Impacket |
| NVIDIA GeForce RTX 2080 SUPER | Hashcat GPU acceleration |

## Tools

| Tool | Purpose |
| --- | --- |
| Rufus 4.15 | Create bootable recovery media |
| `chntpw` `cd140201` image | Boot a compact offline Windows recovery environment |
| Impacket `secretsdump` | Extract local account hashes from offline hives |
| Hashcat 7.1.2 | Audit the extracted NTLM hash |
| PowerShell and Linux shell | File handling, inspection, and troubleshooting |

## Objectives

- Restore access without reinstalling Windows.
- Preserve the original user profile and files.
- Recover the original password if feasible instead of resetting it.
- Avoid committing changes to the Windows registry during collection.
- Document failures, corrective actions, and security implications.
- Verify success by logging into the original account.

## Procedure

### 1. Select recovery media

The first plan was to use Hiren's BootCD PE, but its approximately 3 GB ISO
could not fit on the only available USB drive, which had approximately 515 MB
of usable capacity. I switched to the much smaller `chntpw` boot image.

The first Rufus write failed with:

```text
The selected cluster size is not valid for this device.
```

The successful configuration was:

| Rufus setting | Value |
| --- | --- |
| Boot selection | `cd140201.iso` |
| Partition scheme | MBR |
| Target system | BIOS or UEFI-CSM |
| File system | FAT |
| Cluster size | 8192 bytes |

### 2. Boot and identify the Windows partition

Two of the laptop's USB ports were physically damaged. The USB drive worked
through the eSATA/USB combination port and eventually appeared in the F12 boot
menu.

The recovery environment identified the main Windows installation on
`/dev/sda2` and mounted it at `/disk`.

Discovery commands included:

```bash
mount
fdisk -l
cat /proc/partitions
```

### 3. Repair missing USB device nodes

The recovery kernel detected the USB drive as `sdc` and `sdc1` in
`/proc/partitions`, but the corresponding entries were missing from `/dev`.
Initial mount attempts therefore failed.

Using the detected major and minor numbers, I manually created the block-device
nodes and mounted the USB partition:

```bash
mkdir -p /usb
mknod /dev/sdc b 8 32
mknod /dev/sdc1 b 8 33
ls -l /dev/sdc*
mount /dev/sdc1 /usb
ls /usb
```

### 4. Collect the offline registry hives

An initial copy attempt failed because the Linux recovery environment treated
the source path as case-sensitive. The hive filenames on this installation
were lowercase.

The corrected collection commands were:

```bash
cp /disk/Windows/System32/config/sam /usb/SAM
cp /disk/Windows/System32/config/system /usb/SYSTEM
sync
ls -l /usb/SAM /usb/SYSTEM
umount /usb
```

The registry-edit workflow was exited without accepting writeback. The tool
reported `CANCELLED`, confirming that no password-reset change was committed.

### 5. Extract the NTLM hash in Kali Linux

The copied hives were attached to a Kali Linux virtual machine and processed
locally:

```bash
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

Only the target account's NTLM field was written to the Hashcat input file:

```bash
impacket-secretsdump -sam SAM -system SYSTEM LOCAL 2>/dev/null \
  | awk -F: '$1=="TARGET_USER"{print $4}' > hash.txt
wc -c hash.txt
```

The file measured 33 bytes: a 32-character hexadecimal NTLM value plus a
newline. The hash value is intentionally excluded from this repository.

### 6. Perform a targeted password audit

Hashcat detected the RTX 2080 SUPER through its OpenCL/CUDA backend:

```powershell
.\hashcat.exe -I
```

A small private candidate list was built using the owner's memory of likely
password patterns. The first exact-candidate pass exhausted all 11 entries:

```powershell
.\hashcat.exe -m 1000 -a 0 .\hash.txt .\guesses.txt
```

The expected `best64.rule` file was not present in the installed Hashcat
distribution, so the available rules were enumerated:

```powershell
Get-ChildItem .\rules -File | Select-Object Name
```

An installed rule was then used for a targeted mutation pass:

```powershell
.\hashcat.exe -m 1000 -a 0 .\hash.txt .\guesses.txt `
  -r .\rules\best66.rule
```

This pass recovered the original password. The recovered credential, candidate
list, hash, and Hashcat potfile are not included in this repository.

### 7. Verify the result

The recovered password successfully opened the original local account. The
existing Windows profile and user files were preserved, and no password reset
or operating-system reinstallation was required.

## Troubleshooting Summary

| Problem | Cause | Resolution |
| --- | --- | --- |
| Hiren's ISO would not fit | USB capacity was only about 515 MB | Switched to the compact `chntpw` image |
| Rufus rejected the write | Incompatible filesystem or cluster selection | Used MBR, BIOS/CSM, FAT, and 8192-byte clusters |
| USB missing from boot menu | Damaged ports or incomplete detection | Used the combo port, reinserted the drive, and reopened F12 |
| USB mount failed | Kernel detected the device but `/dev` nodes were missing | Created block-device nodes from `/proc/partitions` values |
| Hive copy failed | Linux paths were case-sensitive | Used lowercase `sam` and `system` source filenames |
| VM reported the volume busy | A terminal still referenced the USB mount | Changed to the home directory, ran `sync`, closed the terminal, and ejected |
| `best64.rule` not found | Installed Hashcat version contained different rule names | Listed the rules and selected the available `best66.rule` |
| Exact candidates failed | The password was a variation of a remembered candidate | Applied targeted rule-based mutations |

## Validation

| Acceptance test | Status |
| --- | --- |
| Bootable recovery media created | Pass |
| Windows installation identified | Pass |
| `SAM` and `SYSTEM` collected | Pass |
| No registry writeback accepted | Pass |
| NTLM hash extracted | Pass |
| Hashcat GPU backend detected | Pass |
| Original password recovered | Pass |
| Original account login verified | Pass |
| User profile and files preserved | Pass |

## Security Findings

This lab demonstrated that a Windows logon password alone does not protect
files from someone with physical access when the system disk is not encrypted.
A bootable operating system can access the offline Windows partition and
collect credential-verifier material.

Recommended controls:

- Use full-disk encryption such as BitLocker on supported systems.
- Store encryption recovery keys separately and securely.
- Use strong, unique passwords stored in a password manager.
- Keep Secure Boot and firmware boot protections enabled where supported.
- Restrict physical access to computers and removable boot media.
- Migrate data away from unsupported operating systems such as Windows 7.
- Treat `SAM`, `SYSTEM`, NTLM hashes, candidate lists, and potfiles as secrets.

## Evidence and Privacy Notes

This was an evidence-conscious recovery rather than a formal forensic
acquisition. The Windows volume was mounted by the recovery utility without a
hardware write blocker, so the procedure should not be represented as
court-defensible forensics.

This public repository intentionally excludes:

- The recovered password
- LM or NTLM hash values
- The private candidate list
- Hashcat potfiles
- Copies of the `SAM` and `SYSTEM` hives
- Personal files or identifying screenshots

## Lessons Learned

1. **Tool choice is constrained by the environment.** The best-known recovery
   suite was unusable because the available media was too small.
2. **Legacy systems fail in unusual ways.** Reading device information from
   `/proc/partitions` made it possible to repair missing `/dev` nodes.
3. **Case sensitivity matters across operating systems.** Lowercase hive
   filenames were required in the Linux environment.
4. **Context can outperform brute force.** A small, high-quality candidate list
   with rule mutations was effective without searching a huge keyspace.
5. **Recovery artifacts are credentials.** Offline hives, hashes, candidate
   lists, and potfiles require strong protection and secure disposal.
6. **Verification completes the lab.** Success was confirmed only after the
   unchanged account opened with its original profile intact.

## Final Status

**SUCCESS — original password recovered, login verified, and data preserved.**

