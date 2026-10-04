# a tale of volatile memories.
Windows is only one in the world. That made Windows the god of this world.

> **Note:** This is the English translation of the Japanese original. The Japanese version is available at https://sumeshi.github.io/posts/knowledges/windows-memory-forensics-101.


## Introduction

When preserving evidence during incident response, you probably capture memory (RAM) too.
But once you've got it, it's easy to end up asking, "Okay, what am I supposed to look at now?"

Memory contains information you can't get from files and logs on disk alone.
Running processes, network connections, file caches, the registry, fragments of data that applications were handling... there's plenty to investigate.

It isn't like reading logs. You're **using tools to dig the information you need out of a binary blob**. It can be a bit of a pain.

This article starts with what can remain in memory, then walks through acquisition, analysis, and organizing the results.


## What Is Memory?

"Memory" can mean a few different things.
Sometimes it means **physical memory** in RAM; sometimes it means the **virtual memory** a process sees. For now, just think of it as where the OS and applications keep their code and data while they're running.

Memory isn't meant for long-term storage. Its contents change as the OS and applications run, and information disappears as processes exit and memory regions get reused.
RAM is **volatile**: cut the power, and its contents are gone. Shut down a compromised computer, and anything that existed only in RAM is lost for good. **Preserve it before shutting down or restarting, if you can.**

Capturing memory from a running computer is called live acquisition. Save that capture to a file, and you have a memory image, or memory dump. In memory forensics, you'll generally spend the rest of your time wrestling with that image.

To use memory efficiently, the OS may write some data to disk, for example in `pagefile.sys`. Those files can also help with the investigation.


### What Can Remain in Memory?

Memory holds program code, stacks and heaps, loaded DLLs, and the structures used to track processes and sockets. Analyzing these can tell you which processes were running at acquisition time, their command lines, and where they had connections.

You may also find cached files and registry data that processes were using. For example, if fragments of EVTX files survive in memory after an attacker deleted them or log rotation removed them from disk, **you may be able to recover event logs that are no longer available on disk**.

There's also no guarantee that freeing a memory region will zero it out immediately. Until it gets overwritten, some data from a terminated process, such as URLs or paths, may remain.


### Physical and Virtual Memory

The virtual memory a process sees is laid out differently from physical RAM. **Windows manages memory in units called pages**, using page tables to map virtual addresses to physical addresses. A region that looks contiguous to a process may be scattered across physical memory.

This affects searches and recovery. For example, if a string spans pages that are physically far apart, a simple scan from the start of physical memory may miss it.

In that case, a tool may find it by resolving the page mappings, reconstructing the process's virtual memory, and searching that instead.
On the other hand, fragments a process no longer references may be easier to find by scanning physical memory directly. Neither approach wins across the board.


## Acquiring Memory

If you're acquiring memory for a forensic investigation, there are a few things worth knowing beforehand.

### Things to Watch During Acquisition

Whichever tool you use, test it beforehand against the target Windows build, CPU architecture, and any restrictions on loading drivers.
It's pretty common to hit an error just when you're ready to capture the evidence. We've all seen an error turn up in production that somehow never appeared during testing.

Live acquisition takes time. It depends on the tool and the speed of the destination storage, but in my experience, roughly a minute per GB isn't unusual.
Keep that in mind if the machine has a stupid amount of RAM.

The OS and applications keep running during acquisition, so different regions of the image are captured at different times. Their structures and data may not agree with each other. This inconsistency is called [Memory Smear](https://www.nist.gov/glossary-term/39326).
It's hard to prevent, so keep in mind that it can happen.


### What to Preserve Besides RAM

On Windows, **some or all of the contents of memory may also be written to these files**. They may help recover missing data or examine an earlier state, so preserve them too if possible.

| File | What it contains |
| --- | --- |
| `C:\pagefile.sys` | Pages moved out of RAM |
| `C:\swapfile.sys` | A file used to swap out application memory; available since Windows 8 / Server 2012 |
| `C:\hiberfil.sys` | State saved during hibernation or Fast Startup |
| `C:\Windows\MEMORY.DMP` | A memory dump from a crash or another dump operation; coverage depends on the dump type |

If you're using `pagefile.sys` or `swapfile.sys` to fill gaps in a RAM capture, acquire them as close to the RAM capture as possible. If the captures are too far apart, reused pages may be incorrectly matched up.


### Acquisition Tools

There are plenty of tools for collecting this evidence. For incident response, a collection tool such as [Magnet RESPONSE](https://www.magnetforensics.com/resources/magnet-response/) is a good option. It can collect memory, `pagefile.sys`, volatile data, and key artifacts in one go.

![capture](https://github.com/user-attachments/assets/ccabbab6-4c10-4d28-a03f-7dffdb4b1ad5)

![capture-completed](https://github.com/user-attachments/assets/223bc154-a91f-4621-84c3-c7ea2a4a98dd)


[FTK Imager](https://www.exterro.com/digital-forensics-software/ftk-imager) used to be a solid option too, but a period of unreliable memory captures and other issues hurt its reputation. Fixed now, maybe? A shame, really.
I get the impression Magnet is more popular in the industry these days.

![ftkimager](https://github.com/user-attachments/assets/b20b304c-4d8d-4ef4-a375-93b17f018c9e)

In Japan, [CDIR-Collector](https://github.com/CyberDefenseInstitute/CDIR/blob/master/README_en.md) is also widely used. It collects memory and key artifacts together. It doesn't collect `pagefile.sys`, but if that isn't a concern for you, this works too.

![cdirini](https://github.com/user-attachments/assets/27205b84-b499-45db-a5db-340b6ecda47d)

Configure the collection in `cdir.ini`. To capture memory, enable `MemoryDump = true`, then double-click `cdir-collector.exe` and let it do its thing.

Whichever tool you use, once the capture finishes, at least check that the file sizes make sense and that you can read a process list. Don't skip this when collecting from multiple machines or leaving the analysis for later. **"We'll just capture it again later" is easier said than done.**


### Recording the Acquisition

Keep the following alongside the captured files so you can reconstruct what happened during acquisition:

- Target hostname
- Acquisition start and end times
- Tool name and version, command line, and collection settings
- Output location and format, and hashes of the acquired files
- Acquisition logs, including errors and failed reads

Magnet RESPONSE writes this information to its logs, which is reassuring.


### Unpacking a Hibernation File

Of the files you've preserved, `hiberfil.sys` needs an extra step before analysis. Its memory data is compressed, so use something like [Hibernation Recon](https://arsenalrecon.com/products/hibernation-recon/faqs) to unpack and reconstruct it into a form your analysis tools can use.

An activation dialog appears on the first launch. Click Cancel to use Free Mode. If you like it, buy a Professional license.

```powershell
> .\HibRec.exe /HiberFil=C:\Cases\hiberfil.sys
```

The main outputs are:

| Output | Contents and uses |
| --- | --- |
| `ActiveMemory.bin` | Decompressed and reconstructed memory; pass it to a compatible memory analysis tool |
| `RawSlackChunks/` | Slack space outside the currently valid hibernation data; you can try string searches or carving against it |
| `HibRec.log` | Processing log, including errors |

Keep in mind that a `hiberfil.sys` written during a Fast Startup shutdown [doesn't contain the full user sessions](https://learn.microsoft.com/en-us/windows/win32/power/system-power-states), so it doesn't give you the same coverage as a live memory capture.

Depending on when the machine hibernated, the hibernation file and a live capture may give you both an earlier memory state and a current one. Looking at both lets you compare two points in time instead of staring at a single snapshot.


## Preparing for Analysis

### What Are You Trying to Find?

Once you have a memory image, decide what to examine based on what you want to know.
If you're looking for traces of known malware, you can search the memory image for distinctive strings or IOCs. If you want to know which process connected where, examine the process and network structures.

| Goal | Approach | Main tools |
| --- | --- | --- |
| Find known domains, paths, commands, or distinctive strings | String searches | Strings, bstrings, ripgrep, ripgrep-all, langscan |
| List URLs and email addresses | Pattern matching | bulk_extractor |
| Find specific byte sequences or regions matching multiple conditions | Rule matching | YARA |
| Investigate processes and their connections | Parse OS data structures | Volatility, MemProcFS |
| Extract files from memory | Analyze file objects and caches | Volatility, MemProcFS |
| Extract files whose tracking structures are gone | Carving | foremost, scalpel, PhotoRec, bulk_extractor-rec |

This article starts with finding strings and files in a memory image, then moves on to following the OS's data structures. Use leads from one approach to dig deeper with the other.


### Setting Up Your Environment

Once you've picked the approaches and tools you want to use, set up your analysis environment. I find it easier to match the analysis OS to the target.
For Windows targets, working on Windows tends to mean fewer problems with the analysis tools.
Linux can be nicer for working with strings, though, so something like [SIFT Workstation](https://www.sans.org/tools/sift-workstation) is also worth having around.

![sift](https://github.com/user-attachments/assets/7c6c83cd-5044-4781-ae52-60fad30ebeb0)


## Analysis Method 1: Working with the Raw Data

First, let's examine the memory image as a sequence of bytes. This is less about studying binary structures and more about dumping readable strings from an enormous mystery blob and seeing what you can figure out.

It's easy to get started, but finding a string doesn't tell you which process used it or why. **Use nearby strings and other clues to get your bearings.**
"I found the malware's name!" Great, except a closer look often reveals an antivirus signature file or something. Read the results with a healthy amount of suspicion.

### Extracting Strings

First things first: dump some strings.

I recommend [Sysinternals Strings](https://learn.microsoft.com/en-us/sysinternals/downloads/strings), which extracts both ASCII and Unicode strings by default.
A minimum string length of 6 or 8 with `-n` is a reasonable starting point. Adjust it if you're not finding anything.

```powershell
> .\strings64.exe -n 8 memory.raw > memory-strings-ascii-and-unicode.txt
```

If you're using GNU Strings on Linux, pay attention to the encoding. For example, here's how to extract ASCII and UTF-16LE strings separately:

```bash
$ strings -a -n 8 -t x memory.raw > memory-strings-ascii.txt
$ strings -a -n 8 -t x -e l memory.raw > memory-strings-utf16le.txt
```

Adding `-t x` records the file offset of each string. That's useful when you want to return to the original memory image and inspect the surrounding bytes.
`-e l` tells it to read the input as UTF-16LE.

#### Compressing the Output

Extract strings from an entire memory image and the text alone can run to several GB. Multiply that by a hundred machines, and keeping it all on your workstation gets awkward. Text compresses well, so **just pipe it into gzip** if you like.

```bash
$ strings -a -n 8 -t x memory.raw | gzip -c > memory-strings-ascii.txt.gz
```

Windows doesn't include gzip by default. Save the output first, then compress it with something like [7-Zip](https://www.7-zip.org/).

```powershell
> .\strings64.exe -n 8 memory.raw > memory-strings.txt
> .\7z.exe a -tgzip memory-strings.txt.gz memory-strings.txt
```

#### Showing Progress

You probably won't need this just to run strings against memory, but if not knowing the progress drives you nuts, try [Pipe Viewer](https://www.ivarch.com/programs/pv.shtml).

On Linux, feed the input through `pv` to see the amount processed, throughput, percentage complete, and estimated time remaining.

```bash
$ pv memory.raw | strings -a -n 8 -t x | gzip -c > memory-strings-ascii.txt.gz
```

Windows doesn't come with an equivalent. Oh well.


### Searching Strings

Once you've extracted the strings, start looking for leads. If you have indicators of compromise (IOCs), such as a suspicious domain or filename, search for those values.
grep works, but [ripgrep](https://github.com/burntsushi/ripgrep) is ridiculously fast.

```bash
$ rg -i -F 'malicious.example.com' strings.txt
```

Some useful options:

| Option | Description |
| --- | --- |
| `-i` | Ignore case |
| `-F` | Match fixed strings rather than regular expressions |
| `-f FILE` | Read one search pattern per line from a file; handy for a list of IOCs |
| `-A NUM` | Show the specified number of lines after a match |
| `-B NUM` | Show the specified number of lines before a match |
| `-C NUM` | Show the specified number of lines on each side of a match |
| `-o` | Show only the matching portion |

For example, if `ioc.txt` contains one IOC per line, this displays matches with three lines of surrounding context.
Remember that strings close together in physical memory don't necessarily belong to the same process or the same point in time.

```bash
$ rg -i -F -f ioc.txt -C 3 strings.txt
```

For gzip-compressed output, zgrep works. [ripgrep-all](https://github.com/phiresky/ripgrep-all) also lets you search compressed files much like you would with ripgrep.

```bash
$ rga -i -F 'malicious.example.com' strings.txt.gz
```


#### Pattern Searches with bstrings

Even without specific IOCs, you can look for recognizable formats such as email addresses or URLs. [bstrings](https://github.com/EricZimmerman/bstrings) can search strings using a set of common regular-expression patterns.
List the available patterns like this:

```powershell
> .\bstrings.exe -p
```

Pick whichever pattern you need from the list. It works on both binary and text files. For example, to find email addresses in strings you've already extracted:

```powershell
> .\bstrings.exe -f strings.txt --lr email
```

These are some of the ones I use often:

| Pattern | Description |
| --- | --- |
| b64 | Base64 strings |
| bitcoin | Bitcoin wallet addresses |
| bitlocker | BitLocker recovery keys |
| cc | Credit card numbers |
| email | Email addresses |
| guid | GUIDs |
| ipv4 | IPv4 addresses |
| ipv6 | IPv6 addresses |
| mac | MAC addresses |
| reg_path | Paths related to registry hives |
| sid | Security identifiers (SIDs) |
| unc | UNC paths |
| url3986 | RFC 3986 URLs |
| win_path | Windows file paths |
| zip | US ZIP codes |


#### Pattern Searches with bulk_extractor

If you don't have IOCs yet and want to collect URLs, email addresses, and similar data all at once, try [bulk_extractor](https://github.com/simsong/bulk_extractor). It scans the input as bytes without interpreting a filesystem, so memory images work as input too.

```bash
$ bulk_extractor -o ./bulk memory.raw
```

The main outputs are listed below. Which files appear depends on the enabled scanners and the data they actually find. See [forensics.wiki](https://forensics.wiki/bulk_extractor/) for details.

| Output file | Description |
| --- | --- |
| ccn.txt | Credit card numbers |
| domain.txt | Internet domains |
| email.txt | Email addresses |
| ip.txt | IP addresses |
| telephone.txt | US and international telephone numbers |
| url.txt | URLs |
| url_searches.txt | Web search terms extracted from URLs. These are surprisingly useful. |
| wordlist.txt | Candidate words, useful for password cracking and similar tasks |
| zip.txt | Information about ZIP files. Useful for Office documents too, since many Office formats are ZIP-based. |

You'll use these findings in other searches and when investigating processes, so keep them organized. Generalize them into reusable patterns, and they can become powerful tools for future investigations.


#### Finding Other Scripts and Languages with langscan

Besides matching formats, you can narrow things down by the language used in the text. [langscan](https://github.com/sumeshi/langscan) searches UTF-8 text for characters used in particular languages, such as Cyrillic or Japanese. Convert the text to UTF-8 first if needed.

```bash
$ langscan strings-utf8.txt
```

If you're thinking, "I just want the Cyrillic stuff," do this:

```bash
$ langscan --lang ru strings-utf8.txt
```

Keep in mind that executables often contain multilingual support code and resources, which can produce a lot of noise.
Use it on a narrower target, such as an individual process memory dump created using the methods discussed later, or as a quick way to narrow things down.


### YARA Searches

If a simple string search isn't finding what you need, or you know the malware family but have no idea what to search for, try [YARA](https://github.com/VirusTotal/yara). It lets you search with rules that specify byte sequences or combine multiple conditions.

Google something like `{malware-family} yara rule` and you'll find plenty. Customize the rules as needed. For a rule collection, try [yara-rules/rules](https://github.com/yara-rules/rules).

The Rust implementation, [YARA-X](https://github.com/virustotal/yara-x), has seen more active development recently. Just watch out for rules that depend on modules it doesn't support.


### File Carving

So far, we've looked for strings and regions matching search conditions, but sometimes you want to extract the files themselves. Carving means finding files or fragments by recognizing file headers and characteristic record structures.

In memory, a file may only have been partially loaded, or its pages may be physically scattered. Incomplete recovery is common. Treat even a piece of an image as a nice bonus.

#### foremost

An old-school tool. [foremost](https://github.com/korczis/foremost) extracts files based on headers, footers, and other recognizable features. To look for JPEGs and PNGs:

```bash
$ foremost -t jpg,png -i memory.raw -o out
```


#### scalpel

[scalpel](https://github.com/sleuthkit/scalpel) is another well-known choice that builds on and improves upon foremost.

Make a working copy of the supplied `scalpel.conf`, then uncomment only the format definitions you want to search for. They're all disabled in the default configuration, so do this before running it.

```bash
$ scalpel -c scalpel.conf -o scalpel-out memory.raw
```


#### PhotoRec

[PhotoRec](https://www.cgsecurity.org/wiki/PhotoRec) is another staple.
The icon looks a bit sketchy, but it's an established tool that's even integrated into [Autopsy](https://sleuthkit.org/autopsy/docs/user-docs/4.20.0/photorec_carver_page.html). Apparently it supports over 400 formats.

It comes bundled with TestDisk. Download the package and run `qphotorec_win.exe` for the GUI.

![photorec](https://github.com/user-attachments/assets/d7fc351b-6f87-4643-b7fb-dae700a616e5)


#### bulk_extractor-rec

[Bulk Extractor with Record Carving](https://www.kazamiya.net/en/bulk_extractor-rec) adds record recovery scanners to bulk_extractor. It can target EVTX files and chunks, MFT records, USN journal records, and more. Even if the full file is gone, individual records may still be recoverable.

Open BE Viewer and choose **Tools → Run Bulk Extractor** to select an input file.
It can pick up quite a few event logs and other artifacts that PhotoRec missed.

![bulkextractor](https://github.com/user-attachments/assets/99041bfd-5aee-47a8-9e90-76ba239afc60)


## Analysis Method 2: Following OS Data Structures

Next, we follow the OS's data structures. This lets us connect leads from string searches and other methods to processes and network connections.

If you already know the name of a suspicious process, check whether it ran. If you don't have any candidates, dump the process lists, command lines, and network connections first, then review them later for leads.

[MemProcFS](https://github.com/ufrisk/memprocfs) and Volatility are the usual names here. We'll use both in the following sections. The SANS [Memory Forensics Cheat Sheet](https://www.sans.org/posters/memory-forensics) is also handy to keep nearby.


### MemProcFS

MemProcFS's forensic mode is useful when you want to browse the files and artifacts recoverable from memory in one place.

#### Mounting an Image

On Windows, follow the [Wiki](https://github.com/ufrisk/MemProcFS/wiki) to set up Dokany and the other prerequisites, then mount the image on an unused drive letter, usually `M:`.

Enable forensic mode at startup with `-forensic`. This helps make results reproducible for the same image, settings, and MemProcFS version. Enabling it after mounting can produce differences due to caching and processing order.

Choose a mode for `-forensic` based on how you want it to handle the SQLite database containing the analysis results:

| Mode | Behavior |
| --- | --- |
| 1 | Create an SQLite database in memory only |
| 2 | Create a temporary database file and delete it when MemProcFS exits |
| 3 | Create a temporary database file and keep it after MemProcFS exits |
| 4 | Create a database with a fixed filename (`vmm.sqlite3`) and keep it after MemProcFS exits |

Here, we'll use mode 4 to keep the database after MemProcFS exits.

```powershell
> .\MemProcFS.exe -device C:\Cases\memory.raw -mount M -forensic 4
```

If you have page files acquired around the same time, you can add them at startup using the corresponding index numbers. Each page file has an index. In a standard Windows 10 configuration, `pagefile.sys` gets 0 and `swapfile.sys` gets 1.
Those numbers may differ if page files have been added or reconfigured, so check the target's configuration too.

```powershell
> .\MemProcFS.exe -device C:\Cases\memory.raw -pagefile0 C:\Cases\pagefile.sys -pagefile1 C:\Cases\swapfile.sys -mount M -forensic 4
```

![mount](https://github.com/user-attachments/assets/a81eeb9f-84a6-4941-88a7-d30c1c79be6d)

After mounting, check `M:\forensic\database.txt` for the database location. Mine was here:

```
C:\Users\example\AppData\Local\Temp\vmm.sqlite3
```

Analysis progress is recorded in `M:\forensic\progress_percent.txt`. Wait until it reaches 100 before examining the results.

The mounted drive contains folders like these:

| Folder | Description |
| --- | --- |
| conf | MemProcFS state and configuration |
| forensic | Forensic results. Really important. |
| misc | Other plugins, such as [phys2virt](https://github.com/ufrisk/MemProcFS/wiki/FS_Phys2Virt) for finding virtual addresses corresponding to a physical address, and [bitlocker](https://github.com/ufrisk/MemProcFS/wiki/FS_BitLocker) for recovering BitLocker keys |
| name | Process information, organized by name |
| pid | Process information, organized by process ID |
| py | Python plugins. Install something like [pypykatz regsecrets](https://github.com/ufrisk/MemProcFS-plugins) and it appears here. |
| registry | Registry hives, which may be incomplete or corrupt due to paging and other causes |
| sys | System-wide information about the OS, users, processes, networking, and more |
| vm | Detected Hyper-V VMs, Windows Sandbox, WSL2, and similar environments. VMware / VirtualBox support covers configurations running on Hyper-V. |

Start with the system-wide information and analysis results, then move on to individual processes and the registry. Let's take a quick look at the main folders.

#### sys

All sorts of system information lives here, including the time zone, OS version, and computer name. A good place to start.

![sys](https://github.com/user-attachments/assets/4261220b-6288-40f5-acf6-ca9f17c8cd1f)


After checking the basics, look at the process tree in `proc/proc.txt`.

![proc](https://github.com/user-attachments/assets/7c4b011d-f64c-4e71-bae3-eee311c4dce5)

Similarly, `users/users.txt` lists users, `tasks/tasks.txt` lists scheduled tasks, and `net/netstat.txt` shows network connections.
Have a look through everything, find a lead, then dig deeper.


#### forensic

This contains data organized for forensic analysis.

![forensic](https://github.com/user-attachments/assets/8d10ed53-bd37-4254-9439-e018e5907a10)

I'd start with `csv`, `files`, and `ntfs`.

[csv](https://github.com/ufrisk/MemProcFS/wiki/FS_Forensic_CSV) holds analysis results for processes, network connections, and other artifacts as CSV files. Open them in something like Timeline Explorer for easier reading.
Results from `findevil` and `yara` are collected here too, making it a convenient starting point.

![csv](https://github.com/user-attachments/assets/06994dc4-c49f-4ac9-b7bb-ca0af24966d4)

To browse recovered files, look at the list in `M:\forensic\files\files.txt`. If something catches your eye, grab it from under `files` and analyze it. The folders are reconstructed along their original paths, so things are easy to find.

If you can't find a file under `files`, try `ntfs`. On NTFS, a small file's contents may be stored directly in its MFT record.


#### name / pid

Once a process catches your eye, open `name` or `pid` to examine it. Both contain process information; one organizes it by name, the other by PID. Entries under `name` also have the PID appended, so that view may be easier to browse.

![name](https://github.com/user-attachments/assets/6984680a-fa98-40bd-a6de-84451325e76c)

![pid](https://github.com/user-attachments/assets/d09a5fd7-b4f7-495f-b092-4315ea7eac7c)

`name-long` is the process name, `pid` is the process ID, `ppid` is the parent process ID, `time-create` is the creation time, `win-cmdline` is the command line, and `win-environment` contains environment variables. Pretty much what the names say.

To examine related files, start with something like `files/handles`. It contains files reconstructed using the process's open file handles.

`files/modules` contains EXEs, DLLs, and other modules reconstructed from memory. `files/vads` contains files reconstructed using VADs (Virtual Address Descriptors). A VAD describes a region of a process's virtual memory, including its protection attributes and associated file, if any.

Wherever you recover a file from, the whole thing may not have been in memory. If it opens as-is, take the win.

For example, search for `.evtx` and you may find event logs. Some events might be recoverable even if the logs were deleted from disk. If you find any, [dig into them](https://sumeshi.github.io/posts/knowledges/windows-eventlog-analysis-101-en).

![evtx](https://github.com/user-attachments/assets/6a43c9d8-1567-4f5d-8e92-259d42a39592)

![others](https://github.com/user-attachments/assets/8e40de73-f7f4-4efd-9d2c-9f88660b8b2a)


#### registry

Registry hives, as the name suggests. You'll find both hive files and parsed text.

Registry changes are applied to the in-memory hives and written back to disk using transaction logs (`.LOG1` and `.LOG2`). The copy in memory may therefore be newer than the one on disk. Worth a look.

![registry](https://github.com/user-attachments/assets/ad63c942-d688-4e79-a2ca-0093020020d9)

Personally, I find it easier to browse the hive files in something like Registry Explorer. The hives are often broken, though.

![regexp](https://github.com/user-attachments/assets/533a8a97-ae71-4abe-b6a2-8e86bdd6db19)


### Volatility

This is probably the first tool people think of for memory forensics, but some plugins take so bloody long you'll burst a blood vessel. MemProcFS is faster for a quick look around.

Still, Volatility has a broader plugin selection. Start a job, go get dinner. That sort of mindset.

You'll run into both Volatility 2 and 3. For a recent OS, go with 3. Every now and then, someone hands you an archaeological find that might only work with 2. Keeping both around is a good idea.

Until you're used to Volatility, wrappers such as [Volatility Workbench](https://www.osforensics.com/tools/volatility-workbench.html) and [KaniVola](https://github.com/4n6ist/KaniVola) (documentation in Japanese) make life much easier. I get it, hammering away at commands feels good, but you rarely have that kind of time during an actual incident.

Volatility Workbench is very easy to use with version 3.

![vwork](https://github.com/user-attachments/assets/1ccd3179-596f-47e0-ba29-07303ab4128b)

For version 2, KaniVola is a good choice.

![kanivol](https://github.com/user-attachments/assets/deba30e7-c826-47c6-93a6-04b8d5d574f1)

There's also a fast Rust implementation called [vol-rs](https://github.com/daffainfo/vol-rs). It might be worth a shot for CTFs. It isn't a mature product yet, so I'd want to evaluate and validate it before using it on a real case.

The following examples use Volatility 3 from the command line. In Volatility Workbench, select the corresponding plugin and set options such as the PID.

Here, `vol3.py` stands for your Volatility 3 launcher. Depending on your installation, replace it with `vol` or `python vol.py`.


#### Symbols

Volatility 3 uses symbols to interpret Windows structures in memory. If the required symbols aren't available locally, it downloads them from Microsoft's servers, which requires an internet connection.

I'm an absolute offline fanatic, so I prepare the symbols locally beforehand.
JPCERT/CC's [How to Use Volatility 3 Offline](https://blogs.jpcert.or.jp/en/2021/09/volatility3_offline.html) is an excellent reference.

Writing a script to prepare the symbols automatically can save you trouble during an incident.

Volatility 2, on the other hand, uses `--profile` to select a profile containing structure definitions and other information for the target OS. Choose the wrong profile and it may appear to work while interpreting values incorrectly.

#### Basic Information

Start with `windows.info` to check whether the OS and kernel information can be read. If this fails, check the acquisition format, missing data, and symbol availability before moving on.

```bash
$ vol3.py -q -f memory.raw windows.info > info.txt
```

#### Listing Processes

Once you can read the basic information, record the running processes, their parent-child relationships, and their command lines.

```bash
$ vol3.py -q -f memory.raw windows.pslist > pslist.txt
$ vol3.py -q -f memory.raw windows.pstree > pstree.txt
$ vol3.py -q -f memory.raw windows.cmdline > cmdline.txt
```

`pslist` walks the OS's process list. `pstree` displays the same enumeration as a tree of parent-child relationships.

A process name alone won't tell you whether something is suspicious. Look at the executable path, parent process, arguments, and user together. Dig into processes whose names mimic legitimate ones, and executables launched from [commonly abused directories](https://attack.mitre.org/techniques/T1074/001/) such as `C:\Users\Public`.

A parent process may already have exited and be missing from the list. If its PID has been reused, you might mistake a different process for the parent. Check creation times when reading the tree.

To look for processes missing from the list, use `psscan`. It scans kernel pool memory for process structures, so it may find terminated or hidden processes. It also takes a while. Leave it running in the background, grab a coffee, and look through `pslist` in the meantime.

```bash
$ vol3.py -q -f memory.raw windows.psscan > psscan.txt
```

`psxview` compares several process enumeration methods. It can show you things like a process appearing in `psscan` but missing from `pslist`.

```bash
$ vol3.py -q -f memory.raw windows.malware.psxview > psxview.txt
```


#### Network Connections

Alongside the process list, check network connections. Use `netscan` to examine network structures.

```bash
$ vol3.py -q -f memory.raw windows.netscan > netscan.txt
```

Match the PIDs you find against the process list and creation times, then investigate those processes further. Structures for closed connections or freed objects may survive, so read `State` and `Created` too. `Created` is the network object's creation time.

Keep in mind that `netscan` doesn't tell you the contents of communications or how much data was transferred.
To work out what actually passed between the endpoints, expand the investigation to proxy logs, DNS logs, firewall logs, EDR telemetry, and other sources.


#### Examining a Process

Once the process list or network connections have helped you narrow things down to a PID, say `4240`, look at what the process loaded and referenced.

Here we'll examine DLLs and handles as well as the command line. Handles are identifiers used to reference objects such as files, registry keys, and other processes. They can give you more leads to follow.

```bash
$ vol3.py -f memory.raw windows.cmdline --pid 4240
$ vol3.py -f memory.raw windows.dlllist --pid 4240
$ vol3.py -f memory.raw windows.handles --pid 4240
```

Use `dlllist` to examine the paths and locations of loaded DLLs, and `handles` to see referenced files, registry keys, and other objects. Don't confuse `dlllist` with `windows.modules`, which lists kernel modules.

For example, if the command line mentions a script in a temporary directory, search for or try to recover that file. If a handle points to a document, also check the disk for the file and traces of access. **A handle alone doesn't prove that the process read the entire file or sent it outside the system.**


#### Command History

To follow what happened after a process started, check command history too. Use `cmdline` for process launch arguments and `cmdscan` for input history remaining in a console. If you want to know what someone typed after launching `cmd.exe`, try the latter too.

```bash
$ vol3.py -q -f memory.raw windows.cmdscan > cmdscan.txt
```

You can only recover what remains in supported console history structures, so **this won't reconstruct every shell command**. For PowerShell history and script execution, cross-reference PSReadLine history files, PowerShell logs, and other records.


#### Suspicious Executable Regions

To look for suspicious code in a process's memory, use `malfind`. This plugin uses VAD attributes and other clues to find suspicious executable memory regions. It's useful when looking for traces of code executing inside another process without leaving a file on disk.

```bash
$ vol3.py -f memory.raw windows.malware.malfind --pid 4240
```

Look at the addresses, protection attributes, initial bytes, and disassembly to decide which regions to examine further. Legitimate activity such as JIT compilation also produces hits, so inspect the contents and related modules.

You can extract candidate regions with `--dump`. We'll look at an example in "Extracting PE Files and Memory" below.


#### Searching Within Processes

You can also work from a string back to a process. If an earlier string search found a domain but you don't know which process it's associated with, put the string into a YARA rule and search the processes' virtual memory.

```bash
$ vol3.py -f memory.raw windows.vadyarascan --yara-file ioc.yar
```

`vadyarascan` follows VADs to search virtual memory regions belonging to processes.
A hit establishes that the IOC was present in a readable region of that process. To work out how it was used, cross-reference other artifacts.


#### Extracting PE Files and Memory

Once you've chosen a process or region to investigate, extract it to a file if needed. Choose the extraction method based on whether you want an executable in PE format or memory that includes heaps and other regions for string searches.

Create the output directories first.

```bash
$ mkdir -p 4240/pe 4240/pages 4240/suspicious

# Dump the process executable image as a PE
$ vol3.py -f memory.raw -o 4240/pe windows.pslist --pid 4240 --dump

# Extract readable process memory and save the address mapping
$ vol3.py -q -f memory.raw -o 4240/pages windows.memmap --pid 4240 --dump > memmap-4240.txt

# Extract the regions detected by malfind
$ vol3.py -f memory.raw -o 4240/suspicious windows.malware.malfind --pid 4240 --dump
```

A dumped PE usually differs from the original file, so you generally can't identify the original sample by looking up the dump's hash on VirusTotal or a similar service.
Getting it to run is difficult too. ~~You may need to rebuild the IAT and so on, but that's outside the scope of this article.~~

Still, applying the string extraction and YARA searches described earlier to the extracted data can give you useful leads.

For a closer look at a PE, you can also use [FLOSS](https://github.com/mandiant/flare-floss). It analyzes executable code and attempts to extract obfuscated strings and strings assembled at runtime.
It's useful for finding strings that ordinary strings tools won't show you.

Feed it the dumped PE. Missing data or damaged headers may prevent FLOSS from analyzing it.

```bash
$ floss recovered.exe
```


#### Recovering Files

Besides process executable images and memory regions, you can try recovering files used by processes and data in the OS's caches. If you know the name or path of a file you're interested in, try recovering it, then feed the result to a parser for that format.

`filescan` scans for the `FILE_OBJECT` structures Windows uses to track files. Find candidates by filename, then use `dumpfiles` to recover the contents associated with those objects.

```bash
$ vol3.py -q -f memory.raw windows.filescan > filescan.txt
```

For Windows 10 / 11, the following example supplies the virtual address of a discovered `FILE_OBJECT`. Replace the address with one from your actual output.

```bash
$ mkdir -p out/cache
$ vol3.py -f memory.raw -o out/cache windows.dumpfiles --virtaddr 0xffff800012345670
```

`--virtaddr` and `--physaddr` take a `FILE_OBJECT`'s virtual and physical address, respectively.

If you've linked the file to a process, you can also narrow down recovery candidates by PID.

```bash
$ vol3.py -f memory.raw -o out/cache windows.dumpfiles --pid 4240
```

Finding a name with `filescan` doesn't guarantee that the file's contents are still there.
If a parser can't open the recovered data, try string searches or carving against what you managed to recover.


## Organizing the Results

Memory forensics tends to leave you with a complete mess of files. Separate things into folders by host and by the process you extracted them from, and try to keep it all vaguely organized.
MemProcFS also produces timelines and CSVs, so copy those out before unmounting.

When comparing results, you'll often find that different tools disagree.
Before deciding that one of them is wrong, check what each tool is following to enumerate the data. Different structures, handling of terminated objects, or treatment of missing pages can produce different results.

Organizing all these results is a pain. If you can't be bothered, I think it's fine to [dump that job on AI](https://sumeshi.github.io/posts/works/dont-make-ai-your-forensic-analyst-en). Just **be extremely careful about how you handle the data**.


## Closing Thoughts

Compared with disk forensics, memory forensics involves **a lot more incomplete or broken data**. What you can do with it often comes down to experience and a feel for what you're looking at, so I can't give you a neat answer.

When stuck, try a different way of looking at the data you have. Read it as strings, or if it looks like an image fragment, chuck it into GIMP. If you're really stuck, I think asking AI for help is fair game too.

Memory forensics really isn't a job for humans.
