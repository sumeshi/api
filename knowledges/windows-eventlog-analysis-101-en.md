# How Do You Investigate Windows Event Logs?

Vague notes on Windows event log investigation. I'll add more as I remember.

<img width="960" height="540" alt="win" src="https://github.com/user-attachments/assets/519ecf99-7a14-40a1-8e17-cc8690da1e86" />

## Introduction

Because I wanted something like a Japanese-language document for doing Windows event log investigations.  
Mainly from an **incident response perspective**.

The purpose of event log investigation is **not to comprehensively scoop up every suspicious event ID, but to grasp what happened in the target environment and connect it to initial response and further investigation**.  
Given that premise, keep the following in mind at minimum.

- Event logs only tell you about what's still around.
    - Quite a lot of events aren't recorded by default. Depends on the [audit policy](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/audit-policy-recommendations?tabs=winclient).
    - Old event logs get overwritten. Depends on the [rotation settings](https://learn.microsoft.com/en-us/security-updates/planningandimplementationguide/19869709).
    - Logs can also be deleted by the attacker. [There are even cases where deletion is automated in ransomware incidents](https://www.cybereason.co.jp/blog/threat-analysis-report/10544/), so sometimes you can't tell a thing from the surviving logs alone.
    - The possibility of log tampering can't be ruled out either. EVTX has a [checksum mechanism for integrity checking](https://github.com/libyal/libevtx/blob/main/documentation/Windows%20XML%20Event%20Log%20(EVTX).asciidoc), but it's not a cryptographic tamper-protection mechanism. It costs more than simple deletion, so you don't see it much.
    - From the above, it's important for an analyst to **not treat surviving event logs as standalone facts**, but to **build and verify hypotheses about the attacker's actions** while cross-referencing other artifacts.
- Preserve logs first thing
    - Even while you're reading this article, new events are being recorded and old ones overwritten.
    - The Security log, which is especially important evidence, gets overwritten fast. You're lucky if a few days' worth survives, and sometimes only a few hours' worth is left. ([Like when it's buried under logon failures.](https://www.softwareisac.jp/a2ad/log-analysis-4625/))
    - If you've got time to hesitate, preserve first.
- How to cut noise out of a massive pile of logs? Hugely important.
    - Be prepared to look at a few GB per client machine, and tens to hundreds of GB per server.
    - Understanding how the device is normally used and what events it records is the first step in cutting noise.
    - "Just look at everything, you say?" I will punch you.


> **Note:** This is the English translation of the Japanese original. The Japanese version is available at https://sumeshi.github.io/posts/knowledges/windows-eventlog-analysis-101.
> The snark is preserved as-is. Deal with it.

## Purpose

**You can't investigate logs without knowing what you're trying to confirm.**  
Break down from the big goal and clarify the investigation's purpose.

Whether it's [5W1H](https://en.wikipedia.org/wiki/Five_Ws) or the [Cyber Kill Chain](https://www.lockheedmartin.com/en-us/capabilities/cyber/cyber-kill-chain.html) or whatever, organizing around some framework makes it harder to miss things.

- Did an intrusion occur? (Detection)
- Where did they get in? (Initial access)
- How did it spread? (Lateral movement / Persistence)
- What did they do? (Activity)
- What was affected? (Impact scope)
- Why was it possible? (Cause)

Stuff like that.

Well, the reason you want to investigate is usually "whether there was an intrusion", and if there was, "whether there was data leakage" or something along those lines.  
So what do you need to look at to investigate that? What's recorded in logs is ultimately concrete events, so you have to make a high-confidence guess from circumstantial evidence about what happened from the chain of those events.  

For example, **if you want to know "whether there was data leakage", the event log is never going to literally say "data was leaked".** What you should look at is the **traces of actions that could lead to leakage** — suspicious logons, file access, external communication, removable media connections, access to admin shares, execution of suspicious tools, and so on.


## Cutting Scope

Log investigation has no end (seriously).  
**Resources vanish in proportion to how much you throw at it**, so cutting scope up front is important.  

If the results show the scope was too narrow, you can think about it then.

- Which hosts' logs to investigate? (Servers? Clients?)
- Which time period? (The intrusion period?)
- Which logs? (Security? System? Application? PowerShell? TaskScheduler?)
- Which users' logs? (Admins? Regular users? Service accounts?)
- Which event IDs? (Logon events? Process creation events? Network connection events?)


## Knack for Investigation

For starters, go through the major event IDs once.  
After that, build hypotheses based on them and think about what to investigate.

Interpreting logs does rely on experience to some extent... But basically it's best to refer to reliable official docs like [MS Learn](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4624).  

Also, if you have [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) you can look at finer events too,,, but in most environments it isn't installed. Don't get your hopes up.

The [Security Log Encyclopedia](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/) is also super helpful. There are a lot of undocumented events, so if you're curious about an event ID, look it up there.

If you still can't find it, sometimes you can find it on [MyEventlog](https://www.myeventlog.com/search/show/980). Surprisingly, EventID is an insane spec that isn't a unique identifier, but this site properly shows search results per Provider/Channel.

Even more niche ones are sometimes written up in the [exabeam Documentation](https://docs.logrhythm.com/?l=en). If it's still not there, you have no choice but to Google like crazy.

If you know the tools or techniques used, it's good to look for traces while referring to things like JPCERT/CC's [Tool Analysis Result Sheet](https://jpcertcc.github.io/ToolAnalysisResultSheet_jp/).

You can also look at Yamato Security's [Guide to Windows Event Log Settings for DFIR and Threat Hunting](https://github.com/Yamato-Security/EnableWindowsLogSettings/blob/main/README-Japanese.md). It references [Sigma](https://github.com/sigmahq/sigma) and discusses what the default event log settings look like. If possible, look at the original [Sigma](https://github.com/sigmahq/sigma) rule definitions, but there are so many you couldn't possibly memorize them.


## Log Analysis and Hypothesis Building

Once you've looked at the logs to some extent, build hypotheses about what might have happened, what attack techniques are conceivable, and what damage is conceivable.  
To verify those hypotheses, dig deeper into what other events to look for and what artifacts to cross-reference.

If you don't know how to investigate, you can ask generative AI. But don't paste logs as-is. Since they may contain sensitive info, always mask the input and follow your organization's rules.  
Also, if you ask a slightly deeper question, keep in the back of your mind that the accuracy isn't that great (or rather, it leans too hard toward generalities).

## Reporting

Report what might have happened as a result of the investigation and what to do next.  
Or, if that's not the analyst's job, it's good to clarify where another team's territory begins. **In incident response, pushing nasty work onto each other** happens often enough, so negotiating it in advance makes things smoother.

Also, for the materials you hand to the report recipients, you have to hammer them into your brain enough that you can explain everything no matter what you're asked.
For everything else, just keep materials on hand so you can answer if asked.

If there's a second opinion involved or the customer has security expertise, you might get asked "Did you check ◯◯?". Resources are limited so you don't need to prepare for everything, but **be able to explain why you didn't check that item.** "I excluded it from the investigation for this and that reason. It's out of contract scope." and so on.

Also, it's fine to narrow down what you report. Rather, **there is absolutely no need to report every hypothesis conceivable from the investigation results.**  
To avoid needlessly worrying or confusing the other party with baseless hypotheses, it's enough to focus on the highest-confidence hypothesis and the investigation you did to verify it.


## Appendix. Major Event IDs

Rather than looking at logs comprehensively, it's better to look at them from the perspective of **finding a starting point**.  
Once you find a suspicious event as a starting point, diligently tracing the timeline before and after it should turn up something useful even for unknown events.

### Logon / Authentication

Look at who, when, from where, logged onto which machine (lateral movement).  
Start investigating from the device where the problem occurred, or the bastion server that's the entry point from the internet.

When looking, be conscious of whether the recorded event is from the logon source or the logon destination.

- Security.evtx
    - [4624](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events): A user successfully logged on to a computer
    - [4625](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events): Logon failure
    - [4634](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events): User logoff completed
    - [4647](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events): User logoff initiated
    - [4648](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events): A logon was attempted using explicit credentials while the user was already logged on as a different user
    - [4672](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4672): Special privileges assigned to a new logon
    - [4673](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4673): A privileged service was called
    - [4674](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4674): An operation was attempted on a privileged object

The [LogonType](https://learn.microsoft.com/en-us/windows-server/identity/securing-privileged-access/reference-tools-logon-types) of event ID 4624 is extremely important. Basically you'll look mainly at 3 (Network) and 10 (RemoteInteractive), with 9 (NewCredentials) and 12 (CachedRemoteInteractive) as support.

For logon failures in event ID 4625, the [logon error code](https://www.softwareisac.jp/wp/active-directory-domain-controller-security-log/) tells you why it failed. Like the username doesn't exist in the first place, or the username is right but the password is wrong.. This tells you how much information the attacker had at the time they did it. For example, if they logged on successfully on the first try, they're dumping passwords somewhere or reusing the same one.

| Logon Type | Description |
| --- | --- |
| [0](https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-logonsession) | **Logon type used only by the SYSTEM account.** |
| 1 | **No information.** On [Reddit](https://www.reddit.com/r/windows/comments/18wqzs8/what_is_or_was_logon_type_1/), the rumor is it's a leftover from the NT 3.x era. |
| [2](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **Interactive. Logon where a user interactively uses the machine.** Console logon, RUNAS, remote shell, KVM, operation via Lights-Out cards, IIS Basic Auth (before 6.0), etc. |
| [3](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **Network. Logon to access a target over the network.** Shared access via `net use`, MMC snap-in to a remote computer, PowerShell WinRM, PsExec, Remote Registry, Remote Desktop Gateway, vulnerability scanners, IIS Integrated Windows Auth, SQL Windows Auth, etc. `LogonUser` does not cache credentials for this logon type. Reusable credentials, as a rule, do not remain in the destination LSA session, but watch out for exceptions like when Kerberos delegation is enabled. PsExec, when explicit credentials are specified, can create both Network and Interactive logon sessions. |
| [4](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **Batch. Logon for running a process on behalf of a user without their direct operation.** Scheduled tasks, etc. Also used for high-performance server purposes like mail/web servers that process many plaintext auth attempts at once. `LogonUser` does not cache credentials for this logon type. However, in scheduled tasks, the password may be stored on disk as an LSA Secret. |
| [5](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **Service. Logon by a service.** The target account needs the "Log on as a service" right. Credentials for running the service can remain in the LSA session as reusable credentials. Also, the password may be stored on disk as an LSA Secret. |
| [6](https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-logonsession) | **Proxy (proxy logon). Officially described as a proxy-type logon.** On [Reddit](https://www.reddit.com/r/windows/comments/18wqzs8/what_is_or_was_logon_type_1), the rumor is it's for internal/dev builds only. |
| [7](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **Unlock. Workstation unlock.** A logon type for recording the unlock of a user interactively using the machine, like via GINA DLL. |
| [8](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **NetworkCleartext (network plaintext auth). Holds the name and password in the auth package, and the server can connect to other network servers while impersonating the client.** IIS Basic Auth (6.0+), PowerShell WinRM with CredSSP, etc. Reusable credentials remain on the destination side, so the credential-theft risk is high. |
| [9](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **NewCredentials. Clones the current token and specifies different credentials for outbound network connections.** Locally keeps the original identity, and only uses the specified credentials for network connections. `RUNAS /NETWORK`, etc. Reusable credentials can remain in the LSA session, so be careful. |
| [10](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **RemoteInteractive (remote interactive). Remote and interactive terminal services session. Remote Desktop, etc.** Not only successful RDP logons, but 4625 logon failures can also be recorded as RemoteInteractive. Reusable credentials remain in the destination LSA session, so privileged account RDP to a compromised machine is dangerous. |
| [11](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **CachedInteractive (cached interactive). Interactive logon using cached credentials without accessing the network.** Not necessarily a logon that authenticated by querying a domain controller. |
| [12](https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-logonsession) | **CachedRemoteInteractive (cached remote interactive). Same as RemoteInteractive. For internal audit purposes.** |
| [13](https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-logonsession) | **CachedUnlock (cached unlock). Workstation unlock using cached credentials.** |


### Kerberos / NTLM

On the DC, look at which account tried to authenticate to which service.  
For single-device cases like abnormal-event incidents, you basically don't need to look.

- Security.evtx
    - [4768](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4768): A Kerberos authentication ticket (TGT) was requested
    - [4769](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration): A Kerberos service ticket was requested
    - [4770](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration): A Kerberos service ticket was renewed
    - [4771](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4771): Kerberos pre-authentication failed
    - [4776](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4776): The computer attempted to validate the credentials for an account (NTLM)
- Microsoft-Windows-NTLM%4Operational.evtx
    - [4020](https://support.microsoft.com/en-us/topic/overview-of-ntlm-auditing-enhancements-in-windows-11-version-24h2-and-windows-server-2025-b7ead732-6fc5-46a3-a943-27a4571d9e7b): Client - Outbound NTLM authentication attempt
    - [4021](https://support.microsoft.com/en-us/topic/overview-of-ntlm-auditing-enhancements-in-windows-11-version-24h2-and-windows-server-2025-b7ead732-6fc5-46a3-a943-27a4571d9e7b): Client - Outbound NTLM authentication attempt
    - [4022](https://support.microsoft.com/en-us/topic/overview-of-ntlm-auditing-enhancements-in-windows-11-version-24h2-and-windows-server-2025-b7ead732-6fc5-46a3-a943-27a4571d9e7b): Server - Inbound NTLM authentication attempt
    - [4023](https://support.microsoft.com/en-us/topic/overview-of-ntlm-auditing-enhancements-in-windows-11-version-24h2-and-windows-server-2025-b7ead732-6fc5-46a3-a943-27a4571d9e7b): Server - Inbound NTLM authentication attempt
    - [4030](https://support.microsoft.com/en-us/topic/overview-of-ntlm-auditing-enhancements-in-windows-11-version-24h2-and-windows-server-2025-b7ead732-6fc5-46a3-a943-27a4571d9e7b): Domain Controller - Forwarded NTLM authentication request
    - [4031](https://support.microsoft.com/en-us/topic/overview-of-ntlm-auditing-enhancements-in-windows-11-version-24h2-and-windows-server-2025-b7ead732-6fc5-46a3-a943-27a4571d9e7b): Domain Controller - Forwarded NTLM authentication request


### RDP / TerminalServices
RDP connection attempts, authentication, session start, disconnect, reconnect, etc.  
Even if the Security event log is wiped, RDP-related logs are often left. Sometimes you can trace the flow from here.

- Security.evtx
    - [4778](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4778): A session was reconnected to a window station
    - [4779](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4779): A session was disconnected from a window station
- Microsoft-Windows-TerminalServices-LocalSessionManager%4Operational.evtx
    - [21](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: Session logon succeeded
    - [22](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: Shell start notification received
    - [23](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: Session logoff succeeded
    - [24](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: Session has been disconnected
    - [25](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: Session reconnection succeeded
    - [39](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: Session disconnected by another session
    - [40](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: Session disconnected (with reason)
- Microsoft-Windows-TerminalServices-RDPClient%4Operational.evtx
    - [1024](https://jpcertcc.github.io/ToolAnalysisResultSheet_jp/): RDP ClientActiveX is trying to connect to the server
    - [1026](https://jpcertcc.github.io/ToolAnalysisResultSheet_jp/): RDP ClientActiveX has disconnected 
- Microsoft-Windows-TerminalServices-RemoteConnectionManager%4Operational.evtx
    - [261](https://jpcertcc.github.io/ToolAnalysisResultSheet_jp/): Listener RDP-Tcp received a connection
    - [1146](https://gist.github.com/MHaggis/138c6bf563bacbda4a2524f089773706): Remote Desktop Services: Session initialization succeeded
    - [1147](https://gist.github.com/MHaggis/138c6bf563bacbda4a2524f089773706): Remote Desktop Services: Session connection succeeded
    - [1148](https://gist.github.com/MHaggis/138c6bf563bacbda4a2524f089773706): Remote Desktop Services: Session connection failed
    - [1149](https://jpcertcc.github.io/ToolAnalysisResultSheet_jp/): Remote Desktop Services: User authentication succeeded (however, this alone doesn't confirm a successful RDP logon)
- System.evtx
    - [9009](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Desktop Window Manager terminated (e.g. RDP session disconnect)


### Process Execution
What, under whose privileges, launched from where, etc.  
If it's often disabled, count yourself lucky.

- Security.evtx
    - [4688](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4688): A new process has been created
    - [4689](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4689): A process has been terminated

4688 is recorded when "Audit Process Creation" is enabled.  
Furthermore, the command line is only recorded if you separately enable "Include command line in process creation events".


### PowerShell / Scripting
PowerShell startup, execution content, script blocks, logs, etc. It's settings-dependent, so if it's recorded, look at it as much as possible.  
Some are run on a schedule, so grasp the normal operation and do your best to cut noise.

- Windows PowerShell.evtx
    - [400](https://www.myeventlog.com/search/show/970): PowerShell engine started
    - [403](https://www.myeventlog.com/search/show/971): PowerShell engine stopped
- Microsoft-Windows-PowerShell%4Operational.evtx
    - [4103](https://www.myeventlog.com/search/show/977): Module logging
    - [4104](https://www.myeventlog.com/search/show/980): PowerShell script block logging
    - [4105](https://docs.logrhythm.com/devices/docs/evid-4105-4106): Script block execution started
    - [4106](https://docs.logrhythm.com/devices/docs/evid-4105-4106): Script block execution completed


### WMI
Event filters, consumers, etc. used for operations via WMI and for persistence.  
Lots of noise, so you don't need to look too hard.

- Microsoft-Windows-WMI-Activity%4Operational.evtx
    - [5857](https://docs.logrhythm.com/devices/docs/evid-5857-operation-started): A WMI operation started
    - [5858](https://learn.microsoft.com/en-us/troubleshoot/windows-client/system-management-components/wmi-activity-event-5858-logged-with-resultcode-0x80041032): WMI query/operation failure / result code


### Lateral Movement / Remote Operation
Look at traces of share access, admin shares, file operations over SMB, and remote operations.  
There are quite a few [cases of abuse](https://www.mbsd.jp/research/20210413/conti-ransomware/), so check it.

- Security.evtx
    - [5140](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-5140): A network share object was accessed
    - [5142](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-5142): A network share object was added
    - [5143](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-5143): A network share object was modified
    - [5144](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-5144): A network share object was deleted
    - [5145](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-5145): Detailed share object access
    - [5168](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-5168): SPN check for SMB/SMB2 failed
- Microsoft-Windows-SMBClient%4Connectivity.evtx
    - [30800](https://learn.microsoft.com/en-us/archive/msdn-technet-forums/1d001951-6985-48cb-be38-e33c60f4fc20): Cannot resolve server name
    - [30803](https://learn.microsoft.com/en-us/archive/msdn-technet-forums/ef3e9243-5a22-4020-97a0-219595666cd7): Network connection failed
    - [30804](https://www.microsoft.com/en-us/security/blog/2023/03/24/guidance-for-investigating-attacks-using-cve-2023-23397): Network connection was disconnected
    - [30805](https://live.paloaltonetworks.com/t5/globalprotect-discussions/gp-5-2-5-disconnects-in-connected-standby/td-p/389275): Client lost session with server
    - [30806](https://www.microsoft.com/en-us/security/blog/2023/03/24/guidance-for-investigating-attacks-using-cve-2023-23397): Client re-established session with server
    - [30807](https://live.paloaltonetworks.com/t5/globalprotect-discussions/gp-5-2-5-disconnects-in-connected-standby/td-p/389275): Connection to share was lost
    - [30809](https://learn.microsoft.com/en-us/troubleshoot/system-center/dpm/bare-metal-recovery-backup-fails): Request timed out because the server didn't respond
    - [31001](https://learn.microsoft.com/en-us/archive/msdn-technet-forums/d275b7b0-00d3-49b1-b921-54822187c504) Security context initialization failed
    - [31010](https://learn.microsoft.com/en-us/answers/questions/439044/win-2019-server-smb-session-authentication-failure) SMB client failed to connect to the share
- Microsoft-Windows-SMBClient%4Security.evtx
    - [32000](https://learn.microsoft.com/en-us/windows-server/storage/file-server/troubleshoot/smbv1-not-installed-by-default-in-windows) The connection target required SMBv1, but SMBv1 is disabled or not installed locally so the connection couldn't be made (the target is probably not Windows)
    - [32002](https://learn.microsoft.com/en-us/windows-server/storage/file-server/troubleshoot/smbv1-not-installed-by-default-in-windows) Received an SMBv1 negotiation response (the target is probably not Windows)
- Microsoft-Windows-SMBServer%4Security.evtx
    - [551](https://learn.microsoft.com/en-us/answers/questions/937522/smb-share-error-551-and-error-1009): SMB session authentication failure
    - [1006](https://learn.microsoft.com/en-us/answers/questions/1186193/continuously-available-smb-share-on-workgroup-clus): Share denied access from the client
    - [1009](https://learn.microsoft.com/en-us/answers/questions/937522/smb-share-error-551-and-error-1009): Server denied anonymous access from the client
- Microsoft-Windows-SMBServer%4Operational.evtx
    - [1016](https://learn.microsoft.com/en-us/answers/questions/1186193/continuously-available-smb-share-on-workgroup-clus): Reopen failed
    - [1020](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/troubleshoot-event-id-1020-warnings-file-server): Filesystem operation is taking longer than expected


### Service Modification
Look at service creation, start, stop, and startup type changes.  
Traces of PsExec-family tools, persistence, EDR/AV termination, backup product termination, etc. can be left.

- Security.evtx
    - [4697](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4697): A service was installed
- System.evtx
    - [7036](https://learn.microsoft.com/en-us/answers/questions/262317/whats-the-audit-event-id-for-windows-service-start): Service entered started/stopped state
    - [7040](https://learn.microsoft.com/en-us/answers/questions/262317/whats-the-audit-event-id-for-windows-service-start): The start type of a service was changed
    - [7045](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-7045): A service was installed on the system
  

### Scheduled Tasks
Look at task creation, deletion, enable, disable, and update.  
It's often used for malware persistence and delayed execution, so check the task name, execution command, creator, and creation time for anything suspicious.

- Security.evtx
    - [4698](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4698): A scheduled task was created
    - [4699](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4699): A scheduled task was deleted
    - [4700](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4700): A scheduled task was enabled
    - [4701](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4701): A scheduled task was disabled
    - [4702](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4702): A scheduled task was updated
- Microsoft-Windows-TaskScheduler%4Operational.evtx
    - [100](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Task started
    - [101](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Task failed to start
    - [102](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Task completed
    - [103](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Action failed to start
    - [106](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Task registered
    - [107](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Task triggered by scheduler
    - [108](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Task triggered by event
    - [110](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Task triggered by user
    - [111](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Task terminated
    - [118](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Task triggered on computer startup
    - [119](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Task triggered on logon
    - [129](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Task process created
    - [140](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Task registration info updated
    - [141](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Task registration deleted
    - [142](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Task disabled
    - [200](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Action started
    - [201](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Action completed
    - [203](https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/): Action failed to start


### Account / Group / Policy Modification
Look at account creation, deletion, enable, password change, group addition, and policy change.  
In cases where they were in for a long time, [weird accounts are often added](https://attack.mitre.org/techniques/T1136/).

- Security.evtx
    - [4719](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4719): System audit policy was changed
    - [4720](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4720): A user account was created
    - [4722](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4722): A user account was enabled
    - [4723](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4723): An attempt was made to change an account's password
    - [4724](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4724): An attempt was made to reset an account's password
    - [4726](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4726): A user account was deleted
    - [4728](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4728): A member was added to a security-enabled global group
    - [4729](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4729): A member was removed from a security-enabled global group
    - [4730](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4730): A security-enabled global group was deleted
    - [4731](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4731): A security-enabled local group was created
    - [4732](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4732): A member was added to a security-enabled local group
    - [4733](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4733): A member was removed from a security-enabled local group
    - [4734](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4734): A security-enabled local group was deleted
    - [4735](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4735): A security-enabled local group was changed
    - [4737](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4737): A security-enabled global group was changed
    - [4738](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4738): A user account was changed
    - [4739](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4739): Domain policy was changed
    - [4740](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4740): A user account was locked out
    - [4756](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4756): A member was added to a security-enabled universal group
    - [4764](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4764): A group type was changed
- Directory Service.evtx
    - [5136](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-5136): A directory service object was modified
    - [5137](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration): A directory service object was created
    - [5139](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration): A directory service object was moved
    - [5141](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration): A directory service object was deleted


### Network
Look at communications permitted by the Windows Filtering Platform.  
Lots of noise and rarely useful, but you can look if you want.

- Security.evtx
    - [5156](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-5156): Connection permitted (WFP)


### Microsoft Defender (formerly Windows Defender)
Look at malware detection, removal/quarantine, and disabling of protection features.  
Check the detection name, target path, action taken, and when protection was stopped.

There are a fair number of cases where it detected something but didn't remove/quarantine it, so it ended up running.

Note that the evtx name can differ by environment.

- Microsoft-Windows-Windows Defender%4Operational.evtx
    - [1116](https://learn.microsoft.com/en-us/defender-endpoint/troubleshoot-microsoft-defender-antivirus): Detected malware or potentially unwanted software
    - [1117](https://learn.microsoft.com/en-us/defender-endpoint/troubleshoot-microsoft-defender-antivirus): Took action on a detected threat
    - [5001](https://learn.microsoft.com/en-us/defender-endpoint/troubleshoot-microsoft-defender-antivirus): Real-time protection was disabled


### Log Tampering / Trace Deletion
Log clearing, Event Log service stop, audit event discard, etc.  
This isn't limited to here, but when investigating logs, it's better to first list which logs survive from when to when. So you don't end up with "No suspicious logons found!" → "Since when has the Security log been preserved?" → "...".

- Security.evtx
    - [1100](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-1100): The Event Log service was shut down
    - [1101](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=1101): Audit events were dropped by the transport
    - [1102](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-1102): The Security log was cleared
- System.evtx
    - [104](https://docs.logrhythm.com/devices/docs/evid-104-log-cleared): Other log cleared (System)


### Log Rotation
You might not use this much in investigation, but knowing it makes it easier to judge why there's a gap in the logs when you see one.

For event logs, you can specify per Channel "maximum size" and "what to do when full". The latter has the following three options.

1. Overwrite events as needed (overwrite old events as needed)
2. Archive the log when full, do not overwrite events (archive the full file and record to a new log file)
3. Do not overwrite events / Clear log manually (don't record new events when full)

1 reuses old log space in a FIFO manner to write new events. So old RecordID/RecordNumber numbers appear to be missing. Don't jump to "attacker deletion" just because there's a gap — check the rotation settings and think about why it's gone.

2 archives (auto-backs-up) the full log while creating a new file and keeps recording. As an investigator this is the happiest option, but naturally it eats up capacity fast. If you want to design log management properly, rather than hoarding it on local disk, you should design it to forward to an event log collection server or SIEM for storage. Also in this case, `1105` is recorded in the Security log.

3 stops recording new events once full. This setting is genuinely a pain, please stop using it. What possible joy does it bring. Also in this case, `1104` is recorded in the Security log.

- Security.evtx
    - [1104](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-1104): The security log is full
    - [1105](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-1105): The event log was automatically backed up


### Time / Power / Reboot / Shutdown
Log time settings and such — unglamorous but important.  
Also I check whether the power state is consistent with other traces. If something is recorded at a time when the power should be off, one of them is wrong. (Or sometimes I'm the one who's wrong.)

- Security.evtx
    - [4616](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4616): The system time was changed
- System.evtx
    - [12](https://learn.microsoft.com/en-us/troubleshoot/windows-server/performance/troubleshoot-unexpected-reboots-system-event-logs): Kernel-General: OS started
    - [13](https://learn.microsoft.com/en-us/troubleshoot/windows-server/performance/troubleshoot-unexpected-reboots-system-event-logs): Kernel-General: OS initiated shutdown
    - [41](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/event-id-41-restart): Kernel-Power: System rebooted without a clean shutdown
    - [1074](https://learn.microsoft.com/en-us/troubleshoot/windows-server/performance/troubleshoot-unexpected-reboots-system-event-logs): User32: Shutdown/reboot initiated by a process or user
    - [6005](https://learn.microsoft.com/en-us/troubleshoot/windows-server/performance/troubleshoot-unexpected-reboots-system-event-logs): EventLog: Event Log service started
    - [6006](https://learn.microsoft.com/en-us/troubleshoot/windows-server/performance/troubleshoot-unexpected-reboots-system-event-logs): EventLog: Event Log service stopped
    - [6008](https://learn.microsoft.com/en-us/troubleshoot/windows-server/performance/troubleshoot-unexpected-reboots-system-event-logs): EventLog: The previous shutdown was unexpected
    - [6009](https://learn.microsoft.com/en-us/troubleshoot/windows-server/performance/troubleshoot-unexpected-reboots-system-event-logs): EventLog: OS version info at boot
    - [6013](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=6013): EventLog: System uptime


### Application Anomaly
Application crashes, etc.  
Sometimes there's info on user-installed applications, sometimes not.

- Application.evtx
    - [1000](https://www.myeventlog.com/search/show/43): Application error
    - [1001](https://www.myeventlog.com/search/show/446): Application hang
    - [1002](https://www.myeventlog.com/search/show/726): Application hang


### Sysmon
If [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) is installed, honestly you won't struggle much if you just look at this and logons.  

If you're about to set it up, or want to observe malware behavior in an analysis environment, install it.  
That said, it's not install-and-done; if you don't tweak the config yourself to some extent, it tends to get noisy. The classic config is [SwiftOnSecurity/sysmon-config](https://github.com/swiftonsecurity/sysmon-config).


[A standard feature from Win11](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/sysmon) (not that I'm saying it's enabled)

- Microsoft-Windows-Sysmon%4Operational.evtx
    - [1](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90001):	Process creation
    - [2](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90002):	A process changed a file creation time
    - [3](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90003):	Network connection detected
    - [4](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90004):	Sysmon service state changed
    - [5](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90005):	Process terminated
    - [6](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90006):	Driver loaded
    - [7](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90007):	Image loaded
    - [8](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90008):	CreateRemoteThread
    - [9](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90009):	RawAccessRead
    - [10](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90010):	ProcessAccess
    - [11](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90011):	FileCreate
    - [12](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90012):	RegistryEvent (Object create and delete)
    - [13](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90013):	RegistryEvent (Value Set)
    - [14](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90014):	RegistryEvent (Key and Value Rename)
    - [15](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90015):	FileCreateStreamHash
    - [16](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90016):	Sysmon config state changed
    - [17](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90017):	Pipe created
    - [18](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90018):	Pipe connected
    - [19](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90019):	WmiEventFilter activity detected
    - [20](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90020):	WmiEventConsumer activity detected
    - [21](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90021):	WmiEventConsumerToFilter activity detected
    - [22](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90022):	DNSEvent
    - [23](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90023):	FileDelete
    - [24](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90024):	ClipboardChange
    - [25](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90025):	Process Tampering
    - [26](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90026):	File Delete Logged
    - [27](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90027):	File Block Executable
    - [28](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90028):	File Block Shredding
    - [29](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90029):	File Executable Detected
    - [225](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90225):	Error


## Logs to Look at Minimum
Preservation itself is fine to just do for everything, but when it's time to actually look and you have no idea what's what, it's time for determined visual grepping.  
For now, when you want to narrow by Provider/Channel count and skim, focus on the following.

- ActiveDirectoryWebService.evtx
- Application.evtx
- DFS Replication.evtx
- DNS Server.evtx
- Directory Service.evtx
- Microsoft-Windows-AppLocker%4EXE and DLL.evtx
- Microsoft-Windows-AppLocker%4MSI and Script.evtx
- Microsoft-Windows-AppLocker%4Packaged app-Deployment.evtx
- Microsoft-Windows-AppLocker%4Packaged app-Execution.evtx
- Microsoft-Windows-Bits-Client%4Operational.evtx
- Microsoft-Windows-CodeIntegrity%4Operational.evtx
- Microsoft-Windows-DNSServer%4Audit.evtx
- Microsoft-Windows-GroupPolicy%4Operational.evtx
- Microsoft-Windows-Kernel-Boot%4Operational.evtx
- Microsoft-Windows-Microsoft Defender%4Operational.evtx
- Microsoft-Windows-NTLM%4Operational.evtx
- Microsoft-Windows-PowerShell%4Admin.evtx
- Microsoft-Windows-PowerShell%4Operational.evtx
- Microsoft-Windows-RemoteDesktopServicesRdpCoreTS%4Operational.evtx
- Microsoft-Windows-SENSE%4Operational.evtx
- Microsoft-Windows-SMBServer%4Operational.evtx
- Microsoft-Windows-SMBServer%4Security.evtx
- Microsoft-Windows-SmbClient%4Connectivity.evtx
- Microsoft-Windows-Sysmon%4Operational.evtx
- Microsoft-Windows-TaskScheduler%4Operational.evtx
- Microsoft-Windows-TerminalServices-LocalSessionManager%4Operational.evtx
- Microsoft-Windows-TerminalServices-RDPClient%4Operational.evtx
- Microsoft-Windows-TerminalServices-RemoteConnectionManager%4Operational.evtx
- Microsoft-Windows-Time-Service%4Operational.evtx
- Microsoft-Windows-WMI-Activity%4Operational.evtx
- Microsoft-Windows-WinRM%4Operational.evtx
- Microsoft-Windows-Windows Defender%4Operational.evtx
- Microsoft-Windows-Windows Firewall With Advanced Security%4Firewall.evtx
- Microsoft-Windows-Windows Firewall With Advanced Security%4FirewallDiagnostics.evtx
- Microsoft-Windows-WindowsUpdateClient%4Operational.evtx
- OpenSSH%4Admin.evtx
- OpenSSH%4Operational.evtx
- Security.evtx
- Setup.evtx
- System.evtx
- Windows PowerShell.evtx

Reference: [Tokushima Prefecture Tsurugi-cho Handa Hospital Computer Virus Incident Expert Committee Investigation Report — Technical Edition](https://www.handa-hospital.jp/topics/2022/0616/index.html)


## Choosing the Right Tool

### For Non-Analysts

<img width="1040" height="664" alt="image" src="https://github.com/user-attachments/assets/78ccffb2-ad2d-4c9b-938f-80677004d9e8" />

[Event Viewer](https://learn.microsoft.com/en-us/shows/inside/event-viewer)

If you're a field SE or device administrator and you're told to look at the logs first and check whether there might be an intrusion, start here.

Use the Windows built-in Event Viewer.  
You can search and filter in the GUI, so it's plenty usable for an initial check.

However, **do NOT hand a CSV export to CSIRT or an analyst. Seriously, stop.**  
From the investigator's side, they have to start by reshaping that garbage CSV into something usable, which is extra work. Also a lot of information is lost. If you're handing it to the investigator, preserve it in `.evtx` format, not `.csv`.

<img width="999" height="333" alt="image" src="https://github.com/user-attachments/assets/1575ba11-560f-4f4f-b90e-f3a578c68d19" />

Example of a baffling CSV file exported from Event Viewer

If you're collecting data with fast forensics in mind, you can use something like [CDIR-Collector](https://github.com/CyberDefenseInstitute/CDIR). Click the `exe` a few times and you can grab the major data in one go.  
However, if you hand it to an overseas security vendor, they might go "what's that?", so ask which tool to collect with. Maybe [CyLR](https://github.com/orlikoski/CyLR), [KAPE](https://www.kroll.com/en/services/cyber/incident-response-recovery/kroll-artifact-parser-and-extractor-kape), [Velociraptor](https://github.com/Velocidex/velociraptor) or so..

If you're manually doing one or two files, Event Viewer's "Save All Events As" is fine, or you can export with `wevtutil epl`.  

The target files are under `C:\Windows\System32\winevt\Logs\`, but copying the file can fail due to locks, so if possible it's safer to get them via the proper export procedure.

Example:

```powershell
> wevtutil epl Security C:\Temp\Security.evtx
> wevtutil epl System C:\Temp\System.evtx
> wevtutil epl Application C:\Temp\Application.evtx
```


### Analysts

If you're CSIRT or a forensic investigator, pick whichever method you like.  
However, it's good to agree within the team on what to use.

**Even if you're a contrarian nerd who loves using a different tool from everyone else, don't forget that you have to be able to explain that the tool can guarantee evidentiary integrity.**

You need to be able to explain how the tool you used parses what, what format it outputs, and how it maps to the original log.

Below is my personal recommendation order.


| Parser        | Overview                                              | Notes                                     |
| ---------- | ----------------------------------------------- | -------------------------------------- |
| [EvtxECmd](https://github.com/EricZimmerman/evtx) | A command-line tool that converts EVTX files to CSV or JSON.              | One of the tools by [Eric Zimmerman](https://www.sans.org/profiles/eric-zimmerman), a SANS instructor. Widely used for DFIR. |
| [evtx2es/evtx2json](https://github.com/sumeshi/evtx2es)    | A command-line tool that imports EVTX files into Elasticsearch / converts to JSON.    | I made it because the event log parsers at the time were ridiculously slow. Handy when you want to cross-search a large number of event logs.                |
|[plaso](https://github.com/log2timeline/plaso)|A tool that creates a super timeline from various artifacts.|Closer to a framework than a single conversion tool. Fairly complex.|
|[log2timeline](https://code.google.com/archive/p/log2timeline/downloads)|The predecessor to plaso. This one is relatively simple.|A tool from way back in ancient times, so unless you have a strong reason, you don't need to adopt it now.|
| PowerShell | Windows' built-in scripting language. Can be used to extract and shape event logs.           | Reusable once you make it. Watch out, because a bug can cause secondary damage.                     |
| [Log Parser](https://www.microsoft.com/en-us/download/details.aspx?id=24659) | A command-line tool provided by Microsoft. Analyzes logs with SQL-like queries. | An official tool, but development has ended. Barely any docs. Someone please write them.                     |


| Analysis / Shaping Tool                | Overview                                    | Notes                                   |
| ----------------------- | ------------------------------------- | ------------------------------------ |
| [Timeline Explorer](https://www.sans.org/tools/timeline-explorer)       | A tool for viewing CSV etc. in timeline format. Strong at filtering and grouping. | One of [Eric Zimmerman](https://www.sans.org/profiles/eric-zimmerman)'s tools. Basically the one to use for a quick look, but saving/loading project files is weak. |
| [Quilter-CSV](https://github.com/sumeshi/qsv-rs) | CSV filter / conversion tool.            | Originally used [xsv](https://github.com/burntsushi/xsv), but it didn't scratch where it itched, so I made this.                       |
| [LibreOffice Calc](https://ja.libreoffice.org/)        | Open-source spreadsheet software.| Usable even where there's no Excel. Huge CSVs are a bit tough, huh.        |
| Excel                   | Spreadsheet software from Microsoft. Everyone knows it.| Paid. Lots of users, but it can't load huge logs and auto-converts times to numbers. I will smack you. |
| Elasticsearch + Kibana  | Search engine / visualization tool. Efficiently analyzes super-large logs.   | Strong at log indexing and cross-search. The query language is a rookie-killer.                 |
| [Event Log Explorer](https://eventlogxp.com/ja/) | A GUI tool specialized for viewing and analyzing event logs. | Paid for commercial use. A trusted tool also used in SANS training, but I don't find it very easy to use. |
| [TimeSketch](https://github.com/google/timesketch)  | A Google-made log analysis support tool. You can flag suspicious ones in the GUI.| High affinity with plaso. I [tried it](https://github.com/google/timesketch), and it might be nice if you're doing simultaneous work with multiple people. |
| [Log Parser Studio](https://learn.microsoft.com/en-us/exchange/iis-logs-and-log-parser-studio-reports-exchange-2013-help)  | A GUI front-end for Log Parser.   | Probably no longer distributed. You can still find it if you look, though.           |
| [Log Parser Lizard](https://log-parser.com/)  | A GUI tool in the Log Parser family.      | More capable than Log Parser Studio. But I can't really figure out how to use it.      |
| Splunk                  | A commercial log management / analysis platform. Strong at searching and visualizing huge logs.    | I hate it.                 |

Also, slightly different in flavor from the above, an intro to **hunting tools?**. You can scan logs with pre-made detection rules and catch suspicious events.

They don't just extract simply but also process and normalize, so it's hard to use the results directly for reporting, but a quick scan to grasp the overall picture and then detailed analysis makes it easy to find suspicious points fast.

That said, false positives can't be prevented, so don't swallow the results whole and watch out for bias.

| Hunting Tool| Overview| Notes|
| - | - | - |
| [Zircolite](https://github.com/wagga40/Zircolite)  | A tool that scans event logs with [Sigma](https://github.com/SigmaHQ/sigma) rules and detects. | The pioneer of this kind of tool, I think. (Sorry if I'm wrong.) |
| [Chainsaw](https://github.com/WithSecureLabs/chainsaw)  | Same as above. Insanely fast. | Seems pretty used overseas. |
| [Hayabusa](https://github.com/Yamato-Security/hayabusa) | Same as above. Super high-function. | The famed Yamato Security product. Docs are abundant and easy to use. The commit frequency is insane. |

#### Parsing with EvtxEcmd

CSV is the most handable format. You can grep it too.
Convert with file spec `-f` or folder spec `-d`. With a folder spec, the `.evtx` files underneath are merged into a single `.csv`.  

```.ps1
> EvtxECmd.exe -f Security.evtx --csv . --csvf Security.csv
```

<img width="1051" height="628" alt="image" src="https://github.com/user-attachments/assets/7b9bb6c4-e731-4c82-bb08-ad737ebee891" />


#### Filtering with Timeline Explorer

<img width="1257" height="761" alt="image" src="https://github.com/user-attachments/assets/03952f18-8d18-4cde-bb75-cf68ca3b0239" />

You can filter with conditions like `Event Id = 4624`, click columns to sort, group, and do almost anything you can think of.  
You can't automate it, but for a "let's just look at a bunch of stuff~" moment, this alone is fine.


#### Filtering with Quilter-CSV

[Quilter-CSV](https://github.com/sumeshi/qsv-rs) is a tool I made myself and I'm pretty happy with it.  

You can chain multiple processes like this. For simple processing, this alone is enough.

```.ps1
> qsv load Security.csv - isin EventId 4624 - select TimeCreated,EventId,PayloadData1 - sort TimeCreated - head 10 - showtable
```

<img width="1051" height="628" alt="image" src="https://github.com/user-attachments/assets/b276b742-bc38-41ca-ba07-58809baeb043" />


Alternatively, you can process along a pre-defined yaml file.  
This is nicer when you want to apply it to multiple files.

```
> qsv quilt test.yaml --var input=Security.csv
```

```.yaml
title: test

stages:
  load_stage:
    type: process
    steps:
      load:
        path: ${input}

  filtering:
    type: process
    source: load_stage
    steps:
      isin:
        colname: EventId
        values:
          - "4624"
      select:
        colnames:
          - TimeCreated
          - EventId
          - PayloadData1
      sort:
        colnames:
          - TimeCreated
      head:
        number: 10

  output_stage:
    type: process
    source: filtering
    steps:
      showtable:
```

<img width="1051" height="628" alt="image" src="https://github.com/user-attachments/assets/2727736a-dc4c-4f9c-8ebd-775fd5681d84" />

Security log from [JPCERTCC/LogonTracer](https://github.com/jpcertcc/logontracer)


## Closing

I'm so done with event log investigations!

The end
