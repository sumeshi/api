# NTFS Spinal Cord Sword
How to rip files straight out of Windows disk images. AHGAHGAHGAHGAHGAHGAHGAHGAHGAHGAHGAHGAH

## Introduction

In digital forensics, it's common practice to preserve data as a disk image so that the investigation itself doesn't alter (or contaminate) the system being examined.

The trouble is that disk images are generally about as large as the original disk. Image a PC with a 1 TB drive, and you can end up with a ridiculously huge file that's close to 1 TB.

When you need to investigate it, you mount the image, open it in a tool that lets you browse its contents, extract the files you need... and so on. It's a pain.

For Windows systems, people commonly use tools such as [FTK Imager](https://www.exterro.com/digital-forensics-software/ftk-imager) or [Arsenal Image Mounter](https://arsenalrecon.com/products/arsenal-image-mounter).

![ftkimager](https://github.com/user-attachments/assets/7da6f2ca-9b57-49e0-b720-22b8fcdf3859)

These tools are very easy to use thanks to their GUIs, but they're not particularly well suited to processing large numbers of images automatically.

When you've got dozens of machines to investigate, this gets exhausting pretty quickly. So I made two tools that let you search for and rip files directly out of disk images.

[ntfsdump](https://github.com/sumeshi/ntfsdump)

[ntfsfind](https://github.com/sumeshi/ntfsfind)

## Supported Image Formats

The following image formats are supported. There's automatic format detection, so you normally don't need to worry about which one you're working with.

- `dd` (RAW)
- `E01` (EnCase)
- `VHD` / `VHDX` (including `AVHDX` differencing disks)
- `VMDK` (including split extents and snapshot delta chains)
- VMware VM directories, `.vmx` files, and `.vmsd` files

The supported file system is **NTFS**, and both **GPT** and **MBR** partition tables are supported.

In versions prior to `v3.2.0`, if a virtual machine image consisted of a chain of differencing snapshots, you had to convert it to a RAW disk first. That's generally no longer necessary.

```powershell
> VBoxManage clonehd --format raw vmdiskimage.vmdk imagefile.raw
```

## Installation

### Precompiled Binaries

Precompiled binaries for Windows and Linux (Ubuntu) are available on GitHub. Just download the appropriate binary from the release pages and run it.

- [ntfsdump - Releases](https://github.com/sumeshi/ntfsdump/releases)
- [ntfsfind - Releases](https://github.com/sumeshi/ntfsfind/releases)

The executables are built using [**Nuitka**](https://nuitka.net/), which compiles Python code into executables. Because of this, **some antivirus products may flag them**.

If you're concerned about that, you can install the tools from PyPI as described below, or run the binaries in an isolated virtual machine.

### Installing from PyPI

Python 3.13 or later is supported. To install both tools, run:

```bash
$ pip install ntfsdump ntfsfind
```

## Searching for Files

[`ntfsfind`](https://github.com/sumeshi/ntfsfind) searches file paths by parsing MFT records directly from a disk image.

For example, if you want to find files with the `.evtx` extension, run:

```powershell
> ntfsfind.exe .\example.E01 ".*\.evtx"
/Windows/System32/winevt/Logs/Setup.evtx
/Windows/System32/winevt/Logs/Microsoft-Windows-WindowsUpdateClient%4Operational.evtx
/Windows/System32/winevt/Logs/Microsoft-Windows-Winlogon%4Operational.evtx
/Windows/System32/winevt/Logs/Microsoft-Windows-WindowsBackup%4ActionCenter.evtx
/Windows/System32/winevt/Logs/Microsoft-Windows-TerminalServices-LocalSessionManager%4Admin.evtx
/Windows/System32/winevt/Logs/Microsoft-Windows-TerminalServices-LocalSessionManager%4Operational.evtx
/Windows/System32/winevt/Logs/Microsoft-Windows-PrintService%4Admin.evtx
...
```

Search queries support regular expressions. Path separators are normalized to `/` on both Windows and Linux.

### Metadata Filters

You can also narrow down the results using metadata stored in MFT records.

```powershell
# EVTX and EXE files at least 1 MB in size, created in 2015 or later
> ntfsfind.exe .\example.E01 --extension evtx,exe --size ">=1MB" --created ">=2015-01-01"
/$Recycle.Bin/S-1-5-21-2425377081-3129163575-2985601102-1000/$RJEMT64.exe
/Users/informant/Desktop/Download/ccsetup504.exe
```

```powershell
# Deleted entries under System32
> ntfsfind.exe .\example.E01 --path /Windows/System32 --deleted-only
/Windows/System32/LogFiles/WMI/RtBackup/EtwRTEventlog-Security.etl
/Windows/System32/LogFiles/WMI/RtBackup/EtwRTUBPM.etl
/Windows/System32/LogFiles/WMI/RtBackup/EtwRTMsMpPsSession7.etl
```

```powershell
# Display executable files of at least 5 MB in a table
> ntfsfind.exe .\example.E01 -e exe --size ">=5MB" --output-format table
72096  ALLOC-FILE    68.3 MiB  2015-03-23 19:55:47  2015-03-23 19:56:53  -     /Users/informant/Downloads/icloudsetup.exe
75186  DELETED-FILE  5.1 MiB   2015-03-25 14:48:28  2015-03-25 14:48:28  -     /Users/informant/Desktop/Download/ccsetup504.exe
```

Here are some of the main filters. When multiple filters are specified, they are combined using **AND** logic.

| Option | Description |
| --- | --- |
| `--extension, -e` | Filter by extension (comma-separated, e.g. `-e evtx,exe`) |
| `--path` | Match a path prefix |
| `--size` | Filter by size (e.g. `>=10MB`, `<1KB`, `4KB..10MB`) |
| `--created / --modified / --accessed` | Filter by timestamp (e.g. `2024-01-01..2024-12-31`) |
| `--timestamp-source` | Select the timestamp source: `si` (`$STANDARD_INFORMATION`, default) or `fn` (`$FILE_NAME`) |
| `--deleted-only / --allocated-only` | Include only deleted or allocated entries |
| `--files-only / --dirs-only` | Include only files or directories |
| `--ads-only` | Include only named `$DATA` stream records |
| `--no-ads` | Exclude named `$DATA` stream records |
| `--attributes` | Filter by file attributes (e.g. `hidden,system,readonly`) |

### Output Formats

You can choose the output format using `--output-format`.

| Format | Description |
| --- | --- |
| `text` | One path per line. Default; useful for piping into other commands |
| `json` | JSON Lines |
| `csv` | CSV |
| `table` | Human-readable table |

### Exporting the MFT

If you plan to search the same image over and over, it's more efficient to extract just the `$MFT` first instead of repeatedly reading a massive disk image.

```powershell
# Export the MFT from the image
> ntfsfind.exe --out-mft C:\tmp\my_mft.bin .\example.E01

# Search the exported MFT file directly from then on
> ntfsfind.exe C:\tmp\my_mft.bin ".evtx"
```

## Extracting Files

`ntfsdump` extracts files directly from disk images by specifying their paths.

```powershell
# Extract a single file
> ntfsdump.exe .\example.E01 "/hoge.txt"

# Recursively extract a directory and its contents
> ntfsdump.exe -o .\dump .\example.E01 /Windows/System32/winevt/Logs

# Extract an alternate data stream (ADS)
> ntfsdump.exe -o .\dump .\example.E01 '/$Extend/$UsnJrnl:$J'
```

The original directory structure is recreated in the output location (for example, `./dump/Windows/System32/winevt/Logs/System.evtx`).

If you'd rather put everything into a single directory, use `--flat`. The output filename joins the original path components with `_`: for example, `/Windows/System32/cmd.exe` becomes `Windows_System32_cmd.exe`.

For ADS, `:` is replaced with `_` in the filename so the stream can be saved as a regular file on Windows. In the example above, `$UsnJrnl:$J` becomes `$UsnJrnl_$J`.

### Using ntfsfind with ntfsdump

You can also pipe the results from `ntfsfind` straight into `ntfsdump` to extract the matching files.

Alternatively, redirect the search results to a text file, keep only the paths you need, and then feed that list to `ntfsdump`.

```powershell
> ntfsfind.exe .\example.E01 ".*\.evtx" | ntfsdump.exe -o .\dump .\example.E01
```

## Searching and Extracting Files from VM Snapshots

For VMware virtual machines, you can specify the VM directory directly. You can also select a snapshot to search or extract files from the NTFS volume as it existed at that point in time.

```powershell
# List available snapshots
> ntfsfind.exe .\WindowsVM --list-snapshots
ID  NAME          CREATED              PARENT
1   Initialized   2026-09-01 12:33:43  -
5   NetConnect    2026-09-02 02:25:08  1
6   PrepareTools  2026-09-10 17:46:35  5
7   SetConfigs    2026-09-25 00:51:39  5

# Search the NTFS volume as it was at snapshot 5
> ntfsfind.exe .\WindowsVM --snapshot 5 ".*\.evtx"
```

```powershell
# Extract the SYSTEM registry hive from snapshot 5
> ntfsdump.exe .\WindowsVM --snapshot 5 /Windows/System32/config/SYSTEM
```

The ID passed to `--snapshot` is the snapshot UID managed by VMware. Snapshot display names aren't supported, so use the ID shown by `--list-snapshots`.

If you omit `--snapshot`, the tools read the VM's **current state**.

### Selecting a Virtual Disk

If a VM has multiple virtual disks attached, use `--list-disks` to see what's available, then select the one you want with `--disk`.

```powershell
> ntfsdump.exe .\WindowsVM --list-disks
ID  NODE     SIZE     VMDK
0   nvme0:0  100 GiB  Windows10_22H2(x64).vmdk
1   scsi0:1  500 GiB  Data.vmdk

# Read the second virtual disk
> ntfsdump.exe .\WindowsVM --disk 1 /Evidence
```

If there's only one disk, it is selected automatically. If there are multiple disks, the tools require you to choose one explicitly to avoid accidentally reading the wrong disk.

You can also specify a differencing VMDK directly.

```powershell
> ntfsdump.exe .\WindowsVM\Windows-000003.vmdk /$MFT
```

### Hyper-V Checkpoints

Similarly, for Hyper-V, you can specify an `AVHDX` differencing disk directly.

```powershell
> ntfsfind.exe .\HyperVM\Disk_0.avhdx ".*\.evtx"
```

Even if you provide just the path to a differencing VHDX file, the tool automatically follows the parent chain using the disk metadata. This lets it treat the base disk and all its differencing disks as a single logical disk.

Unlike the VMware support, there's currently no feature for listing Hyper-V checkpoints. You'll need to specify the `AVHDX` file directly.

## Conclusion

This isn't unique to these tools, but what I'm aiming for is the kind of utility you might not use every day, yet you'll be glad to have around when you need it. Something where just having a single binary in your toolbox can save you a lot of hassle.

If you find these tools useful, I'd be happy if you kept them tucked away in a corner of your toolbox.

That's all for now.
