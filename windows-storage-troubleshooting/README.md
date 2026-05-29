# Windows Storage Troubleshooting – Linux-Formatted HDD, Partition Recovery and SMART Analysis

## Project Overview

This case study documents the troubleshooting and recovery process of a 1 TB Seagate HDD connected through a USB docking station.

The drive was initially formatted from Linux and later became inaccessible in Windows. The investigation included partition analysis, filesystem troubleshooting, DiskPart recovery procedures, disk health verification using SMART attributes, and filesystem integrity validation with CHKDSK.

## Environment

### Hardware

* Seagate ST31000524AS 1 TB HDD
* USB SATA Docking Station
* Windows 11

### Tools Used

* Windows Disk Management
* PowerShell
* DiskPart
* CHKDSK
* CrystalDiskInfo

---

# Initial Problem

After connecting the HDD to Windows:

* The disk appeared in Disk Management.
* The partition was visible.
* No drive letter was assigned.
* The option **"Change Drive Letter and Paths"** was unavailable.
* The drive did not appear in File Explorer.

## Disk Management Findings

Disk Management reported:

* Disk online
* Primary partition present
* Capacity: 931.51 GB
* No accessible filesystem detected

---

# Investigation

## PowerShell Analysis

Command:

```powershell
Get-Partition
```

Relevant output:

```text
PartitionNumber : 1
Size            : 931.51 GB
Type            : Unknown
```

Further investigation:

```powershell
Get-Partition -DiskNumber 4 | Format-List *
```

Important findings:

```text
Type       : Unknown
IsHidden   : True
MbrType    : 131
```

## Root Cause Identification

MBR Type 131 (0x83) corresponds to a Linux native partition type.

Windows successfully detected the physical disk but treated the partition as a Linux volume and therefore:

* did not mount it,
* did not assign a drive letter,
* did not expose it through File Explorer.

Although the intention was to create an NTFS volume under Linux, the resulting partition metadata was interpreted by Windows as a Linux partition.

---

# Resolution

The disk contained no important data.

The partition structure was recreated using DiskPart.

## Commands Used

```cmd
diskpart
list disk
select disk 4
clean
convert gpt
create partition primary
format fs=ntfs quick
assign
exit
```
## PowerShell commands

![PowerShell commands](ps1.png

## Result

Immediately after assigning a drive letter:

* Windows mounted the volume successfully.
* The disk appeared in File Explorer.
* NTFS was recognized correctly.
* The drive became usable for storage and backup purposes.

---

# Disk Health Assessment

After recovering access to the drive, SMART diagnostics were performed using CrystalDiskInfo.

## Initial SMART Status

Relevant attributes:

| ID | Attribute                    | Value |
| -- | ---------------------------- | ----- |
| 05 | Reallocated Sectors Count    | 40    |
| C5 | Current Pending Sector Count | 12    |
| C6 | Uncorrectable Sector Count   | 12    |

### Interpretation

#### Reallocated Sectors (05)

A reallocated sector is a physical sector that has failed and has been replaced using the drive's spare sector pool.

Value:

```text
40
```

This indicates that the drive has already experienced physical media degradation.

#### Current Pending Sectors (C5)

Pending sectors are sectors that could not be read reliably and are awaiting re-evaluation.

Value:

```text
12
```

These sectors may later:

* recover,
* become readable,
* or be remapped permanently.

#### Uncorrectable Sectors (C6)

Sectors that the drive was unable to read correctly during previous operations.

Value:

```text
12
```

This indicates a history of read failures.

---

# Filesystem Validation

To validate the drive surface and filesystem integrity:

```cmd
chkdsk G: /r
```

Execution time:

```text
2.93 hours
```

Result:

```text
0 KB in bad sectors
Windows has scanned the file system and found no problems.
```

---

# Post-CHKDSK SMART Verification

SMART values after the full surface scan:

| ID | Before | After |
| -- | ------ | ----- |
| 05 | 40     | 40    |
| C5 | 12     | 6     |
| C6 | 12     | 6     |

## Analysis

The number of pending and uncorrectable sectors decreased by 50%.

This suggests that some problematic sectors were successfully re-read during the scan and no longer remain flagged as problematic.

No increase occurred in the number of reallocated sectors.

---

# Final Assessment

## Storage Reliability

The drive remains operational and usable.

However, SMART data indicates historical media degradation.

### Suitable Uses

* Linux labs
* Virtual machines
* ISO repositories
* Driver archives
* Temporary storage
* Secondary backup destination

### Unsuitable Uses

* Sole backup location
* Critical business data
* Only copy of personal documents
* Long-term archival storage

---

# Lessons Learned

* Disk visibility does not guarantee filesystem accessibility.
* Filesystem, partition type, and drive letter assignment are separate layers.
* PowerShell provides more detailed diagnostics than Disk Management alone.
* MBR Type 131 (0x83) is a strong indicator of a Linux partition.
* SMART data should always be reviewed before assigning a storage role.
* CHKDSK and SMART complement each other and provide different perspectives on disk health.
* DiskPart can quickly recover a non-critical disk when data preservation is not required.

---

# Skills Demonstrated

* Windows Troubleshooting
* Storage Administration
* PowerShell Diagnostics
* DiskPart
* Filesystem Analysis
* SMART Interpretation
* CHKDSK Validation
* Root Cause Analysis
* Technical Documentation
