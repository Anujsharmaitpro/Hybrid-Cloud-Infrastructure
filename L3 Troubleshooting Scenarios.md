# PART 35 — L3 Troubleshooting Scenarios (Complete Study Notes)

> **Scope:** Every scenario from the JD, each following the 13-step Interview-Focused Troubleshooting Framework. Organized by priority domain with full study-note structure.

---

## 🔷 DOMAIN 1: WINDOWS SERVER (⭐⭐⭐⭐⭐)

---

### 1.1 — Server Won't Boot

**Concept:** A Windows Server fails to complete the boot process, halting at a specific stage (BIOS/UEFI, boot manager, OS loader, kernel, or logon).

**Architecture:** Boot flow: Firmware (BIOS/UEFI) → Boot Manager (`bootmgfw.efi`/`bootmgr`) → Windows Boot Loader (`winload.efi`/`winload.exe`) → Kernel (`ntoskrnl.exe`) → Session Manager (`smss.exe`) → Logon Manager (`winlogon.exe`).

**Components:** Boot config data (BCD), boot files, disk controller drivers, storage drivers, system registry hives, system files.

**Configuration:** BCD store on system partition, boot-critical drivers in `C:\Windows\System32\drivers`, registry `HKLM\SYSTEM\CurrentControlSet`.

**Commands:**
```powershell
bcdedit /enum all
bootrec /fixmbr
bootrec /fixboot
bootrec /scanos
bootrec /rebuildbcd
sfc /scannow /offbootdir=C:\ /offwindir=C:\Windows
chkdsk C: /f /r
diskpart → list volume → select volume → detail volume
```

**Real-World Example:** After a botched patch, server boots to "BOOTMGR is missing" — the BCD entry was corrupted by a failed disk resize.

**Common Failures:**
- Corrupt BCD
- Missing/incompatible boot driver after BIOS update
- Disk cable/controller failure
- Corrupt system registry hive
- BitLocker recovery key not available
- Incorrect boot order in BIOS/UEFI

**Troubleshooting Steps (13-Framework):**

| Step | Action |
|------|--------|
| 1. Understand symptom | Server powers on but does not reach Windows login screen. Note exact error/message/hang point. |
| 2. Determine scope | Single server? Multiple? Recent change? All servers or isolated? |
| 3. Check recent changes | Patches, BIOS/UEFI update, disk replacement, driver update, BitLocker suspension. |
| 4. Check monitoring | Check iLO/iDRAC/IPMI, UPS logs, BIOS POST codes. |
| 5. Validate connectivity | N/A at this stage — ensure console/serial access is available. |
| 6. Check OS | Boot from Windows PE/Install media → access Recovery Environment (WinRE). |
| 7. Check dependencies | Disk visible in BIOS? Storage controller driver loaded? RAID status degraded? |
| 8. Check logs | In WinRE: `C:\Windows\System32\LogFiles`, `C:\Windows\System32\config\SYSTEM` registry, Setup logs (`C:\$WINDOWS.~BT\Sources\Panther`). |
| 9. Identify root cause | `bcdedit` shows missing entries? `chkdsk` reports corruption? Event ID 7 from disk? |
| 10. Implement fix | Rebuild BCD (`bootrec /rebuildbcd`), restore system files (`sfc /scannow /offbootdir`), restore from backup, replace disk. |
| 11. Validate service | Server boots fully to desktop/service login. Run `sfc /scannow` and `dism /online /cleanup-image /restorehealth`. |
| 12. Monitor | Watch for repeat boot failures over next 24-72 hours; check Event Viewer System log. |
| 13. Document RCA | Record what caused the corruption, the fix applied, and the preventive action (e.g., firmware update procedure change). |

**Root Cause → Fix → Validation:** Firmware update overwrote storage driver → rollback firmware / inject driver via WinRE → validate `sfc` clean, server boots normally.

**Interview Questions:**
- "A server returns 'BOOTMGR is missing' — walk me through your troubleshooting."
- "How does UEFI boot differ from Legacy BIOS boot in terms of recovery?"
- "What's the difference between `bootrec /fixboot` and `bootrec /rebuildbcd`?"

---

### 1.2 — Server Extremely Slow

**Concept:** Server exhibits degraded performance — high response times, application timeouts, sluggish operation.

**Architecture:** Performance depends on CPU, memory, disk I/O, network, and system services. Bottleneck at any layer causes slowness.

**Components:** Processes, services, drivers, disks, network adapters, scheduled tasks, antivirus, Windows Update.

**Configuration:** Power plan (Balanced vs. High Performance), virtual memory settings, antivirus exclusions, Windows Update policy.

**Commands:**
```powershell
Get-Process | Sort-Object CPU -Descending | Select-Object -First 20
Get-Process | Sort-Object WorkingSet64 -Descending | Select-Object -First 20
perfmon /res
resmon
tasklist /v
gpresult /r
Get-WindowsUpdateLog
Get-WinEvent -LogName 'Microsoft-Windows-WindowsUpdateClient/Operational' -MaxEvents 50
```

**Real-World Example:** SQL Server on a DC became extremely slow — Windows Defender was performing a full scan due to a missed exclusion.

**Common Failures:**
- Antivirus scan running
- Windows Update stuck/downloading
- Memory leak in application
- Disk fragmentation (HDD)
- Power plan set to "Balanced" on production server
- CPU throttling (thermal/power)
- Network saturation
- Resource exhaustion from runaway process

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Define "slow" — slow logins? Slow applications? Slow network? Slow disk I/O? When did it start? |
| 2. Determine scope | One server or many? Specific applications affected? All users or some? |
| 3. Check recent changes | New software, patch, GPO change, configuration change, new services deployed. |
| 4. Check monitoring | Performance counters (CPU %, Privileged Time, Disk Queue Length, Memory Available MBytes). Check SCOM/Zabbix/Nagios baselines. |
| 5. Validate connectivity | Test network latency: `ping`, `Test-NetConnection`, `tracert`. Check for packet loss. |
| 6. Check OS | `perfmon /res` for real-time overview. Check `Task Manager` → Details tab for per-process CPU/Memory/Disk/Network. |
| 7. Check dependencies | Is a backend service (DB, AD) slow? Is SAN/NAS responding? Check disk latency. |
| 8. Check logs | Event Viewer: System/Application logs. `Get-WinEvent` for WU, AppHang, Service Control Manager events. Check `C:\Windows\Prefetch` for heavy processes. |
| 9. Identify root cause | Process using 90% CPU? Memory commit limit reached? Disk queue length > 2? |
| 10. Implement fix | Kill runaway process, exclude AV paths, change power plan, restart service, clear temp files, defrag HDD, apply hotfix. |
| 11. Validate service | Re-run performance baselines. Confirm response times within SLA. Run `perfmon` counter sets. |
| 12. Monitor | Watch performance counters for 24-48 hours. Confirm no recurrence. |
| 13. Document RCA | Root cause, resolution, and monitoring alert adjustments. |

**Interview Questions:**
- "Users say the server is slow. What's your systematic approach?"
- "How do you differentiate between a memory issue and a disk I/O issue?"
- "What performance counters do you use to establish a baseline?"

---

### 1.3 — Service Keeps Stopping

**Concept:** A Windows service terminates unexpectedly, fails to start, or enters a restart loop.

**Architecture:** Services run under Service Control Manager (SCM). Each service has a configuration (start type, account, dependencies, recovery actions).

**Components:** Service executable, service account credentials, dependent services, DLLs, event logs, recovery configuration.

**Configuration:** `sc qc <servicename>` for config, recovery actions (restart on failure), service account, startup type.

**Commands:**
```powershell
Get-Service | Where-Object {$_.Status -eq 'Stopped'}
sc qc "ServiceName"
sc query "ServiceName"
sc start "ServiceName"
sc stop "ServiceName"
Get-WinEvent -LogName System | Where-Object {$_.Id -eq 7031 -or $_.Id -eq 7034}
eventvwr.msc → System log → Filter Event ID 7031, 7034, 7000, 7009
```

**Real-World Example:** The print spooler kept stopping on a print server — corrupted driver cache in `C:\Windows\System32\spool\drivers`.

**Common Failures:**
- Service account password expired/locked
- Dependency service stopped
- Corrupt service binary/DLL
- Insufficient permissions on service directories
- Port conflict (another service binding to same port)
- Crash loop due to unhandled exception
- Resource exhaustion

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Service stops, fails to start, or restarts in loop. Which service? How often? Error messages? |
| 2. Determine scope | Single service? Multiple services? Affects users/applications? |
| 3. Check recent changes | Service update, account password change, patch, config change. |
| 4. Check monitoring | SCOM/agent health, uptime monitoring, auto-restart events. |
| 5. Validate connectivity | If network service, test port connectivity (`Test-NetConnection -Port`). |
| 6. Check OS | `services.msc`, `sc qc <name>`, check startup type and recovery options. |
| 7. Check dependencies | `sc qc <name>` → DEPENDENCIES field. Are dependent services running? `sc enumdepend <name>`. |
| 8. Check logs | Event IDs: 7031 (service stopped unexpectedly), 7034 (service terminated unexpectedly), 7000 (service failed to start), 7009 (timeout). Application log for application-specific errors. |
| 9. Identify root cause | Expired password? Corrupt binary? Missing dependency? |
| 10. Implement fix | Reset service account password, restart dependencies, replace corrupt binary, adjust recovery actions, run `sfc /scannow`. |
| 11. Validate service | Service stays Running for extended period. Test functionality (e.g., connect to app using the service). |
| 12. Monitor | Watch event log and service status for 48 hours. |
| 13. Document RCA | Root cause, fix, preventive action (e.g., add password never-expires + service account monitoring). |

**Interview Questions:**
- "Event ID 7031 vs 7034 — what's the difference?"
- "A service keeps stopping. What do you check first?"
- "How do service recovery options work, and when would you use each?"

---

### 1.4 — High CPU

**Concept:** One or more processes or the system overall is consuming excessive CPU resources.

**Architecture:** CPU scheduling, process/thread management, hypervisor CPU overcommit (if virtualized), interrupt handling.

**Components:** Processes, threads, drivers, scheduled tasks, WMI providers, antivirus, Windows Update.

**Configuration:** Processor affinity, priority settings, power plan, CPU limits (in VM settings).

**Commands:**
```powershell
Get-Process | Sort-Object CPU -Descending | Select-Object -First 20 Name, Id, CPU, WorkingSet64
Get-Counter '\Processor(_Total)\% Processor Time', '\Process(*)\% Processor Time' -SampleInterval 2 -MaxSamples 5
Get-WinEvent -LogName System | Where-Object {$_.Id -eq 41} -MaxEvents 20  # kernel panic
wmic process get name,processid,workingsetsize /format:list | sort /R
tasklist /FI "STATUS eq RUNNING" /FO TABLE /NH | sort
```

**Real-World Example:** A server's CPU pegged at 100% — a WMI provider from a third-party application was in an infinite query loop.

**Common Failures:**
- Runaway process/thread
- Infinite loop in application
- Antivirus scan
- Windows Update reapplying patches
- Too many VMs on one host (CPU ready)
- Driver bug causing interrupt storms
- Scheduled task running at wrong time
- Cryptominer/malware

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | 100% CPU overall or per-core? One process or system-wide? Persistent or intermittent? |
| 2. Determine scope | One server or all VMs on host? Affects which users/applications? |
| 3. Check recent changes | New application, patch, scheduled task, configuration change. |
| 4. Check monitoring | CPU % per core, Privileged Time %, DPC/ISR latency. Check baselines. |
| 5. Validate connectivity | N/A primary — but verify network isn't causing connection storms. |
| 6. Check OS | `Task Manager` → Details. `Process Explorer` (Sysinternals) → look at thread-level CPU by process. `perfmon`. |
| 7. Check dependencies | Is a dependent service causing cascade? Is the CPU being stolen by another VM (VMware)? |
| 8. Check logs | Event ID 41 (unexpected shutdown from CPU overload/thermal). Application logs. Performance logs. |
| 9. Identify root cause | Process X consuming 85% CPU. Is it expected? Is it a known bug? Malware? |
| 10. Implement fix | Kill/limit process, update application, add CPU, fix scheduled task, exclude from AV scan, apply hotfix. |
| 11. Validate service | CPU returns to normal baseline. Application performs correctly. |
| 12. Monitor | Watch CPU counters for 24-48 hours. Set alert thresholds. |
| 13. Document RCA | Record root cause and long-term fix. |

**Interview Questions:**
- "You see 100% CPU. Walk me through your diagnosis."
- "What's the difference between User Time and Privileged Time in CPU analysis?"
- "How would you troubleshoot a CPU issue on a VM vs. a physical server?"

---

### 1.5 — High Memory

**Concept:** System is consuming excessive RAM, leading to swapping (pagefile thrashing), OOM kills, or application failures.

**Architecture:** Windows memory management: physical RAM → pagefile (`pagefile.sys`) → standby list → modified page list → disk I/O.

**Components:** Processes, working sets, shared memory, non-paged pool, system cache, pagefile configuration.

**Configuration:** Pagefile size and location, memory limits per process, working set trim thresholds.

**Commands:**
```powershell
Get-Process | Sort-Object WorkingSet64 -Descending | Select-Object -First 20 Name, Id, @{N='RAM(GB)';E={[math]::Round($_.WorkingSet64/1GB,2)}}
Get-Counter '\Memory\Available MBytes', '\Memory\% Committed Bytes In Use', '\Paging File(_Total)\% Usage' -SampleInterval 2 -MaxSamples 5
Get-Counter '\Process(*)\Working Set - Private' -SampleInterval 2
systempageproperties  # pagefile settings
Get-WinEvent -LogName System | Where-Object {$_.Id -eq 2004} -MaxEvents 10  # pagefile full warnings
```

**Real-World Example:** A server with 128GB RAM and a 32GB pagefile hit 100% — a Java app had a memory leak and was never configured with proper heap limits.

**Common Failures:**
- Memory leak in application
- Insufficient physical RAM for workload
- Pagefile too small or on slow disk
- Non-paged pool leak (driver issue)
- Too many running services
- Large system cache (file server)

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | High memory usage %? Application crashes with OOM? System paging excessively? Slow due to disk swapping? |
| 2. Determine scope | Single server? All servers with same config? Specific applications? |
| 3. Check recent changes | Application upgrade, new service, memory-intensive workload added. |
| 4. Check monitoring | Available MBytes (below 10% of total = critical), Committed Bytes In Use vs. Limit, Page File % Usage. |
| 5. Validate connectivity | N/A primary. |
| 6. Check OS | `Task Manager` → Performance tab. `Resource Monitor` → Memory tab. `perfmon`. Use RAMMap (Sysinternals) for detailed analysis. |
| 7. Check dependencies | Does an application depend on excessive caching? Is a database server starving other services? |
| 8. Check logs | Event ID 2004 (pagefile full), Event ID 2019 (insufficient memory from pool). Application crash logs. |
| 9. Identify root cause | Process X has growing private working set (memory leak). Pagefile is undersized. |
| 10. Implement fix | Restart leaking process, increase RAM, enlarge pagefile, fix application memory leak, tune application settings. |
| 11. Validate service | Memory stabilizes. Application runs without OOM errors. |
| 12. Monitor | Memory counters for 48 hours. Set alerts at 80% and 90% committed. |
| 13. Document RCA | Memory leak identified, vendor notified, workaround applied. |

**Interview Questions:**
- "What's the difference between Working Set and Working Set - Private?"
- "At what point does memory pressure cause performance issues?"
- "How do you identify a memory leak?"

---

### 1.6 — Disk 100%

**Concept:** Disk activity is saturated — high latency, long queue lengths, and I/O wait times degrade all disk-dependent operations.

**Architecture:** Disk I/O path: OS → filesystem driver → storage stack (class driver, miniport driver) → storage controller → physical disk (or SAN/NAS).

**Components:** Disk partitions, volumes, filesystem, pagefile, temp directories, antivirus, search indexing, storage controllers, SAN/NAS.

**Configuration:** Disk caching, RAID level, disk queue depth, SMB signing, storage QoS.

**Commands:**
```powershell
Get-Counter '\PhysicalDisk(_Total)\% Disk Time', '\PhysicalDisk(_Total)\Current Disk Queue Length', '\PhysicalDisk(_Total)\Avg. Disk sec/Read', '\PhysicalDisk(_Total)\Avg. Disk sec/Write' -SampleInterval 2
resmon → Disk tab
Get-ChildItem C:\ -Recurse -ErrorAction SilentlyContinue | Sort-Object Length -Descending | Select-Object -First 50 FullName, @{N='SizeGB';E={[math]::Round($_.Length/1GB,2)}}
chkdsk C: /f
Get-WinEvent -LogName System | Where-Object {$_.Id -eq 51} -MaxEvents 20  # disk error
```

**Real-World Example:** Disk 100% on a file server — Windows Search Indexer was indexing millions of files on an HDD RAID 1 array.

**Common Failures:**
- HDD degradation/failing disk in RAID
- RAID controller battery low
- Antivirus scanning heavy volumes
- Windows Search Indexing
- Temp/pagefile on system drive
- Fragmented HDD
- SAN/NAS latency (storage backend issue)
- Excessive small I/O operations

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | High disk latency? High queue length? Application timeouts on I/O? Blue screen (disk timeout)? |
| 2. Determine scope | One volume? All volumes? One server or all VMs on host? Physical or virtual disk? |
| 3. Check recent changes | New application, data migration, AV policy change, index rebuild. |
| 4. Check monitoring | `% Disk Time` (>90% = saturated), `Current Disk Queue Length` (>2 = bottleneck), `Avg. Disk sec/Read or Write` (>0.025s = slow). |
| 5. Validate connectivity | For SAN/NAS: check network to storage, multipath status, HBA links. |
| 6. Check OS | `Resource Monitor` → Disk tab. `Resource Monitor` → Disk → Storage tab. Check volume details. |
| 7. Check dependencies | Is storage controller healthy? RAID degraded? Battery低? SAN snapshot running? |
| 8. Check logs | Event ID 51 (disk error), 55 (file system corruption), 57 (I/O timeout). `chkdsk` results. |
| 9. Identify root cause | Failing disk in RAID 5? Search Indexer hammering disk? Block storage latency from storage tier? |
| 10. Implement fix | Replace failing disk, pause indexing, move pagefile/temp to faster disk, exclude from AV, defrag HDD, rebuild RAID, upgrade to SSD/NVMe. |
| 11. Validate service | Disk latency below 20ms, queue length < 1, applications responsive. |
| 12. Monitor | Disk counters for 48 hours. Check SMART data via `Get-PhysicalDisk`. |
| 13. Document RCA | Record root cause, replacement, and preventive monitoring. |

**Interview Questions:**
- "Disk 100% — what are the top 3 causes you check?"
- "What counter values indicate disk I/O is the bottleneck vs. CPU?"
- "How do you check if a disk is failing?"

---

### 1.7 — Disk Full

**Concept:** A volume or disk has exhausted its allocated storage space, causing application failures, log corruption, or system instability.

**Architecture:** File systems (NTFS/ReFS) track allocation. When free space = 0, writes fail, causing cascading errors.

**Components:** Volumes, file system, logs (event, application, IIS, SQL), temp files, pagefile, data files, shadow copies, Windows Update files, DFSR database.

**Configuration:** Disk quotas, volume size, retention policies, log rotation, DFSR settings.

**Commands:**
```powershell
Get-PSDrive C, D, E | Select-Object Name, Used, Free, @{N='Free%';E={[math]::Round($_.Free/$_.Used*100,2)}}
fsutil volume diskfree C:
wmic logicaldisk get name,size,freespace,caption
Get-ChildItem C:\ -Recurse -ErrorAction SilentlyContinue | Sort-Object Length -Descending | Select-Object -First 30 FullName, @{N='SizeGB';E={[math]::Round($_.Length/1GB,2)}}
CleanMgr /d C
Get-WinEvent -LogName System | Where-Object {$_.Id -eq 2013} -MaxEvents 10  # volume full event
```

**Real-World Example:** C: drive full on a server — `C:\Windows\Logs\CBS` had accumulated 50GB of logs from failed updates over two years.

**Common Failures:**
- Log file explosion (Event logs, application logs, CBS logs, WU logs)
- Windows Update accumulation (`C:\$WINDOWS.~BT`, `C:\Windows\SoftwareDistribution`)
- Temp files
- DFSR database growth
- Shadow copies not managed
- Database file growth (SQL, Exchange)
- Large file uploads/copies

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Error messages about disk space? Applications failing to write? Event log errors about volume full? |
| 2. Determine scope | Which volume? Single server? Multiple servers? Affects what services? |
| 3. Check recent changes | Increased logging, backup retention change, data import, failed updates. |
| 4. Check monitoring | Disk free % monitoring alerts (typically 15% threshold). |
| 5. Validate connectivity | N/A primary. |
| 6. Check OS | `Get-PSDrive`. `WinDirStat` / `TreeSize` / `fsutil` to identify largest space consumers. `CleanMgr`. |
| 7. Check dependencies | If C: is full, can the system write temp/pagefile? Are services storing data on C:? |
| 8. Check logs | Event ID 2013 (volume full). Application errors about write failures. Check log directories for size. |
| 9. Identify root cause | `C:\Windows\Logs\CBS` = 50GB. `C:\Windows\Temp` = 20GB. IIS logs = 100GB. |
| 10. Implement fix | Clear temp files, rotate logs, clear CBS logs (after uninstalling bad updates), extend volume, add disk, configure log retention, move pagefile. |
| 11. Validate service | Free space restored above threshold. Applications writing successfully. `Get-EventLog` no new 2013 events. |
| 12. Monitor | Set persistent monitoring/alerting. Implement log rotation policies. |
| 13. Document RCA | Root cause, cleanup performed, and policy changes (e.g., weekly CBS log cleanup, log retention 30 days). |

**Interview Questions:**
- "C: drive is 100% full. What's your approach?"
- "What are the safe vs. unsafe things to delete when a disk is full?"
- "How do you prevent disk full issues proactively?"

---

### 1.8 — RDP Unavailable

**Concept:** Remote Desktop Protocol connections to a server fail or refuse.

**Architecture:** RDP (TCP 3389) → RDP Service (`TermService`) → Session Manager → User session. SSL/TLS certificate for encryption.

**Components:** TermService, RDP listener, firewall rules, network connectivity, RDP certificate, licensing, user session limits.

**Configuration:** RDP settings (System Properties → Remote), firewall rules, `rdp-tcp` listener in `regedit`, MaxIdleTime, MaxConnections.

**Commands:**
```powershell
Test-NetConnection -ComputerName ServerName -Port 3389
Get-Service TermService
sc qc TermService
Get-NetFirewallRule -DisplayName "*Remote Desktop*" | Get-NetFirewallPortFilter
Get-WmiObject Win32_TerminalServiceSetting | Select-Object *
tsdiscon /admin
qwinsta /server:ServerName
reg query "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections
powershell -Command "Get-WmiObject -Class Win32_TSIPEConnection"  # RDP connections
```

**Real-World Example:** RDP stopped working after Windows update — the RDP certificate thumbprint in registry didn't match the new certificate.

**Common Failures:**
- TermService stopped or hung
- Firewall blocking port 3389
- Network/firewall (corporate firewall, cloud NSG)
- RDP disabled in registry
- RDP certificate expired
- Max connection limit reached (licensed)
- Session hangs (orphaned sessions)
- Network Level Authentication (NLA) issues with outdated client
- IP conflict

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | RDP connection refused? Timeout? Blue screen on connect? Authentication failure? |
| 2. Determine scope | One server? Multiple? All users or specific? Local or remote connection? |
| 3. Check recent changes | Windows update, firewall rule change, certificate renewal, GPO change. |
| 4. Check monitoring | RDP service status, network monitoring. |
| 5. Validate connectivity | `Test-NetConnection -Port 3389`. `traceroute`. `ping`. |
| 6. Check OS | `Get-Service TermService`. `System Properties → Remote → Allow connections`. `regedit`: `fDenyTSConnections`. |
| 7. Check dependencies | Firewall allows 3389? NLA compatible? Certificate valid? |
| 8. Check logs | Event Viewer: `Applications and Services Logs → Microsoft → Windows → TerminalServices-LocalSessionManager → Operational`. Event ID 1149 (auth failure), 1148, etc. |
| 9. Identify root cause | Service stopped? Firewall blocking? Certificate expired? Session limit? |
| 10. Implement fix | Restart TermService, open firewall rule, replace certificate, increase session limit, delete orphaned sessions (`tsdiscon`). |
| 11. Validate service | Successfully connect via RDP. Run commands. |
| 12. Monitor | RDP service uptime, connection count. |
| 13. Document RCA | Root cause, fix, and preventive action. |

**Interview Questions:**
- "RDP not working — what do you check first?"
- "How do you troubleshoot RDP when Test-NetConnection shows port open but you still can't connect?"
- "What registry keys control RDP behavior?"

---

### 1.9 — Windows Update Failure

**Concept:** Windows Update fails to download, install, or complete updates.

**Architecture:** Windows Update client → Windows Update Agent (WUA) → Microsoft Update servers (or WSUS) → download → install → reboot → finalize.

**Components:** Windows Update service (wuauserv), BITS, Windows Update database (`C:\Windows\SoftwareDistribution`), CBS (Component-Based Servicing), DISM, WSUS configuration.

**Configuration:** Update policy (auto-updates, active hours), WSUS target, GPO update settings, proxy settings.

**Commands:**
```powershell
Get-Service wuauserv, bits, cryptsvc
net stop wuauserv; net stop bits; net stop cryptsvc
ren C:\Windows\SoftwareDistribution SoftwareDistribution.bak
ren C:\Windows\System32\catroot2 catroot2.bak
net start wuauserv; net start bits; net start cryptsvc
wuauclt /resetauthorization /detectnow
dism /online /cleanup-image /restorehealth
sfc /scannow
Get-WindowsUpdateLog
Get-WinEvent -LogName "Microsoft-Windows-WindowsUpdateClient/Operational" -MaxEvents 50
Get-WinEvent -LogName "Microsoft-Windows-WindowsUpdate/Operational" -MaxEvents 50
```

**Real-World Example:** Updates stuck at 0% for hours — BITS service was hung after a power failure; renaming `SoftwareDistribution` resolved it.

**Common Failures:**
- Corrupt `SoftwareDistribution` folder
- CBS corruption
- DISM/WIM corruption
- WSUS connectivity issues
- Proxy/firewall blocking Microsoft Update
- Insufficient disk space
- Stuck install requiring reboot
- Pending file rename operations
- Certificate issues for update signing

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Updates not checking? Download stuck? Installation fails? Error code? |
| 2. Determine scope | One server? All servers? Specific update(s)? |
| 3. Check recent changes | Previous failed update, pending reboot, GPO change, WSUS update approval. |
| 4. Check monitoring | WSUS console for update status, computer target group membership. |
| 5. Validate connectivity | `Test-NetConnection` to Microsoft Update endpoints (e.g., `windowsupdate.com` on 443/80). Proxy configured correctly? |
| 6. Check OS | `Windows Update` settings. `Get-WindowsUpdateLog`. `Event Viewer → Windows Update logs`. |
| 7. Check dependencies | WUA service running? BITS running? CryptSvc running? Disk space? |
| 8. Check logs | `C:\Windows\WindowsUpdate.log` (Win10+), CBS log (`C:\Windows\Logs\CBS\CBS.log`), DISM log (`C:\Windows\Logs\DISM\dism.log`). Event IDs: WU client events. |
| 9. Identify root cause | Corrupt `SoftwareDistribution`? Pending reboot? WSUS error? Specific KB fails? |
| 10. Implement fix | Reset WUA components (stop services → rename folders → restart), run DISM restorehealth + sfc, manually install KB, free disk space, clear pending reboot registry. |
| 11. Validate service | `wuauclt /detectnow` finds and installs updates successfully. No errors in logs. |
| 12. Monitor | Check update health over next 7-14 days. |
| 13. Document RCA | Root cause, steps taken, and prevention (e.g., regular DISM/sfc maintenance). |

**Interview Questions:**
- "Walk me through resetting Windows Update components."
- "What's the difference between `sfc /scannow` and `DISM /RestoreHealth`?"
- "How do you troubleshoot a Windows Update error code?"

---

### 1.10 — Blue Screen (BSOD)

**Concept:** The kernel encounters a fatal error and halts the system, displaying a blue screen with a stop code.

**Architecture:** Kernel mode crash → memory dump written (`C:\Windows\Minidump` or `MEMORY.DMP`) → system reboots.

**Components:** Kernel (`ntoskrnl.exe`) and its modules, drivers (especially third-party), HAL, registry.

**Configuration:** Crash dump configuration (`C:\Windows\Minidump`), pagefile for crash dump writing.

**Commands:**
```powershell
Get-WinEvent -LogName System | Where-Object {$_.Id -eq 41 -or $_.Id -eq 1074} -MaxEvents 20
# For crash analysis:
# Download WinDbg / Debugging Tools for Windows
# Open .dmp file in WinDbg → !analyze -v
# Alternatively use BlueScreenView (NirSoft) for quick analysis
# Check: C:\Windows\Minidump\*.dmp
# Recent dump: C:\Windows\MEMORY.DMP
# Check stop code from Event Viewer: System log → Event ID 41 → Kernel-Power or Bugcheck
```

**Real-World Example:** BSOD with `DRIVER_IRQL_NOT_LESS_OR_EQUAL` — a third-party VPN driver was incompatible with the latest Windows update.

**Common Failures:**
- Faulty/incompatible driver (most common)
- Faulty RAM
- Corrupt system files
- Overheating
- Incompatible BIOS/UEFI settings
- Storage controller driver issue
- BIOS/UEFI firmware bug

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Stop code? Error message? Parameters? Frequency? Blue screen on boot or during use? |
| 2. Determine scope | One server? Multiple? After specific change? Random? |
| 3. Check recent changes | Driver update, hardware change, BIOS update, Windows update, new software. |
| 4. Check monitoring | Server uptime before crash. Hardware health (iLO/iDRAC). Temperature logs. |
| 5. Validate connectivity | N/A primary. |
| 6. Check OS | `BSODStopCode` from Event Viewer (System log → filter for critical events). Analyze minidump in WinDbg. |
| 7. Check dependencies | Is the crash tied to a specific driver? Check `C:\Windows\Minidump` files. |
| 8. Check logs | Event ID 41 (kernel-power, bugcheck), Event ID 1074 (unexpected shutdown by user/process). Minidump analysis in WinDbg (`!analyze -v`). Look at the driver listed in the stack. |
| 9. Identify root cause | Driver `xxxx.sys` identified in crash dump. Faulty RAM address. Corrupt system file. |
| 10. Implement fix | Update/rollback driver, run `mdsched.exe` for RAM test, run `sfc /scannow` + `DISM`, update BIOS, disable overclocking, replace hardware. |
| 11. Validate service | Server runs stable for 7+ days without recurrence. |
| 12. Monitor | Watch uptime, crash monitoring, Event Viewer. |
| 13. Document RCA | Stop code, faulty component/driver, resolution, and preventive action. |

**Interview Questions:**
- "A server BSODs with `0x0000007B` — what does that mean and how do you fix it?"
- "How do you analyze a memory dump?"
- "What tools do you use for BSOD analysis?"

---

### 1.11 — Active Directory Troubleshooting (User Login, Lockouts, Trust, GPO, Replication)

Covered in dedicated AD section below due to scope.

---

## 🔷 DOMAIN 2: ACTIVE DIRECTORY (⭐⭐⭐⭐⭐)

---

### 2.1 — User Cannot Log In

**Concept:** An authenticated user is denied access to a domain-joined computer or service.

**Architecture:** Logon process: Computer (securty policy) → Netlogon service → DC → KDC (Kerberos) / NTLM → Authentication Service → Token generation → Session.

**Components:** User account, computer account, AD database, KDC service, Netlogon, DNS, site/subnet configuration, trust relationships.

**Configuration:** Account properties (password, expiration, logon hours, logon restrictions, smart card required), SPN configuration.

**Commands:**
```powershell
# On DC:
Get-ADUser -Identity username -Properties * | Select-Object Name, Enabled, LockedOut, PasswordExpired, PasswordLastSet, LastLogon, LastLogonTimestamp
# Check if locked:
Get-ADUser -Identity username -Properties lockedout
net user username /domain
# On client:
klist
klist purge
gpresult /r
gpresult /h report.html
nltest /dsgetdc:domain.com
nltest /sc_query:domain.com
# Check secure channel:
Test-ComputerSecureChannel -Verbose
# Reset machine account:
Reset-ComputerMachinePassword -Server DC01.domain.com -Credential (Get-Credential)
```

**Real-World Example:** User locked out after typing wrong password 5 times on a kiosk machine — had to unlock account in ADUC or set `LockoutTime = 0`.

**Common Failures:**
- Account disabled
- Password expired
- Account locked out
- Logon hours restricted
- Computer account stale/deleted
- Secure channel broken (machine password out of sync)
- DNS pointing to wrong DC
- KDC service not running on DC
- Cached credentials mismatch after password change
- GPO restricting logon (logon workstation restriction)

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Error message? "Account locked"? "Password incorrect"? "Access denied"? At logon screen or after credentials? |
| 2. Determine scope | One user? Multiple users? One computer? All computers? Domain-wide? |
| 3. Check recent changes | Password change, account manipulation, GPO change, computer move, DC issue. |
| 4. Check monitoring | AD health, DC replication, event logs on DC and client. |
| 5. Validate connectivity | Client can reach DC? `nltest /dsgetdc:domain.com`. `ping` DC. DNS resolution of DC. `Test-NetConnection DC -Port 88/389/445/636`. |
| 6. Check OS | On client: `klist` (ticket cache). `gpresult`. On DC: ADUC → user properties. `Get-ADUser` properties. |
| 7. Check dependencies | Is the user account enabled? Not expired? Not locked? Correct password? DC available? KDC running? DNS correct? |
| 8. Check logs | DC Security log: Event ID 4625 (failed logon) with sub-status codes (e.g., 0xC0000133 = clock skew, 0xC000006A = bad password, 0xC0000234 = account locked). Client Security log. Netlogon log (`nltest /dbflag:0x2080ffff`). |
| 9. Identify root cause | Locked out (wrong password on another device)? Expired password? Disabled account? Stale secure channel? |
| 10. Implement fix | Unlock account (`Unlock-ADAccount`), reset password, re-enter password on client (`Ctrl+Alt+Del` → change password if expired), verify DNS to correct DC, rebuild secure channel (`Test-ComputerSecureChannel -Repair`). |
| 11. Validate service | User logs in successfully. Runs `klist` to see TGT. `gpresult` shows correct GPOs. |
| 12. Monitor | Event logs for repeat 4625 failures. Account lockout monitoring. |
| 13. Document RCA | Root cause, resolution, and prevention (e.g., account lockout threshold review, password expiry notification). |

**Interview Questions:**
- "A user says 'password is correct but login fails.' What's your approach?"
- "How do you find out why a logon failed using Event Viewer?"
- "What Event IDs relate to logon failures and what do their sub-status codes mean?"

---

### 2.2 — Account Repeatedly Locks

**Concept:** An account exceeds the lockout threshold and is repeatedly locked, often due to authentication attempts from various sources.

**Architecture:** Lockout mechanism: Each DC maintains lockout counter. When threshold met → account locked for lockout duration.

**Components:** User account, lockout policy (GPO: Account Policies → Account Lockout), DCs, applications/services using the account, cached credentials.

**Configuration:** Lockout threshold, lockout duration, reset counter after. Registered in GPO.

**Commands:**
```powershell
# Find what DC locked the account:
# Event ID 4740 on DCs:
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4740} -MaxEvents 50
# Find source of failed logons:
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} | Where-Object {$_.Properties[1].Value -eq 'username'}
# Microsoft Account Lockout Tools (download from Microsoft):
# ALTools (AccountLockoutTools.exe) — use `LockoutStatus.exe` to check all DCs for lockout status
# EventCombMT tool for scanning
# Check pwdLastSet and lastLogon:
Get-ADUser -Identity username -Properties LockedOut, LockoutTime, LastLogon, LastLogonTimestamp, pwdLastSet
# Unlock:
Unlock-ADAccount -Identity username
# Find which service/computer is causing lockouts - check 4625 events for source workstation name:
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} | Select-Object @{N='Source';E={$_.Properties[11].Value}}, @{N='User';E={$_.Properties[0].Value}} | Group-Object Source | Sort-Object Count -Descending
```

**Real-World Example:** Service account locked daily — a scheduled task on an old server was using stale credentials. Found via 4625 Event source analysis.

**Common Failures:**
- Service accounts with cached/wrong passwords on servers
- Scheduled tasks using expired credentials
- Web applications with hardcoded credentials
- Devices with cached credentials (phones, printers)
- VPN connections with stored credentials
- Microsoft Outlook profiles with cached credentials
- Mapped drives with saved credentials

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Account locked repeatedly. How often? Which account? |
| 2. Determine scope | Single account? Multiple accounts? Domain-wide? |
| 3. Check recent changes | Password change not propagated? New service? New device? |
| 4. Check monitoring | AD account lockout events, event log aggregation from all DCs. |
| 5. Validate connectivity | All DCs responding? Replication healthy? |
| 6. Check OS | On DCs: Security log Event ID 4740 (account locked), 4625 (failed logon). `LockoutStatus.exe` from Microsoft tools (shows lockout status on all DCs). |
| 7. Check dependencies | Are services/applications using this account? Are devices connecting with cached credentials? |
| 8. Check logs | Correlate 4740 (lockout) events across all DCs with 4625 (failed logon) events to identify source IP/computer name. |
| 9. Identify root cause | Source computer X is sending wrong passwords. What service runs on X? |
| 10. Implement fix | Update password on source service/device, reset lockout counter, unlock account, reconfigure service account password, implement managed service account (gMSA) to avoid this entirely. |
| 11. Validate service | Account remains unlocked for 7+ days. Source service authenticates successfully. |
| 12. Monitor | Set alert on 4740 events (Lockout Analyzer in SCOM or custom script). |
| 13. Document RCA | Root cause identified (specific service/device), resolution, and prevention (move to gMSA). |

**Interview Questions:**
- "An account is locked out. How do you find the source?"
- "How do gMSAs solve service account lockout problems?"
- "What Event IDs and sub-status codes indicate a lockout?"

---

### 2.3 — Computer Trust Relationship Broken

**Concept:** The computer account password is out of sync between the domain-joined computer and AD, causing authentication failures.

**Architecture:** Machine authentication: Computer generates hash of machine password → sends to DC → DC verifies against stored machine password hash → if match, secure channel established.

**Components:** Computer account in AD, Netlogon service, secure channel, machine password stored in both AD and local registry.

**Configuration:** Machine password change frequency (default: 30 days), secure channel settings.

**Commands:**
```powershell
# Test secure channel:
Test-ComputerSecureChannel -Verbose
Test-ComputerSecureChannel -Repair -Verbose
# Reset machine password:
Reset-ComputerMachinePassword -Server DC01.domain.com -Credential (Get-Credential)
# On DC, check computer account:
Get-ADComputer -Identity COMPUTERNAME -Properties * | Select-Object Name, Enabled, LastLogon, LastLogonTimestamp, PasswordLastSet
# Nltest:
nltest /sc_query:domain.com
nltest /sc_reset:domain.com
# Event logs:
Get-WinEvent -FilterHashtable @{LogName='System'; Id=5719 -or $_.Id -eq 5805}
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4626}
# Netdom:
netdom resetpwd /server:DC01.domain.com /userd:domain\adminuser /passwordd:*
```

**Real-World Example:** After restoring a DC from snapshot, computer trust broke for ~200 machines — machine passwords had been set to the old value from before the snapshot.

**Common Failures:**
- DC restored from snapshot
- Password changed manually on local machine
- Machine password not synced after VM clone
- Computer account disabled or deleted in AD
- Long downtime (> default 30-day password change period)
- DC time skew beyond 5-minute Kerberos tolerance

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | "The trust relationship between this workstation and the primary domain failed." When does it happen? At login or randomly? |
| 2. Determine scope | One computer? Many? Specific OU? After DC restore? |
| 3. Check recent changes | DC restore, computer reimage, machine password manually changed, VM clone. |
| 4. Check monitoring | Secure channel health (can be monitored via scripts/SCOM). |
| 5. Validate connectivity | Computer can reach DC on ports 88, 389, 445, 636, 53. |
| 6. Check OS | `Test-ComputerSecureChannel`. Check computer account in ADUC. `nltest /sc_query`. |
| 7. Check dependencies | DC operational? Replication healthy? Time synced? |
| 8. Check logs | Event ID 5719 (cannot find domain controller), 5805 (machine password change failed), Security Event ID 4626 (secure channel failure). |
| 9. Identify root cause | Machine password mismatch. DC time wrong? Computer account disabled? |
| 10. Implement fix | `Test-ComputerSecureChannel -Repair`, or `Reset-ComputerMachinePassword` from DC, or `netdom resetpwd`. Reboot after fix. |
| 11. Validate service | `Test-ComputerSecureChannel` returns True. User logs in with domain credentials. `klist` shows TGT. |
| 12. Monitor | Watch for repeat 5719/5805 events. |
| 13. Document RCA | Root cause, fix, and process update (e.g., never restore DC without knowing implications). |

**Interview Questions:**
- "How do you repair a broken trust relationship?"
- "What causes trust relationship failures in VMs specifically?"
- "When is `Test-ComputerSecureChannel -Repair` not enough?"

---

### 2.4 — GPO Not Applying

**Concept:** Group Policy Objects fail to apply to users or computers, resulting in missing configurations.

**Architecture:** GPO processing: Computer starts → Netlogon → Group Policy Client (`gpsvc`) → finds GPOs linked to OU → downloads from SYSVOL → applies settings → registers results.

**Components:** GPOs, WMI filters, security filtering, OU structure, links, SYSVOL replication, GPMC, `gpsvc`, registry (`HKLM\Software\Microsoft\Windows\CurrentVersion\Policies`).

**Configuration:** GPO links (enabled/disabled), WMI filters, security group filtering, Enforce, Block Inheritance, loopback processing.

**Commands:**
```powershell
# Force GP update:
gpupdate /force
gpresult /r
gpresult /h C:\gpreport.html /f
# Check which GPOs apply:
Get-GPResultantSetOfPolicy -ReportType Html -Path C:\gpreport.html -Computer ComputerName
# Check GPResultantSetOfPolicy via PowerShell:
Get-GPResultantSetOfPolicy -Computer Server01 -Report rSoP_Current -Path C:\rsop.xml
# Check SYSVOL:
Test-Path \\domain.com\SYSVOL\domain.com\Policies\{GPO-GUID}
# Check GPO links:
Get-GPLink -Target "OU=Servers,DC=domain,DC=com"
# Check WMI filters:
Get-GPWmiFilter -Name "FilterName"
# Check specific GPO:
Get-GPO -Guid "{GUID}" | Select DisplayName, GpoStatus, Enabled
Get-GPOStatistics -Guid "{GUID}"
# Debug GPO:
# Event Viewer: Applications and Services Logs → Microsoft → Windows → GroupPolicy → Operational
Get-WinEvent -LogName "Microsoft-Windows-GroupPolicy/Operational" | Where-Object {$_.Id -eq 4006 -or $_.Id -eq 4015 -or $_.Id -eq 4016 -or $_.Id -eq 4098}
```

**Real-World Example:** New GPO for software deployment not applying — WMI filter was targeting `Win10 21H2` but all machines were `22H2`.

**Common Failures:**
- GPO link disabled
- Security filtering (wrong group)
- WMI filter not matching
- GPO blocked by inheritance/block inheritance
- SYSVOL not replicating (DFS-R issue)
- GPO has errors (validation fails)
- Enforced GPO blocking lower GPOs
- Loopback processing configured unexpectedly
- Client-side extension (CSE) failure
- Computer not in correct OU

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Specific setting not applied? All GPOs or specific? User or computer? |
| 2. Determine scope | One computer/user? Specific OU? All users/computers? |
| 3. Check recent changes | GPO created/modified, OU restructure, WMI filter change, security group change, link disabled. |
| 4. Check monitoring | GPResult reports, SCOM GPO monitoring, GroupPolicy Operational log. |
| 5. Validate connectivity | Client can reach SYSVOL share (`\\domain.com\SYSVOL`). DNS resolution of DC. Port 445 open. |
| 6. Check OS | `gpresult /r` (shows which GPOs applied/not applied and why). `gpupdate /force`. Check registry: `HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\{GPO-GUID}`. |
| 7. Check dependencies | GPO in SYSVOL? DFSR replicating? Correct user/computer account? In correct OU? |
| 8. Check logs | GroupPolicy/Operational log: Event 4006 (GPO processing started), 4015 (GPO processing failed), 4098 (GPO failed to apply). GPResult HTML report has detailed status. |
| 9. Identify root cause | GPO link disabled? Security filtering excludes this machine? WMI filter false? DFS-R not replicating GPO? |
| 10. Implement fix | Enable link, correct security filtering, fix WMI filter, resolve SYSVOL replication, move object to correct OU, check GPO version. |
| 11. Validate service | `gpresult /r` shows GPO applied with correct settings. Registry keys present. Settings verified. |
| 12. Monitor | Run `gpresult` periodically. Check GPStatus reports. |
| 13. Document RCA | Root cause, resolution, and process improvement (e.g., GPO testing in staging OU before production). |

**Interview Questions:**
- "GPO not applying — what's your checklist?"
- "How do WMI filters affect GPO application?"
- "What's the difference between Block Inheritance and Enforced?"
- "How do you troubleshoot when `gpresult` shows 'No' for a GPO?"

---

### 2.5 — AD Replication Failure

**Concept:** Directory changes made on one DC are not propagating to other DCs in the domain/forest.

**Architecture:** Multi-master replication: Changes made at any DC replicate to all DCs via KCC (Knowledge Consistency Checker) automatic topology or manually configured connection objects. Uses RPC over LDAP (port 135 + dynamic).

**Components:** KCC, connection objects, replication partners, site links, replication schedule, USNs (Update Sequence Numbers), DSA objects, DFSR for SYSVOL.

**Configuration:** Site links, bridgehead servers, replication schedule, inter-site replication interval, options (transitive vs. non-transitive).

**Commands:**
```powershell
# Check replication:
repadmin /replsummary
repadmin /showrepl
repadmin /showutdvec  # check USN consistency
repadmin /showobjattr *  # show object attributes
repadmin /failcache  # show failed replication attempts
repadmin /syncall /AdeP  # force sync all
# Check specific DC:
repadmin /showrepl DC01.domain.com
# Check DCDIAG:
dcdiag /v /e /c  # comprehensive
dcdiag /s:DC01 /c  # check specific DC for replication errors
# Check SYSVOL DFSR:
dfsrdiag PollAD /Member:DC01.domain.com
# Check KCC:
# Event Viewer: System log → Directory Service events (Event ID 1988 for knowledge consistency)
Get-WinEvent -FilterHashtable @{LogName='System'; Id=1988} -MaxEvents 20
# Check site links:
Get-ADReplicationSiteLink -Filter * | Select-Object Name, SitesIncluded, ReplicationInterval, Schedule
# Check AD health:
dcdiag /check:replications /v
```

**Real-World Example:** After a site link misconfiguration, two DCs in different sites stopped replicating — `repadmin /replsummary` showed "Access denied" due to wrong credentials on connection object.

**Common Failures:**
- Firewall blocking RPC (135 + dynamic ports) between DCs
- DNS resolution of DCs failing
- DC offline/unreachable
- Site link misconfigured (wrong sites, schedule)
- Connection object broken
- USN rollback (bad restore)
- KCC topology corrupt
- Disk full on DC (AD database can't grow)
- Time skew > 5 minutes (Kerberos)

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Changes not appearing on other DCs? Specific DC or all? Inter-site or intra-site? |
| 2. Determine scope | Single domain? Forest-wide? Single site or multiple? All partitions or specific (Domain, Schema, Configuration)? |
| 3. Check recent changes | DC added/removed, site link changed, firewall rule change, server maintenance, DC reboot. |
| 4. Check monitoring | Replication monitoring dashboards (AD Recycle Bin monitoring, SCOM, custom scripts running `repadmin /replsummary` on schedule). |
| 5. Validate connectivity | Ping all DCs. `Test-NetConnection DC -Port 135`. `Test-NetConnection DC -Port 389`. DNS resolution of all DCs from each DC. |
| 6. Check OS | `repadmin /replsummary` (first thing to run). `repadmin /showrepl` (per-DC detail). `dcdiag /e /v`. |
| 7. Check dependencies | All DCs online? Network between sites? DNS resolution? Firewall rules? Time sync? |
| 8. Check logs | Event ID 1988 (knowledge consistency), 1987 (replication access denied), 2080 (replication diagnostic), Directory Service log. `repadmin` output. |
| 9. Identify root cause | `repadmin /replsummary` shows specific DC with failures. Cause: firewall blocking, DC offline, wrong credentials, DNS. |
| 10. Implement fix | Open firewall port, restart Netlogon, remove/re-add connection object, fix site link, fix DNS, synchronize time, run `repadmin /syncall`. |
| 11. Validate service | `repadmin /replsummary` shows all replications succeeding. `repadmin /showrepl` shows "NTDS Diagnostics ... succeeded." |
| 12. Monitor | Schedule `repadmin /replsummary` script to run every 5 minutes with alerting. |
| 13. Document RCA | Root cause, resolution, and preventive measures (network monitoring between DCs). |

**Interview Questions:**
- "What's your first command for AD replication troubleshooting?"
- "What ports does AD replication use?"
- "What's a USN rollback and how do you prevent it?"
- "How do you force replication between two specific DCs?"

---

### 2.6 — New User Not Syncing

**Concept:** A newly created user account does not appear in AD or does not replicate to other DCs.

**Architecture:** User creation → AD database on creation DC → USN increment → replication to partner DCs → global catalog update.

**Components:** AD database, KCC, GC, replication, FSMO roles (RID Master for SID allocation).

**Configuration:** User creation method (ADUC, PowerShell, LDIF, HRP), OU placement, replication scope.

**Commands:**
```powershell
# Create user:
New-ADUser -Name "John Smith" -SamAccountName jsmith -UserPrincipalName jsmith@domain.com -Path "OU=Users,DC=domain,DC=com" -AccountPassword (ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force) -Enabled $true
# Check user:
Get-ADUser -Identity jsmith -Properties * | Select-Object Name, DistinguishedName, Enabled, LastLogon, LastLogonTimestamp
# Check replication:
repadmin /showrepl
repadmin /syncall
# Check if user is on all DCs:
Get-ADUser -Identity jsmith -Server DC02.domain.com
# Check RID Master:
Get-ADDomainController -Filter * | Select-Object Name, OperationMasterRoles
# Check RID pool:
Get-ADDomainController -Identity DC01 | Select-Object RIDAllocationPoolUsed, RIDPoolAvailable
```

**Real-World Example:** User created on DC01 but not visible on DC02 — replication was broken. `repadmin /syncall` fixed it.

**Common Failures:**
- Replication not completed (timing — wait 15 minutes)
- RID Master FSMO role offline
- RID pool exhausted on a DC
- RID pool not allocated to DC
- AD database full
- Replication failure between DCs

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | User created but doesn't appear? In ADUC but not on GC? After creation or days later? |
| 2. Determine scope | All new users or one? All DCs or specific? |
| 3. Check recent changes | User just created? Which DC? RID Master status? |
| 4. Check monitoring | Replication health (`repadmin /replsummary`). |
| 5. Validate connectivity | DCs can replicate. DNS resolves all DCs. |
| 6. Check OS | `Get-ADUser -Identity jsmith -Server DC01` vs `Get-ADUser -Identity jsmith -Server DC02`. `repadmin /showrepl`. |
| 7. Check dependencies | RID Master operational? RID pool available? Replication working? |
| 8. Check logs | Replication errors in `repadmin`. Event logs for USN/RID issues. |
| 9. Identify root cause | User not replicated (replication failure) or user not created (RID pool exhausted)? |
| 10. Implement fix | Force replication: `repadmin /syncall /AdeP`. If RID pool issue: `Add-ADRIDMasterAllocation`. If creation failure: RID Master FSMO transfer to healthy DC. |
| 11. Validate service | User appears on all DCs and GC: `Get-ADUser -Identity jsmith -Server *` on each DC. |
| 12. Monitor | Verify user sync across DCs. |
| 13. Document RCA | Root cause and resolution. |

**Interview Questions:**
- "I created a user but another DC doesn't see it. What do you check?"
- "What happens if the RID Master is offline?"
- "How do you check RID pool allocation?"

---

### 2.7 — DC Unavailable

**Concept:** A Domain Controller is offline, unreachable, or not responding.

**Architecture:** DC provides authentication, LDAP, DNS, SYSVOL, GC, FSMO roles. Loss of DC reduces redundancy and can cause authentication failure if all DCs down.

**Components:** AD DS role, Netlogon, DNS Server, LDAP, KDC, FSMO roles, AD database (`ntds.dit`).

**Configuration:** DSRM password, directory services restore mode (DSRM), site configuration.

**Commands:**
```powershell
# Check DC status:
Get-ADDomainController -Filter * | Select-Object Name, IPv4Address, OperatingSystem, IsGlobalCatalog, OperationMasterRoles, Enabled
# Test DC:
Test-ComputerSecureChannel -Server DC01.domain.com -Verbose
Test-NetConnection DC01.domain.com -Port 389
Test-NetConnection DC01.domain.com -Port 445
# Check if DC is responding:
nltest /dsgetdc:domain.com
dcdiag /s:DC01
# Restart Netlogon:
Restart-Service Netlogon
# DSRM:
# Boot into DSRM → restore AD → reboot → dcdiag /adrep
```

**Real-World Example:** DC went offline after disk failure — VM restored from backup, AD database consistent, `dcdiag` clean, replication resumed.

**Common Failures:**
- VM/host crash
- Disk failure
- Memory failure causing BSOD
- DC stuck in boot loop
- AD database corruption
- DSRM password forgotten
- IP conflict
- DNS failure (if DC is DNS server)
- Time skew (affects Kerberos)

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | DC unreachable? Slow? Not responding to LDAP? Boot loop? |
| 2. Determine scope | One DC? All DCs? Which roles hosted? |
| 3. Check recent changes | VM maintenance, patch, hardware issue, AD change. |
| 4. Check monitoring | Server monitoring (iLO/iDRAC/SCOM/VMware alerts). |
| 5. Validate connectivity | Ping DC. `Test-NetConnection` on 389, 445, 88, 135, 53. Check VM state in hypervisor. |
| 6. Check OS | Check server OS state. Boot logs. `dcdiag`. |
| 7. Check dependencies | VM running? Network accessible? DNS working? Disks healthy? |
| 8. Check logs | System log, Directory Service log, hardware logs. Check for BSOD (Event ID 41), disk errors (Event ID 51). |
| 9. Identify root cause | Hardware failure? OS crash? AD corruption? Resource exhaustion? |
| 10. Implement fix | Restart VM/service, restore from backup, boot into DSRM for AD repair, failover FSMO roles if needed (`Move-ADDirectoryServerOperationMasterRole`), replace hardware. |
| 11. Validate service | `dcdiag /v` clean. DC responding on all ports. `repadmin /replsummary` healthy. |
| 12. Monitor | DC uptime and health for 7+ days. |
| 13. Document RCA | Root cause, resolution, and preventive action. |

**Interview Questions:**
- "A DC is unreachable. What's your recovery plan?"
- "What FSMO roles can be seized and when?"
- "How do you boot a DC into DSRM?"

---

### 2.8 — SYSVOL Problem

**Concept:** SYSVOL replication fails (DFSR for Win2008R2+), causing GPOs and login scripts to not propagate.

**Architecture:** SYSVOL is replicated via DFSR (since Windows 2008 R2). SYSVOL contains Group Policy templates, login scripts, and netlogon share.

**Components:** DFSR service, SYSVOL share, `netlogon` share, FRS (legacy), AD replication, GPOs.

**Configuration:** DFSR membership, SYSVOL migration state (via `DFSRMig`), DFS namespace configuration.

**Commands:**
```powershell
# Check DFSR status:
Get-Service Dfsr
dfsrdig /puppetget /v:verbose  # verbose DFSR debugging
# Check SYSVOL:
Test-Path C:\Windows\SYSVOL\domain
Test-Path \\domain.com\SYSVOL\domain.com\Policies
# Check DFSR migration state:
DFSRMig /GetMigrationState
DFSRMig /GetGlobalState
# Force DFSR sync:
dfsrdiag PollAD /Member:DC01.domain.com
# Check DFSR backlog:
dfsrdiag Backlog /Member:DC01.domain.com /Folder:SYSVOL Share:SYSVOL
# Event logs:
Get-WinEvent -LogName "Microsoft-Windows-DFS-Replication/Operational" | Where-Object {$_.Id -eq 4012 -or $_.Id -eq 4004} -MaxEvents 50
# Check netlogon:
Test-ComputerSecureChannel -Verbose
nltest /sc_query:domain.com
# If needed, use DfsrMigration tool to move back from Stageing to Authoritative:
DFSRMig /MoveData /GlobalState:Authoritative
```

**Real-World Example:** After a botched DFSR rollback attempt, SYSVOL was stuck in "Auto Recovery" state on one DC — using `DFSRMig` to go through proper state transitions resolved it.

**Common Failures:**
- DFSR service stopped
- DFSR database corrupted
- SYSVOL folder deleted/moved
- AD replication failure (DFSR depends on it)
- Disk full on DC
- DFSR JET database corruption
- Auto restore triggered due to inconsistency
- DFSR stuck in "Dormant" or "Startup" state

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | GPOs not applying? Login scripts not running? SYSVOL share missing? DC stuck in AUTO_RECOVERY? |
| 2. Determine scope | One DC? All DCs? Specific GPOs or all? |
| 3. Check recent changes | SYSVOL folder modified, DFSR service change, AD replication issue, DC restore. |
| 4. Check monitoring | DFSR operational log, SYSVOL share accessibility from clients. |
| 5. Validate connectivity | Client can access `\\domain.com\SYSVOL`. `Test-Path \\domain.com\SYSVOL\domain.com\Policies`. |
| 6. Check OS | `Get-Service Dfsr` (Running?). `Test-Path C:\Windows\SYSVOL\domain`. Check `dfsr.exe` process. |
| 7. Check dependencies | AD replication healthy? DFSR service running? Disk space adequate? |
| 8. Check logs | DFSR Operational log (Event ID 4004: Auto restore, 4012: Pre-existing, 4014: Database corruption detected). System log for DFSR errors. `dcdiag` for SYSVOL check. |
| 9. Identify root cause | DFSR database corrupt? Service stopped? Replication broken? |
| 10. Implement fix | Restart DFSR. If stuck in recovery: use `DFSRMig` commands to move through states. If corruption: restore from backup. If service won't start: rebuild DFSR database (as last resort, following Microsoft guidance). |
| 11. Validate service | SYSVOL share accessible. `dfsrdiag Backlog` shows no backlog. GPOs apply correctly. |
| 12. Monitor | DFSR health monitoring for 7+ days. |
| 13. Document RCA | Root cause, resolution, and process improvement. |

**Interview Questions:**
- "What is DFSR Auto Recovery?"
- "How do you check SYSVOL replication status?"
- "What's the difference between FRS and DFSR for SYSVOL?"

---

### 2.9 — Kerberos Failure

**Concept:** Kerberos authentication fails, causing access denied errors, ticket-granting failures, or constrained delegation issues.

**Architecture:** Kerberos: AS-REQ → KDC → AS-REP (TGT) → Service request → TGS-REQ → TGS-REP (Service Ticket) → Service authentication.

**Components:** KDC service, Kerberos protocol, SPNs (Service Principal Names), Key Distribution Center, Key tab, time synchronization.

**Configuration:** Kerberos policies (ticket lifetime, renewal), SPN registration, delegation settings, clock skew tolerance (default 5 minutes).

**Commands:**
```powershell
# Check Kerberos tickets:
klist
klist tgt
klist purge  # clear tickets
# Test Kerberos authentication:
klist get HTTP/webserver.domain.com
# Test SPN:
setspn -L domain\serviceaccount
setspn -Q HTTP/webserver.domain.com
# Check time sync:
w32tm /query /status
w32tm /query /peers
w32tm /resync
# Check KDC service:
Get-Service KDC
# Kerberos logging:
# Enable via registry: HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters → LogLevel (DWORD, 1 for basic, 2+ for verbose)
# Event Viewer: System log → Kerberos events (Event ID 4, 7, 11, 13, 25, etc.)
Get-WinEvent -FilterHashtable @{LogName='System'; Id=4,7,11,13,25} -MaxEvents 50
```

**Real-World Example:** Users get "Kerberos authentication error" accessing a web app — the SPN was registered under the wrong account (duplicate SPN).

**Common Failures:**
- Clock skew > 5 minutes between client and DC
- Duplicate or missing SPNs
- SPN registered to wrong account
- Expired KDC service certificate
- TGT expired (default 8 hours)
- KDC service not running
- Delegation misconfiguration (constrained/unconstrained)
- AES encryption type disabled
- Double-hop problem (no delegation configured)

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | "Kerberos error"? "Clock skew too large"? "Server not found in Kerberos database"? Access denied to specific service? |
| 2. Determine scope | Single user? All users? Single service? All services? |
| 3. Check recent changes | SPN change, service restart, certificate renewal, password change for service account, time sync issue. |
| 4. Check monitoring | Authentication failures, service availability. |
| 5. Validate connectivity | Client can reach DC on port 88. Time sync: `w32tm /query /status`. DNS resolution of KDC. |
| 6. Check OS | `klist` for tickets. `klist tgt` for TGT. Check time sync on client and DC. `Test-NetConnection DC -Port 88`. |
| 7. Check dependencies | KDC service running? SPN registered correctly? Time synced? Certificate valid? |
| 8. Check logs | Kerberos events in System log (IDs 4, 7, 11, 13). `klist` output. Event IDs related to "Clock skew" (4), "Server not found" (7). |
| 9. Identify root cause | Duplicate SPN? Wrong SPN? Time skew? KDC offline? |
| 10. Implement fix | Fix SPN (`setspn -D` then `setspn -S`), sync time, restart KDC, update service account password and re-register SPN, configure proper delegation, enable correct encryption types. |
| 11. Validate service | `klist` shows valid TGT. User can access target service. `klist get HTTP/service` succeeds. |
| 12. Monitor | Kerberos event logs for repeat failures. |
| 13. Document RCA | Root cause, resolution, and preventive action (e.g., document SPN management procedure). |

**Interview Questions:**
- "What's a duplicate SPN and how does it break Kerberos?"
- "How do you troubleshoot the double-hop problem?"
- "What's clock skew in Kerberos and how do you fix it?"

---

## 🔷 DOMAIN 3: DNS / DHCP (⭐⭐⭐⭐⭐)

---

### 3.1 — DNS Resolution Failure

**Concept:** DNS queries fail or return incorrect results, preventing hostname-to-IP resolution.

**Architecture:** DNS query: Client → DNS Client service → DNS server (recursive query) → root hints/forwarders → authoritative server → response.

**Components:** DNS server service, DNS zones (primary, secondary, stub), forwarding, root hints, DNS cache, DNSSEC, DNS over HTTPS/TLS.

**Configuration:** Forwarders, root hints, zone type, zone transfers, DNSSEC settings, scavenging.

**Commands:**
```powershell
# Test DNS:
Resolve-DnsName google.com -Server 8.8.8.8
Resolve-DnsName google.com
nslookup google.com
nslookup google.com 8.8.8.8
# Check DNS server:
Get-DnsServer
Get-DnsServerSetting
# Check zones:
Get-DnsServerZone
Get-DnsServerResourceRecord -ZoneName "domain.com" -Name "www"
# Check cache:
Clear-DnsClientCache
# Check DNS resolution order:
Get-DnsClientGlobalSetting
# Query specific record types:
Resolve-DnsName www.google.com -Type A
Resolve-DnsName _dmarc.google.com -Type TXT
Resolve-DnsName _kerberos._tcp.domain.com -Type SRV
# Recursive query test:
Resolve-DnsName root.zone
# Forwarders:
Set-DnsServerForwarder -IPAddress 8.8.8.8, 8.8.4.4
# Check DNS server logs:
# Event Viewer: DNS Server log (if enabled, Event ID 5000+)
Get-WinEvent -FilterHashtable @{LogName='DNS Server'; Id=5000} -MaxEvents 20
```

**Real-World Example:** Internal users couldn't resolve `intranet.company.com` — DNS zone missing from primary DNS server after failed zone transfer.

**Common Failures:**
- DNS server service stopped
- Incorrect DNS server configured on client (ISP DNS instead of internal)
- Firewall blocking DNS (port 53)
- Zone missing/corrupt
- Zone transfer failure (secondary DNS)
- Forwarder misconfigured/unreachable
- Root hints missing (if no forwarders)
- DNS cache poisoning
- DNSSEC validation failure
- Scavenging removing records too aggressively

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Hostname not resolving? Wrong IP returned? Intermittent? For all names or specific? |
| 2. Determine scope | One client? All clients? Specific zone? Internet names or internal? |
| 3. Check recent changes | DNS server change, zone deleted, forwarder changed, DNSSEC changed, server reboot. |
| 4. Check monitoring | DNS server availability, query success rate, response time. |
| 5. Validate connectivity | Client can reach DNS server? `Test-NetConnection DNSServer -Port 53`. |
| 6. Check OS | `nslookup`, `Resolve-DnsName`, `Resolve-DnsName -Server IP`. `Get-DnsClientGlobalSetting` (DNS servers in order). `ipconfig /all`. `nslookup` from multiple clients. |
| 7. Check dependencies | DNS server running? Zone exists? Forwarders reachable? Firewall allows 53? |
| 8. Check logs | DNS Server log (if enabled), Event Viewer DNS Server events, query failed events. `Resolve-DnsName` with `-Debug` flag. |
| 9. Identify root cause | Client pointing to wrong DNS? Zone missing? Forwarder down? Firewall blocking? |
| 10. Implement fix | Correct DNS server on client (`Set-DnsClientServerAddress`), restart DNS server service, recreate zone, fix forwarders, open firewall, run `ipconfig /flushdns` on client. |
| 11. Validate service | `nslookup intranet.company.com` returns correct IP from internal DNS. Multiple clients resolve correctly. |
| 12. Monitor | DNS resolution monitoring, query success rate. |
| 13. Document RCA | Root cause, resolution, prevention. |

**Interview Questions:**
- "How do you differentiate between DNS, network, and firewall issues when a name doesn't resolve?"
- "What's the difference between recursive and iterative DNS queries?"
- "How do you check if a DNS zone is properly configured?"

---

### 3.2 — Reverse Lookup Failure

**Concept:** Reverse DNS (PTR) lookups fail — IP-to-hostname resolution doesn't work.

**Architecture:** Reverse lookup uses `in-addr.arpa` zones (IPv4) or `ip6.arpa` zones (IPv6). PTR records map IPs to names.

**Components:** Reverse lookup zones, PTR records, PTR zone replication, dynamic updates.

**Configuration:** PTR zone configuration, dynamic update settings, zone type, replication scope.

**Commands:**
```powershell
# Test reverse lookup:
Resolve-DnsName -Name 192.168.1.10 -Type PTR
nslookup 192.168.1.10
# Check reverse zones:
Get-DnsServerZone | Where-Object {$_.ZoneName -like "*in-addr.arpa*"}
# Check PTR records:
Get-DnsServerResourceRecord -ZoneName "1.168.192.in-addr.arpa" -Name 10
# Check if reverse zone exists:
Get-DnsServerZone -Name "1.168.192.in-addr.arpa"
# If missing, create:
Add-DnsServerPrimaryZone -Name "1.168.192.in-addr.arpa" -ReplicationScope "Domain"
# Add PTR record:
Add-DnsServerResourceRecord -ZoneName "1.168.192.in-addr.arpa" -Name 10 -Ptr -RecordData "server01.domain.com"
# Check DHCP DNS registration:
Get-DhcpServerv4Lease -ScopeId 192.168.1.0 | Select-Object IPAddress, HostName
```

**Real-World Example:** Email server being flagged as spam because reverse DNS for its IP wasn't configured — PTR record added to reverse zone.

**Common Failures:**
- Reverse zone missing entirely
- PTR record missing
- PTR record points to wrong hostname
- Reverse zone not replicating to other DNS servers
- DHCP not registering PTR records (dynamic update disabled)
- Reverse zone transfer failure

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | `nslookup IP` fails or returns wrong name? Email issues? Logging issues? |
| 2. Determine scope | Specific IP or range? All reverse zones? One DNS server or all? |
| 3. Check recent changes | Subnet change, server re-IP, DHCP scope change, DNS server change. |
| 4. Check monitoring | Reverse zone health, PTR record count vs. DHCP leases. |
| 5. Validate connectivity | DNS server responding on port 53. |
| 6. Check OS | `nslookup IP` on multiple DNS servers. Check if reverse zone exists: `Get-DnsServerZone`. |
| 7. Check dependencies | Reverse zone present? PTR record exists? DHCP configured to register PTR? DNS update allowed? |
| 8. Check logs | DNS Server log for query failures. Zone transfer logs. |
| 9. Identify root cause | Reverse zone missing? PTR record deleted? DHCP not updating PTR? |
| 10. Implement fix | Create reverse zone, add PTR record, configure DHCP to register PTR (`Set-DhcpServerv4OptionValue -DnsDomain` and enable dynamic updates). |
| 11. Validate service | `nslookup 192.168.1.10` returns correct hostname from any DNS server. |
| 12. Monitor | PTR records vs DHCP leases match. Reverse zone health. |
| 13. Document RCA | Root cause, resolution, process (e.g., PTR records must be maintained as part of server deployment). |

**Interview Questions:**
- "How do you troubleshoot reverse DNS failures?"
- "Why does reverse DNS matter for email?"
- "How do you automate PTR record creation?"

---

### 3.3 — AD Clients Receiving Wrong DNS

**Concept:** Domain-joined computers are pointing to incorrect DNS servers (e.g., ISP DNS instead of internal AD-integrated DNS).

**Architecture:** DHCP assigns DNS server addresses via Option 006. DHCP relies on AD for authorization (DHCP in AD). Clients register their DNS records via dynamic update.

**Components:** DHCP server, DHCP scopes, DNS server, DHCP relay agents, DHCP authorization in AD, DNS records (A, PTR).

**Configuration:** DHCP Option 006 (DNS servers), DHCP authorization in AD, DNS dynamic update settings, DHCP relay agent configuration.

**Commands:**
```powershell
# Check DHCP server:
Get-DhcpServer
Get-DhcpServerv4Scope
# Check Option 006:
Get-DhcpServerv4OptionValue -ScopeId 192.168.1.0 -OptionId 6
# Check DHCP authorization:
# DHCP console → right-click server → Authorize (or via PowerShell)
# Check DNS records for DHCP servers:
Resolve-DnsName dhcp1.domain.com
# Check DNS is AD-integrated:
Get-DnsServerZone | Where-Object {$_.ReplicationScope -ne "None"}
# Check relay agent:
Get-DhcpServerv4RelayAgent
# Force DHCP renewal on client:
ipconfig /release
ipconfig /renew
ipconfig /registerdns
# Check client DNS settings:
Get-DnsClientServerAddress
Resolve-DnsName -Name "domain.com" -Server 8.8.8.8  # bypass internal DNS to check if external resolves
```

**Real-World Example:** DHCP relay agent on a VLAN switch pointed to wrong IP — all clients in that VLAN got ISP DNS instead of internal DNS.

**Common Failures:**
- DHCP Option 006 configured with wrong DNS server IPs
- DHCP relay agent pointing to wrong DHCP server
- DHCP server not authorized in AD (DHCP not serving in AD unless authorized — wait, this is about authorization)
- DNS records for DHCP server missing (DHCP FQDN record)
- Multiple DHCP servers conflicting
- Client configured with static DNS (not via DHCP)

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Clients getting wrong DNS? Specific VLAN or all? Internet working but AD not? |
| 2. Determine scope | All clients or specific subnet/VLAN? |
| 3. Check recent changes | DHCP Option changed, relay agent changed, DHCP server moved. |
| 4. Check monitoring | DHCP server running, leases being issued. |
| 5. Validate connectivity | Clients can reach DHCP server. DHCP relay agent operational. |
| 6. Check OS | On client: `ipconfig /all` (DNS servers). `Get-DnsClientServerAddress`. On DHCP server: `Get-DhcpServerv4OptionValue -OptionId 6`. |
| 7. Check dependencies | DHCP authorized in AD? DHCP relay correct? DNS server operational? |
| 8. Check logs | DHCP server log (Event ID 20279: DHCP/BINLOG/DHCP logged), DNS events, Event ID for relay. |
| 9. Identify root cause | Option 006 has wrong IP? Relay agent misconfigured? DHCP server not authorized? |
| 10. Implement fix | Correct Option 006 to point to internal DNS servers (`Set-DhcpServerv4OptionValue -DnsServer 192.168.1.10, 192.168.1.11`). Correct relay agent. Authorize DHCP server in AD. Renew client leases. |
| 11. Validate service | Client `ipconfig /all` shows correct DNS servers. `nslookup` works for internal names. AD queries succeed. |
| 12. Monitor | DHCP Option compliance check. Random client DNS checks. |
| 13. Document RCA | Root cause, fix, and preventive monitoring. |

**Interview Questions:**
- "AD clients getting wrong DNS — what's your approach?"
- "What DHCP option specifies DNS servers?"
- "Why does DHCP need to be authorized in AD?"

---

### 3.4 — DHCP Scope Exhausted

**Concept:** DHCP scope runs out of available IP addresses, preventing new devices from getting leases.

**Architecture:** DHCP server manages IP address pools (scopes), leases them to clients with a lease duration, and reclaims expired leases.

**Components:** DHCP scopes, IP address ranges, exclusions, reservations, lease database, superscopes, multihoming, failover pairs.

**Configuration:** Scope range, subnet mask, exclusions, lease duration, reservations, superscope/failover configuration.

**Commands:**
```powershell
# Check scope:
Get-DhcpServerv4Scope | Select-Object ScopeId, Name, StartRange, EndRange, SubnetMask, State, LeaseDuration
# Check used/free addresses:
Get-DhcpServerv4ScopeStatistics -ScopeId 192.168.1.0
# Check leases:
Get-DhcpServerv4Lease -ScopeId 192.168.1.0
# Check free IPs:
Get-DhcpServerv4FreeIP -ScopeId 192.168.1.0
# Add scope:
Add-DhcpServerv4Scope -Name "New Scope" -StartRange 10.0.0.1 -EndRange 10.0.0.254 -SubnetMask 255.255.255.0
# Export leases:
Get-DhcpServerv4Lease -ScopeId 192.168.1.0 | Export-Csv C:\leases.csv
# Renew all leases in a scope:
Invoke-DhcpServerv4Superscope -ScopeId 192.168.1.0, 192.168.2.0  # for superscope
# Clean up old leases:
# DHCP console → Right-click scope → Delete Old Scavenged Leases (or via API)
```

**Real-World Example:** Guest Wi-Fi DHCP scope was /24 but 500+ devices connected — implemented MAC-based filtering and split into two scopes via superscope with a smaller range.

**Common Failures:**
- Scope too small for number of devices
- Lease duration too long (addresses held too long)
- Devices not releasing leases (mobile devices, BYOD)
- Scope not properly excluding reserved IPs (routers, servers, printers)
- Superscope not configured for multi-VLAN
- DHCP server database corrupted
- Failover partner not synced

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | New devices can't get IP? DHCPDECLINE/ACK failures? |
| 2. Determine scope | Which scope(s)? How many addresses used vs. total? |
| 3. Check recent changes | New devices added, scope range reduced, lease duration changed. |
| 4. Check monitoring | DHCP scope utilization % (threshold alerts at 70-80%). |
| 5. Validate connectivity | DHCP server operational. DHCP relay (if applicable) working. |
| 6. Check OS | `Get-DhcpServerv4ScopeStatistics`. `Get-DhcpServerv4Lease` count vs. scope range. `Get-DhcpServerv4FreeIP`. |
| 7. Check dependencies | Scope range sufficient? Exclusions correct? Failover partner synced? |
| 8. Check logs | DHCP server log (Event ID 20279, 20311, etc.). Check lease database. |
| 9. Identify root cause | Scope full (95%+ utilized). Lease duration too long. Too many devices. |
| 10. Implement fix | Expand scope, reduce lease duration (e.g., from 8 hours to 2 hours for guest), clean up old leases, add new scope, superscope configuration. |
| 11. Validate service | Free IP count above 20%. New device gets valid IP via `ipconfig /renew`. |
| 12. Monitor | Scope utilization dashboard. Alert at 70%. |
| 13. Document RCA | Root cause, fix, and capacity planning documentation. |

**Interview Questions:**
- "DHCP scope exhausted — what do you do immediately?"
- "What's a superscope and when do you use it?"
- "How do you clean up stale DHCP leases?"

---

### 3.5 — Client Receives APIPA

**Concept:** Client receives 169.254.x.x address (APIPA/Link-Local) instead of DHCP-assigned address.

**Architecture:** When DHCP is unreachable after retries, Windows auto-assigns a 169.254.0.0/16 address (Automatic Private IP Addressing). Client on local subnet only — no internet or AD access.

**Components:** DHCP client service, DHCP server, DHCP relay, network connectivity, network cable, switch port, VLAN configuration.

**Configuration:** DHCP client settings, network adapter settings (should be DHCP by default).

**Commands:**
```powershell
# Check IP config:
ipconfig /all
Get-NetIPAddress -AddressFamily IPv4 | Where-Object {$_.IPAddress -like "169.254.*"}
# Check DHCP client service:
Get-Service Dhcp
# Release and renew:
ipconfig /release
ipconfig /renew
# Check DHCP events:
Get-WinEvent -FilterHashtable @{LogName='System'; Id=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1011,1012,1013,1014,1015,1016,1017,1018,1019,1020,1021,1022,1023,1024,1025,1026,1027,1028,1029,1030,1031,1032,1033,1034,1035,1036,1037,1038,1039,1040,1041,1042,1043,1044,1045,1046,1047,1048,1049,1050,1051,1052,1053,1054,1055,1056,1057,1058,1059,1060,1061,1062,1063,1064,1065,1066,1067,1068,1069,1070} -MaxEvents 50
# Better: Check Event IDs 1001-1059 in System log (DHCP client events)
# Check if DHCP server reachable:
Test-NetConnection -ComputerName DHCP_SERVER_IP -Port 67
# Check DHCP relay (if different subnet):
Get-DhcpServerv4RelayAgent
```

**Real-World Example:** Conference room PC keeps getting APIPA — switch port had STP (Spanning Tree) blocking, preventing DHCP discover packets.

**Common Failures:**
- DHCP server down
- Network cable disconnected/ faulty
- Switch port disabled/blocked (STP)
- DHCP scope exhausted
- DHCP relay agent not configured (different subnet)
- VLAN mismatch between client and DHCP server
- Firewall blocking DHCP (ports 67/68)
- DHCP server not authorized in AD
- DHCP service crashed

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Client has 169.254.x.x IP. When did this start? One client or multiple? |
| 2. Determine scope | Single client or multiple clients in same location? |
| 3. Check recent changes | Network change, switch config change, DHCP server maintenance. |
| 4. Check monitoring | DHCP server running? Scope utilization? Network monitoring. |
| 5. Validate connectivity | Check physical connectivity (cable, port). `Test-NetConnection DHCPserver -Port 67`. `Test-NetConnection DHCPserver -Port 53`. |
| 6. Check OS | `ipconfig /all` (confirms APIPA). Check network adapter settings (DHCP enabled). Run `ipconfig /release && ipconfig /renew` and observe result. |
| 7. Check dependencies | Network cable/link up? Switch port enabled? VLAN correct? DHCP server reachable? Relay configured? |
| 8. Check logs | System log: DHCP Client events (Event IDs 1001-1059). Event ID 1001 (DHCP Discover sent, no response), Event ID 1002 (DHCP Offer received but rejected), etc. DHCP server log. |
| 9. Identify root cause | DHCP server unreachable? Network issue? Scope full? Relay missing? |
| 10. Implement fix | Fix network connectivity, restart DHCP service, expand scope, configure DHCP relay, fix VLAN, replace cable, enable switch port. |
| 11. Validate service | `ipconfig /renew` returns 192.168.x.x or other valid IP. `ipconfig /all` shows DHCP server and lease information. |
| 12. Monitor | Network connectivity monitoring, DHCP monitoring. |
| 13. Document RCA | Root cause, fix, and preventive action (e.g., DHCP relay on all inter-VLAN routers). |

**Interview Questions:**
- "A PC has 169.254.x.x — what's your approach?"
- "What Event IDs in the System log relate to DHCP client failures?"
- "How is APIPA different from a manually configured IP?"

---

## 🔷 DOMAIN 4: VMWARE vSPHERE (⭐⭐⭐⭐⭐)

---

### 4.1 — VM Inaccessible

**Concept:** A virtual machine becomes inaccessible from vCenter/ESXi — unresponsive, inaccessible, or missing.

**Architecture:** VM running on ESXi host, managed by vCenter. VM files (`.vmx`, `.vmdk`, `.nvram`, `.vmss`) stored on datastores. vCenter communicates with ESXi hosts via vSphere API.

**Components:** VM config file (`.vmx`), virtual disk files (`.vmdk`), snapshot files (`.vmsn`), ESXi host, vCenter, datastore, VMkernel port, VM tools.

**Configuration:** VM resource allocation (CPU, memory, reservations), VM tools status, VM disk controller type (PVSCSI/LSI Logic), hardware version.

**Commands:**
```powershell
# On ESXi host via SSH:
vim-cmd vmsvc/getallvms  # list VMs
vim-cmd vmsvc/power.getstate <vmid>  # get power state
vim-cmd vmsvc/power.reset <vmid>  # reset VM
vim-cmd vmsvc/power.off <vmid>  # force off
ls -la /vmfs/volumes/<datastore>/<vmname>/  # list VM files
# Check VM files:
# .vmx - configuration
# .vmdk - disk descriptor and data extent
# .vmsn - snapshot state
# .vmss - suspended state
# Repair VM (if .vmx corrupt):
# On ESXi: vmkfstools to check VMDK
vmkfstools -y /vmfs/volumes/datastore/vmname/vmname.vmdk  # repair
# On vCenter:
# Right-click VM → Register VM (if unregistered)
# Check host connection:
Get-VMHost | Select-Object Name, ConnectionState, State
# vSphere Client (HTML5) — check VM status, task/error details
# PowerCLI:
Get-VM -Name VMName | Select-Object Name, PowerState, GuestId, Version, VMHost
Get-VM VMName | Get-View | Select-Object Config.Name, Config.Hardware.Device, PowerState
# Check for orphaned VM:
# SSH to ESXi: tail -f /var/log/vmkernel.log and /var/log/vmware/hostd.log
```

**Real-World Example:** VM inaccessible after host HA restart — `.vmx` file was on a datastore that became inaccessible. Moved VM to another datastore via host-level operations.

**Common Failures:**
- Datastore inaccessible (storage path down)
- `.vmx` file corrupted
- `.vmdk` file corrupt or missing
- VM kernel panic on host
- vCenter agent not responding
- Host disconnected from vCenter
- VM file lock (stale lock from crashed VM)
- Insufficient resources (memory/CPU)
- VM hardware version incompatible with host

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | VM inaccessible in vCenter? Unresponsive? Missing? Specific VM or multiple? |
| 2. Determine scope | One VM? Multiple on same host? All on same datastore? |
| 3. Check recent changes | Host maintenance, storage change, VM hardware upgrade, snapshot operations, HA event. |
| 4. Check monitoring | Host status in vCenter. Datastore accessibility. VM health. HA events. |
| 5. Validate connectivity | Can you reach the host via SSH/API? Can you access the datastore? Network to storage? |
| 6. Check OS | On host: `vim-cmd vmsvc/getallvms` to find VM ID. Check VM files on datastore. Check if VM is in power state. |
| 7. Check dependencies | Datastore accessible? Host running? VM tools? VM files intact? |
| 8. Check logs | ESXi host logs: `/var/log/vmkernel.log`, `/var/log/vmware/hostd.log`, `/var/log/vpxa.log`. vCenter logs: `vpxd.log`. VM logs: `/var/log/vmkernel.log` for VM-related events. |
| 9. Identify root cause | Datastore unreachable? .vmx corrupt? Host disconnected? Stale VM kernel lock? |
| 10. Implement fix | Reconnect datastore, re-register VM (if unregistered), restore .vmx from snapshot/backup, force off VM, migrate VM to another host/datastore, delete corrupt snapshot, restart management agents (`services.sh restart`). |
| 11. Validate service | VM accessible in vCenter, power state correct, VM boots, VM tools running. |
| 12. Monitor | VM availability and performance for 24-48 hours. |
| 13. Document RCA | Root cause, resolution, and prevention (e.g., ensure datastores have multipathing). |

**Interview Questions:**
- "VM inaccessible in vCenter but running on host — what do you do?"
- "How do you recover a VM with a corrupt .vmx file?"
- "What's a VMkernel lock and how do you clear it?"

---

### 4.2 — Host Disconnected

**Concept:** An ESXi host disconnects from vCenter, becomes unmanaged, or loses management connectivity.

**Architecture:** vCenter connects to ESXi hosts via vSphere Management Agent (vpxa). ESXi runs management agents (hostd, vpxa) on VMkernel ports.

**Components:** vCenter Server, ESXi host, vSphere Management Network, VMkernel port group, management agents (vpxa, hostd), SSL certificates, vSphere Client.

**Configuration:** Management network configuration (VMkernel port, IP, DNS), agent services, SSL certificate mode (thumbprint, common name), vCenter trust settings.

**Commands:**
```powershell
# On ESXi host (SSH):
# Check services:
/etc/init.d/vpxa status
/etc/init.d/hostd status
# Restart agents:
/etc/init.d/vpxa restart
/etc/init.d/hostd restart
services.sh restart
# Check management network:
esxcli network ip interface ipv4 get
esxcli network ip dns searchdomain get
esxcli network ip dns server list
# Check vpxa logs:
tail -f /var/log/vpxa.log
# Check vCenter:
# vSphere Client → Hosts and Clusters → check host status
# PowerCLI:
Get-VMHost | Select-Object Name, ConnectionState, PowerState, Manufacturer
# Reconnect host:
Connect-VMHost -Server ESXi01.domain.com -Credential (Get-Credential)
# If certificate issue:
# Check thumbprint:
Get-VMHost | Select-Object Name, Thumbprint, Certificate
# Accept thumbprint:
Set-VMHost -VMHost ESXi01.domain.com -Thumbprint "THUMBPRINT" -Confirm:$false
# Check ESXi health:
esxcli system version get
esxcli hardware cpu get
esxcli storage filesystem list
# Check for hostd issues:
tail -f /var/log/hostd.log
```

**Real-World Example:** Host disconnected after SSL certificate renewal — thumbprint mismatch. Reconnect with `Connect-VMHost` and accept new thumbprint resolved it.

**Common Failures:**
- vpxa/hostd service crashed
- Management network down (VMkernel port down, IP conflict, VLAN misconfig)
- DNS resolution failure for vCenter or ESXi
- SSL certificate expired/thumbprint mismatch
- vCenter overloaded (can't handle host connections)
- Network partition (isolated host)
- Management agents stuck after host reboot
- Firewall blocking management traffic
- Host overloaded (CPU/memory starvation)

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Host shows "Disconnected" in vCenter? Multiple hosts? After specific action? |
| 2. Determine scope | One host? Multiple? During maintenance? After vCenter restart? |
| 3. Check recent changes | Certificate renewal, vCenter update, network change, host reboot. |
| 4. Check monitoring | Host heartbeat in vCenter. ESXi host resource usage. Network monitoring. |
| 5. Validate connectivity | Ping ESXi management IP from vCenter server and vice versa. `Test-NetConnection ESXiIP -Port 443, 902`. `Test-NetConnection vcenter.domain.com -Port 443`. |
| 6. Check OS | On ESXi: SSH → `/etc/init.d/vpxa status`, `esxcli network ip interface ipv4 get`. Check if management network configured correctly. |
| 7. Check dependencies | vCenter reachable? DNS resolves both directions? Management VMkernel port up? Certificates valid? |
| 8. Check logs | ESXi: `/var/log/vpxa.log`, `/var/log/hostd.log`, `/var/log/syslog`. vCenter: `vpxd.log`, `vpxd-alert.log`. Check for SSL/certificate errors. |
| 9. Identify root cause | Management network down? vpxa crashed? Certificate expired? DNS issue? |
| 10. Implement fix | Restart vpxa/hostd, fix management network (VMkernel port, VLAN, IP, DNS), renew/replace certificate, restart management agents (`services.sh restart`), reconnect host in vCenter, verify DNS. |
| 11. Validate service | Host connected in vCenter. All VMs managed. vpxa service running. `Test-NetConnection` on management ports passes. |
| 12. Monitor | Host connection stability for 24-48 hours. |
| 13. Document RCA | Root cause, fix, preventive action (e.g., implement certificate auto-renewal, add redundant management VMkernel). |

**Interview Questions:**
- "ESXi host disconnected from vCenter — what's your troubleshooting approach?"
- "What ports need to be open between ESXi and vCenter?"
- "How do you handle SSL thumbprint issues when reconnecting hosts?"

---

### 4.3 — Datastore Full

**Concept:** A VMFS/NFS datastore runs out of space, preventing VM operations (power on, snapshot, migration).

**Architecture:** ESXi hosts access shared storage via VMFS (block-level, iSCSI/FC/NFS) or NFS mounts. VM files (VMDKs, snapshots, logs) consume datastore space.

**Components:** Datastore (VMFS/NFS), VM disk files (.vmdk), snapshot files (.vmsn), log files (.log), swap files (.vswp), ESXi hosts, storage array/LUN.

**Configuration:** Datastore size, VM disk provisioning (thick/thin), snapshot retention policy, log rotation, VMDK size limit (2TB for VMFS-5).

**Commands:**
```powershell
# Check datastore space:
On ESXi:
df -h  # overall filesystem
vdf -h  # more detailed
esxcli storage filesystem list
# On vCenter/PowerCLI:
Get-Datastore | Select-Object Name, CapacityGB, FreeSpaceGB, Type, ProvisionedSpaceGB | Sort-Object FreeSpaceGB
Get-Datastore -Name Datastore1 | Get-HtmlReport  # if PowerCLI reporting module available
# Check what's consuming space:
# SSH to ESXi host:
cd /vmfs/volumes/Datastore1/
du -sh * | sort -rh | head -20
# For each VM:
du -sh /vmfs/volumes/Datastore1/VMName/*
# List all VMDKs:
find /vmfs/volumes/Datastore1/ -name "*.vmdk" -exec ls -la {} \; | sort -k5 -rn | head -30
# Remove snapshot (if consuming space):
# vSphere Client: Right-click VM → Snapshot → Delete All
# PowerCLI:
Get-VM VMName | Get-Snapshot | Remove-Snapshot -Confirm:$false
# If datastore full and can't power on VM:
# SSH to host, navigate to VM directory:
# Move snapshot delta files or delete them carefully (only if committed)
vmkfstools -y /vmfs/volumes/Datastore1/VMName/VMName-flat.vmdk  # check/repair
# Clean up old ISOs, templates, logs:
find /vmfs/volumes/Datastore1/ -name "*.iso" -delete
find /vmfs/volumes/Datastore1/ -name "*.log" -mtime +30 -delete  # delete old VM log files
# Expand datastore:
# Via storage array: extend LUN, then in ESXi: rescan
esxcli storage core adapter rescan --all
# Resize VMFS:
# vSphere Client → Datastore → Properties → Increase size
```

**Real-World Example:** Production datastore hit 98% — 150+ old VM snapshots from testing consumed 400GB. Consolidated all snapshots and implemented snapshot lifecycle policy.

**Common Failures:**
- VM snapshots left too long (delta files growing)
- Thin provisioning with overcommitted storage
- Log files not rotated (VM log files can grow large)
- Old ISOs/templates/OVA files left on datastore
- VMDK file too large (approaching 2TB limit)
- All datastores in cluster full (storage array issue)
- VM swapping (`.vswp` files) consuming space
- Replication or backup staging on datastore

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Can't power on VMs? Snapshot failures? "No space left on device"? Which datastore? |
| 2. Determine scope | One datastore? All in cluster? Specific VMs consuming space? |
| 3. Check recent changes | VMs added, snapshots created, thin provisioning increase, log verbosity increased. |
| 4. Check monitoring | Datastore free space alerts (typically 15-20% threshold). Storage array monitoring. |
| 5. Validate connectivity | Datastore accessible to all hosts? Multipathing OK? Storage array healthy? |
| 6. Check OS | `df -h` on ESXi. `vdf -h`. `Get-Datastore | Select-Object Name, FreeSpaceGB, CapacityGB`. On datastore: `du -sh * | sort -rh`. |
| 7. Check dependencies | Storage array has space? LUN not full at array level? Multipathing operational? |
| 8. Check logs | ESXi storage logs (`/var/log/vmkernel.log` for storage errors). Event ID 2135 (datastore full). vCenter alarms for datastore space. |
| 9. Identify root cause | Snapshots consuming 400GB? Specific VM with huge VMDK? Log files? Thin provisioning? |
| 10. Implement fix | Delete old snapshots (commit first), delete old ISOs/templates/logs, expand datastore (LUN extension + VMFS resize), move VMs to larger datastore, enable Storage DRS (if vSAN/NFS), increase snapshot retention policy, configure log rotation, reclaim space via `vmkfstools` (thin provision reclaim). |
| 11. Validate service | Free space above 20%. VMs can be powered on and snapshotted. |
| 12. Monitor | Datastore capacity monitoring with alerts at 70% and 85%. |
| 13. Document RCA | Root cause, cleanup, and process (snapshot policy, storage SLA, capacity planning). |

**Interview Questions:**
- "Datastore full — what's your first step?"
- "How do you identify which VM is consuming the most space?"
- "What's the risk of deleting a snapshot delta file?"
- "Thin vs. thick provisioning — what are the space implications?"

---

### 4.4 — vMotion Failure

**Concept:** Live migration (vMotion) of a running VM from one ESXi host to another fails.

**Architecture:** vMotion transfers VM's memory state from source host to destination host over VMkernel network. VM is paused, memory copied iteratively, then resumed on destination with brief downtime.

**Components:** VMkernel port with vMotion enabled, VMkernel network (dedicated vMotion network), VM memory, VM configuration, shared storage (for non-Storage vMotion), destination host resources.

**Configuration:** vMotion enabled on VMkernel port, vMotion VMkernel adapter on each host, sufficient bandwidth, shared storage (for vMotion without storage change), CPU compatibility settings.

**Commands:**
```powershell
# Check vMotion enabled on host:
Get-VMHost | Select-Object Name,@{N='vMotionEnabled';E={$_.ExtensionData.Config.Network.Config.VmotionEnabled}}
# Check VMkernel adapters:
Get-VMHostNetworkAdapter -VMHost VMHost01 | Where-Object {$_.ExtensionData.Config.UplinkPortgroup -or $_.ExtensionData.Config.IpConfig.IpAddress} | Select-Object Name, IPAddress, PortGroupName, VMHost
# Check vMotion VMkernel:
Get-VMHostNetworkAdapter -VMHost VMHost01 | Where-Object {$_.ExtensionData.Config.UplinkPortgroup} | Select-Object Name, @{N='vMotion';E={$_.ExtensionData.Config.UplinkPortgroup}}
# Actually:
$vmhost.ExtensionData.Config.Network.Config.VmotionEnabled
# Check if VM can vMotion:
Get-VM -Name VMName | Move-VM -Destination VMHost02 -ErrorAction SilentlyContinue -ValidateOnly
# Test vMotion:
# vSphere Client: Right-click VM → Migrate → Change compute resource only → Select host
# PowerCLI with error:
Move-VM -Name VMName -Destination VMHost02 -RunAsync
# Check vMotion logs:
# On ESXi: /var/log/vmkernel.log (look for "vMotion" entries)
# vCenter logs: /var/log/vpxd.log
# Check VMkernel network:
Test-NetConnection -ComputerName DestinationHostVMkernelIP -Port 8000  # vMotion uses port 8000
# Check host resources:
Get-VMHost VMHost02 | Select-Object Name, MemoryUsageGB, MemoryTotalGB, CpuUsageMHz, CpuTotalMhz
# Check CPU compatibility:
Get-VMHost VMHost01, VMHost02 | Select-Object Name, @{N='CpuCompatibility';E={$_.ExtensionData.Hardware.CpuInfo}}
# For EVC mode:
Get-Cluster -Name ClusterName | Select-Object Name, DrsAutomationLevel, EVCMode
```

**Real-World Example:** vMotion failing with "Insufficient resources" — destination host had memory overcommit. Resolved by adding memory or changing DRS automation level.

**Common Failures:**
- Insufficient resources on destination host (memory/CPU)
- vMotion VMkernel not configured or network issue
- Storage not shared (if not doing Storage vMotion)
- CPU compatibility issue (EVC mode or CPU features mismatch)
- VM has snapshot (vMotion with snapshot needs special handling)
- vMotion not enabled on host or VMkernel
- Port 8000 blocked by firewall
- Destination host has too many VMs (overcommit)
- VM hardware version incompatible with destination host
- SSL certificate issues

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | vMotion job fails. Error message? During migration or at start? |
| 2. Determine scope | One VM? All VMs? Specific hosts? |
| 3. Check recent changes | Host added, network change, EVC mode change, resource change. |
| 4. Check monitoring | vCenter tasks/alerts. Host resource utilization. Network throughput. |
| 5. Validate connectivity | VMkernel vMotion network: `Test-NetConnection DestinationVMkernelIP -Port 8000`. Ping between VMkernel interfaces. Check VMkernel port group. |
| 6. Check OS | On source and destination hosts: vMotion enabled? VMkernel IP configured? Check via: `Get-VMHost | Select-Object Name, @{N='vMotion';E={$_.ExtensionData.Config.Network.Config.VmotionEnabled}}`. |
| 7. Check dependencies | Shared storage? Destination host has enough resources? CPU compatible? |
| 8. Check logs | vmkernel.log on both hosts during vMotion attempt. vpxd.log. Error in vCenter task. `Move-VM -ValidateOnly` for pre-check. |
| 9. Identify root cause | Destination host low memory? vMotion network down? EVC mode mismatch? Firewall blocking? |
| 10. Implement fix | Free resources on destination, fix VMkernel network, configure EVC mode, enable vMotion on correct VMkernel, open firewall port, remove snapshot first, increase vMotion network bandwidth. |
| 11. Validate service | vMotion succeeds for test VM. Multiple vMotions work reliably. |
| 12. Monitor | vMotion success rate. Network throughput. |
| 13. Document RCA | Root cause, fix, and preventive action. |

**Interview Questions:**
- "What are the requirements for vMotion?"
- "Why does vMotion fail with 'CPU compatibility' error?"
- "How do you verify vMotion network connectivity?"

---

### 4.5 — Storage vMotion Failure

**Concept:** Migrating VM storage from one datastore to another fails.

**Architecture:** Storage vMotion copies VMDK files (and VM config) from source datastore to destination datastore while VM continues running. Memory delta is copied iteratively, then storage switch happens.

**Components:** Source datastore, destination datastore, VM disk files, VM config file, VMkernel with storage vMotion enabled, shared storage path, storage array.

**Configuration:** Both datastores accessible to host(s) performing vMotion. Storage vMotion enabled on VMkernel. Sufficient space on destination.

**Commands:**
```powershell
# Test Storage vMotion:
Get-VM VMName | Move-VMStorage -Datastore Datastore2 -ValidateOnly
# Execute:
Get-VM VMName | Move-VMStorage -Datastore Datastore2 -RunAsync
# Check both datastores accessible:
Get-Datastore Datastore1, Datastore2 | Select-Object Name, State, Type, FreeSpaceGB, CapacityGB
# Check host access to both datastores:
Get-VMHost VMHost01 | Get-Datastore Datastore1, Datastore2 | Select-Object VMHost, Name, State
# Check VMDK details:
Get-HardDisk -VM VMName | Select-Object DiskName, Filename, CapacityGB, StorageFormat
# Check if VMDK fits on destination:
$vmstorage = Get-VM VMName | Get-HardDisk | Measure-Object -Property CapacityGB -Sum
$destspace = Get-Datastore Datastore2 | Select-Object -ExpandProperty FreeSpaceGB
"$($vmstorage.Sum) GB needed, $destspace GB free"
# Check for swap files on destination:
Get-VM VMName | Get-VMHost | Select-Object Name, @{N='SwapfileDatastore';E={$_.ExtensionData.Config.SwapfileDatastore}}
# Check Storage vMotion VMkernel:
Get-VMHostNetworkAdapter -VMHost VMHost01 | Where-Object {$_.ExtensionData.Config.UplinkPortgroup -ne $null} | Select-Object Name, IPAddress, PortGroupName
# Event logs:
Get-WinEvent -FilterHashtable @{LogName='System'; Id=2135, 2137} -MaxEvents 20  # datastore events
Get-WinEvent -FilterHashtable @{LogName='vmware.log'} -MaxEvents 50  # on ESXi
```

**Real-World Example:** Storage vMotion failed because destination datastore was on a different storage array with different multipathing — resolved by ensuring proper NMP (Native Multipathing Plugin) configuration.

**Common Failures:**
- Destination datastore full
- Destination datastore not accessible by the host performing Storage vMotion
- Insufficient space for VM swap file on destination
- Different storage types with different performance characteristics
- Storage array replication/snapshot running during migration
- VMDK too large (timeout during copy)
- Storage vMotion not enabled on VMkernel
- Faulty storage path

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Storage migration fails. Error message? Which datastores? |
| 2. Determine scope | One VM? Multiple? Same source/destination? |
| 3. Check recent changes | Storage change, datastore expanded/replicated, snapshot running. |
| 4. Check monitoring | Datastore space, storage latency, storage I/O. |
| 5. Validate connectivity | Both datastores accessible from host? `Get-Datastore | Select-Object State, VMHost`. Check multipathing. |
| 6. Check OS | `Get-Datastore` free space. `Get-VMHost` datastore accessibility. `Get-VM VMName | Get-HardDisk` for disk sizes. |
| 7. Check dependencies | Destination datastore accessible? Enough space? VMkernel configured for Storage vMotion? |
| 8. Check logs | vCenter: vpxd.log. ESXi: vmkernel.log, hostd.log. Look for storage I/O errors, timeout errors. |
| 9. Identify root cause | Destination full? Datastore not accessible? Timeout? Swap file location? |
| 10. Implement fix | Free space on destination, fix datastore accessibility, increase timeout, move VM swap to same datastore, retry with less loaded storage, check and fix storage multipathing. |
| 11. Validate service | VM running on new datastore with all VMDKs moved. Performance normal. |
| 12. Monitor | Storage migration success rate, datastore space. |
| 13. Document RCA | Root cause and resolution. |

**Interview Questions:**
- "What are the requirements for Storage vMotion?"
- "How do you handle Storage vMotion for VMs with snapshots?"
- "What if Storage vMotion times out?"

---

### 4.6 — HA Failover

**Concept:** VMware HA triggers a failover when a host or VM fails, restarting VMs on remaining hosts.

**Architecture:** HA monitors host and VM health via heartbeat (VMtools heartbeat, power-on status). On host failure, HA selects a suitable host and restarts VMs based on restart priority and admission control.

**Components:** HA cluster, VM tools heartbeat, VM power state, VM restart priority, host isolation response, admission control, host reserves, datastore with VM recovery markers.

**Configuration:** HA enabled on cluster, VM restart priority (Low/Medium/High/Disabled), Host Monitoring, Admission Control (percentage/hosts/CPU), VM Tools heartbeat selection, isolation response (power off/keep running/SHUTDOWN).

**Commands:**
```powershell
# Check HA cluster settings:
Get-Cluster ClusterName | Select-Object Name, HAEnabled, HADciMode, HAAdmissionControl, HARestartPriority, HAIsolationAddress
# Check HA VM restart priority:
Get-VM VMName | Select-Object Name, HAAdmissionPriority, HARestartPriority, HAMigrationPriority
# Get HA events:
Get-Cluster ClusterName | Get-VIEvent -Types Warning, Error | Where-Object {$_.GetType().Name -like "*HA*"} | Select-Object CreatedTime, FullFormattedMessage | Sort-Object CreatedTime -Descending | Select-Object -First 30
# Check VMs HA status:
Get-VM -Location ClusterName | Select-Object Name, PowerState, @{N='HAEnabled';E={$_.ExtensionData.Config.HAConfigInfo.IsVmRestartSupported}}, @{N='HARestartPriority';E={$_.ExtensionData.Config.HAConfigInfo.RelationalRestartPriority}}
# Check HA agent status on host:
Get-VMHost VMHost01 | Select-Object Name, @{N='HAState';E={$_.ExtensionData.AgentHAIsAlive}}
# If HA not working:
# Check: VM tools running? Heartbeat enabled? Host isolation addresses pingable? Datastore with VM recovery marker accessible?
# Verify:
Get-Cluster ClusterName | Select-Object @{N='HAEnabled';E={$_.ExtensionData.Config.HAConfigInfo.HAEnabled}}, @{N='HADciMode';E={$_.ExtensionData.Config.HAConfigInfo.HADciMode}}
```

**Real-World Example:** HA restarted VMs but they couldn't start due to admission control — set to reserve 50% resources but remaining host only had 30%. Resolved by adjusting admission control to Fixed slots.

**Common Failures:**
- Admission control prevents VM restart (insufficient resources)
- VM tools not installed/not heartbeat enabled (VM not detected as failed)
- Datastore VM recovery marker not created (no shared storage for marker)
- All remaining hosts full
- HA agent not running on host
- Host isolation response inappropriate (Powered off vs. keep running)
- Network isolation (heartbeat network isolated from other hosts)
- Boot storms (too many VMs restarting simultaneously)

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | HA failed to restart VM? VM restarted but crashed? HA didn't detect failure? |
| 2. Determine scope | Cluster-wide or specific VM? After specific host failure? |
| 3. Check recent changes | Host added/removed, HA config change, VM tools upgrade, admission control setting changed. |
| 4. Check monitoring | HA status, cluster resource utilization, VM restart history. |
| 5. Validate connectivity | Hosts can communicate via HA heartbeat network. Datastores accessible by all hosts. |
| 6. Check OS | HA enabled on cluster? VM tools heartbeat selected as HA heartbeat? `Get-Cluster | Select-Object HAEnabled`. Check VM tools status. |
| 7. Check dependencies | VMs have appropriate restart priority? Admission control configured correctly? Datastores accessible? |
| 8. Check logs | HA events in vCenter. `Get-VIEvent` for HA events. vmkernel.log for host heartbeat events. vpxa.log for HA agent communication. |
| 9. Identify root cause | Admission control preventing restart? VM tools not sending heartbeat? Datastore inaccessible? Boot storm? |
| 10. Implement fix | Adjust admission control, ensure VM tools installed and heartbeat enabled, configure correct isolation response, add host capacity, set VM restart priority appropriately, configure multiple HA heartbeat datastores, increase host heartbeat timeout. |
| 11. Validate service | HA successfully restarts VMs on host failure (test by isolating a host or stopping HA agent). |
| 12. Monitor | HA events, cluster resource levels, VM recovery test. |
| 13. Document RCA | Root cause, fix, and HA design review. |

**Interview Questions:**
- "HA doesn't restart VMs — what could be wrong?"
- "What's admission control and why does it matter for HA?"
- "How do you test HA failover?"
- "What's the difference between VM tools heartbeat and power-on heartbeat?"

---

### 4.7 — DRS Issue

**Concept:** DRS (Distributed Resource Scheduler) doesn't balance workloads across hosts or makes incorrect migration recommendations.

**Architecture:** DRS monitors resource utilization (CPU, memory) across hosts in a cluster and makes VM placement/migration recommendations (or automates them with EVC). Uses vMotion under the hood.

**Components:** vCenter (DRS engine), vMotion, cluster settings, VM resource reservations/limits/Shares, host customizations (DRS groups), VM/host groups, DRS rules (VM-VM anti-affinity, VM-host affinity).

**Configuration:** DRS automation level (Manual/Partially Automated/Fully Automated), DRS migration threshold (1-5), VM resource settings, DRS rules, VM-Host groups and rules.

**Commands:**
```powershell
# Check DRS status:
Get-Cluster ClusterName | Select-Object Name, DrsEnabled, DrsAutomationLevel, DrsMigrationThreshold
# Check DRS recommendations:
Get-DrsRecommendation -Cluster ClusterName | Select-Object Id, @{N='Level';E={$_.Level}}, Reason, Name
# Apply recommendation:
Get-DrsRecommendation -Cluster ClusterName | Apply-DrsRecommendation
# Check VM DRS settings:
Get-VM VMName | Select-Object Name, DrsAutomationLevel, DrsMigrationThreshold, @{N='DRSRule';E={$_.ExtensionData.Config.DrsConfigInfo}}
# Check DRS rules:
Get-DrsRule -Cluster ClusterName | Select-Object Name, Id, Type, Enabled, Mandatory, VMGroupName, HostGroupName
# Check cluster resource utilization:
Get-Cluster ClusterName | Select-Object Name, NumHosts, @{N='CpuUsageMhz';E={($_ | Get-ClusterGroup -Node).Usage.Summary.Cpu}}, @{N='MemoryUsageGB';E={($_ | Get-ClusterGroup -Node).Usage.Summary.Memory}}
# Force DRS to rebalance (manual migration):
Get-VM -Location ClusterName | Measure-Object -Property ProvisionedSpaceGB -Sum
# Check for DRS disabled VMs:
Get-VM -Location ClusterName | Where-Object {$_.DrsAutomationLevel -eq 'Disabled'} | Select-Object Name, DrsAutomationLevel
# Enable DRS:
Set-Cluster -Cluster ClusterName -DrsEnabled $true -DrsAutomationLevel FullyAutomated -DrsMigrationThreshold 3
# View DRS history:
Get-DrsHistory -Cluster ClusterName -Start (Get-Date).AddDays(-7) | Select-Object CreatedTime, Type, Description | Sort-Object CreatedTime -Descending | Select-Object -First 50
```

**Real-World Example:** DRS not making recommendations — VMs had custom resource limits preventing migration. After removing limits, DRS balanced properly.

**Common Failures:**
- DRS disabled or in manual mode
- DRS VM-VM anti-affinity rules preventing migration (VMs pinned to specific hosts)
- VM resource limits set too low (DRS can't move because of constraints)
- vMotion fails for specific VMs (so DRS won't attempt)
- Threshold too conservative (threshold 1-2 means very conservative)
- Host customizations missing (DRS doesn't know host relative weighting)
- VM has VM/Host affinity rule pinning it to one host

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | VMs unbalanced? No migration recommendations? Unnecessary migrations? |
| 2. Determine scope | Cluster-wide? Specific VMs? After config change? |
| 3. Check recent changes | DRS settings changed, rules added, resources modified, VMs pinned to hosts. |
| 4. Check monitoring | Cluster CPU/memory utilization per host. DRS recommendations count. |
| 5. Validate connectivity | vMotion working (test separately). Hosts communicating with vCenter. |
| 6. Check OS | `Get-Cluster | Select-Object DrsEnabled, DrsAutomationLevel, DrsMigrationThreshold`. Check for disabled VMs. |
| 7. Check dependencies | vMotion working? Hosts have compatible CPU (EVC)? Shared storage? |
| 8. Check logs | DRS logs in vCenter (Tasks/Events). DRS recommendations. Check for migration failures. |
| 9. Identify root cause | DRS disabled? Migration threshold too low? Rules preventing? vMotion failing? |
| 10. Implement fix | Enable DRS, adjust automation level, increase threshold (3-5), remove/adjust rules that prevent balancing, remove VM resource limits, fix vMotion issues. |
| 11. Validate service | DRS makes and applies recommendations. Cluster balanced over time. |
| 12. Monitor | DRS recommendations, cluster balance. |
| 13. Document RCA | Root cause, fix, and DRS design review. |

**Interview Questions:**
- "DRS isn't balancing — what could cause this?"
- "What's the difference between DRS automation levels?"
- "How do DRS rules affect VM placement?"
- "How do you safely disable DRS for a VM without losing it?"

---

### 4.8 — Snapshot Consolidation Failure

**Concept:** Snapshot delta disks fail to consolidate into parent VMDK, leaving snapshots growing or blocking operations.

**Architecture:** Snapshot creates delta disk (`.vmdk` delta) and `.vmsn` memory snapshot. Consolidation writes delta disk changes into parent VMDK, removing the snapshot.

**Components:** VMDK parent disk, delta disks (.vmdk delta), snapshot descriptor, .vmsn memory state, VM config (.vmx), VM power state, host.

**Configuration:** Snapshot retention policy, snapshot max depth (best practice ≤ 2-3), snapshot quiesce (freezes guest I/O during snapshot creation).

**Commands:**
```powershell
# Check snapshots on VM:
Get-VM VMName | Get-Snapshot | Select-Object Name, Id, Created, SizeMB, Description, VM
# Check snapshot hierarchy:
Get-VM VMName | Get-Snapshot | Format-Tree
# Try consolidation via vCenter:
Get-VM VMName | Get-Snapshot | Remove-Snapshot -Consolidate -Confirm:$false
# Or via ESXi SSH:
vim-cmd vmsvc/snapshot.getall <vmid>  # list snapshots
vim-cmd vmsvc/snapshot.removeall <vmid>  # remove all snapshots (NOT consolidation!)
# For consolidation on ESXi:
# Right-click VM in vSphere Client → Snapshot → Consolidate
# Check if VM is snapshot-locked:
# Find .lck files:
find /vmfs/volumes/ -name "*.lck" -type d | head -20
# Remove stale lock (ONLY if VM powered off and truly no process using it):
# First verify VM is powered off and no host has it registered
# Check for running consolidation tasks:
# vCenter: Monitor → Tasks → Recent Tasks → filter for "Consolidate"
# Check if consolidation is already in progress:
Get-VM VMName | Get-Snapshot | Select-Object Name, Id, @{N='State';E={$_.State}}
# Check disk chain:
# .vmdk files in VM directory:
ls /vmfs/volumes/Datastore1/VMName/*.vmdk
# Check VM config for snapshot entries:
cat /vmfs/volumes/Datastore1/VMName/VMName.vmx | grep snapshot
# Force snapshot manager:
# Via vCenter API:
# PowerCLI:
Get-VM VMName | Get-View | $_.ConsolidateAllSnapshotDisks_Task()
```

**Real-World Example:** Snapshot consolidation stuck for 6+ hours — VM had large delta disk (100GB+). Consolidation was I/O intensive and timed out. Resolved by consolidating during maintenance window with increased timeout.

**Common Failures:**
- VM powered off during consolidation (needs to be ON for full consolidation)
- Snapshot delta disk is very large (>100GB takes long)
- I/O intensive during consolidation (time out)
- Consolidation task stuck/hung (common after vCenter crash during consolidation)
- Power off VM and consolidation simultaneously (lock issue)
- .vmdk corruption in chain
- Disk chain has too many snapshots (performance degrades with depth)
- Stale lock files preventing consolidation
- vCenter crash during consolidation

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Consolidation fails/stuck? VM performance degraded? Error in vCenter? |
| 2. Determine scope | One VM? Multiple? After specific change? |
| 3. Check recent changes | Snapshot creation, vCenter crash, maintenance activity. |
| 4. Check monitoring | Snapshot age/size monitoring. Alert if snapshot older than 24 hours. |
| 5. Validate connectivity | VM running? Datastore accessible? vCenter accessible? |
| 6. Check OS | `Get-VM VMName | Get-Snapshot` for snapshot state. Check snapshot size and age. `Get-VM VMName | Get-View | $_.ConsolidateAllSnapshotDisks_Task()` for consolidation state. |
| 7. Check dependencies | VM running? vCenter operational? Datastore has space for consolidation? |
| 8. Check logs | vCenter tasks (consolidation). Event IDs for snapshot/consolidation failures. `vmkernel.log` on ESXi. `vpxd.log`. |
| 9. Identify root cause | Consolidation task hung? Delta disk too large? VM crashed? vCenter crash? |
| 10. Implement fix | Restart vCenter management agents (if task hung), re-trigger consolidation, power on VM if it was off, delete unnecessary snapshots first (if delta is huge), wait for consolidation to complete (can take hours for large deltas), in extreme case: clone VM from snapshot, delete old VM, re-register clone (last resort). |
| 11. Validate service | No delta disks left. Snapshot tree empty (or desired state). Consolidation task completed. VM running normally. |
| 12. Monitor | Snapshot size and age monitoring. No consolidation tasks pending. |
| 13. Document RCA | Root cause, fix, and snapshot management policy update (snapshot age ≤ 24 hours, max depth ≤ 3, no production snapshots). |

**Interview Questions:**
- "What are the risks of having many snapshots?"
- "How do you troubleshoot a stuck consolidation?"
- "What happens if a snapshot is deleted without consolidation?"
- "What's snapshot quiescing and what can go wrong?"

---

### 4.9 — High CPU Ready

**Concept:** VM CPU Ready time is high, meaning the VM is ready to run but waiting for physical CPU scheduling on the ESXi host.

**Architecture:** ESXi scheduler allocates physical CPU to VMs. CPU Ready % = time VM is ready but not running. High Ready% = CPU contention.

**Components:** ESXi CPU scheduler, VM CPU allocation (1+ vCPUs), CPU reservations, CPU shares, host CPU overcommit ratio, other VMs on host, ESXi itself.

**Configuration:** CPU reservation (0 = no guarantee), CPU shares (Low/Normal/High/Custom), CPU limit (unlimited by default), number of vCPUs, host CPU overcommit ratio.

**Commands:**
```powershell
# Performance monitoring in vCenter: Performance tab for VM.
# PowerCLI:
Get-VM VMName | Select-Object Name, NumCpu, Cpu, @{N='CPUUsageMHz';E={($_ | Get-Stat -Stat cpu.usage.average -MaxSamples 1 -Start (Get-Date).AddMinutes(-10) | Measure-Object -Property Value -Average).Average}}
# Check CPU Ready:
Get-VM VMName | Get-Stat -Stat cpu.ready.summation -Start (Get-Date).AddHours(-1) -IntervalMins 5 | Select-Object Timestamp, Value
# Get average CPU Ready % for VM:
# In vCenter: Performance Chart → CPU → Ready%
# Check host CPU capacity:
Get-VMHost VMHost01 | Select-Object Name, @{N='CPUUsedMHz';E={($_.ExtensionData.Summary.QuickStats.OverallCpuUsage)}}, @{N='CPUTotalMHz';E={($_.ExtensionData.Hardware.CpuInfo.NumCpuCores * $_.ExtensionData.Hardware.CpuInfo.ClockSpeed / 1000