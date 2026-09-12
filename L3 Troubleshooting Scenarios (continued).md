# CONTINUED — PART 35 & 36: L3 Troubleshooting Scenarios (continued)

---

## 🔷 DOMAIN 4: VMWARE vSPHERE (continued)

---

### 4.9 — High CPU Ready (continued)

| Step | Action |
|------|--------|
| 10. Implement fix | Reduce vCPUs (vCPU to pCPU ratio: 1:1 preferred, don't overprovision), increase CPU reservation for critical VMs, add physical CPU/cores to host, move VMs to less loaded host (DRS), disable CPU hot-add if not needed (reduces scheduler overhead), check for zombie/defunct VMs consuming CPU. |
| 11. Validate service | CPU Ready % below 5-8% (sustainable threshold). VM performance within SLA. |
| 12. Monitor | CPU Ready % for all VMs on host. Set alert >20% sustained. Use vCenter dashboard. |
| 13. Document RCA | Over-provisioned vCPUs or host resource exhaustion. Right-size VMs. |

**Interview Questions:**
- "What's a healthy CPU Ready percentage?"
- "How does CPU overcommit affect performance?"
- "What's the relationship between CPU Ready and CPU Co-Stopped?"

---

### 4.10 — High Storage Latency

**Concept:** VM disk I/O experiencing high latency, causing application slowdowns, timeouts, and poor storage performance.

**Architecture:** VM I/O path: Guest OS → virtual disk (VMDK) → SCSI controller → VMkernel → HBA/NIC → storage array (SAN/NAS) or local disk. Each hop adds latency.

**Components:** VMDK, VMKernel, HBA, storage controller, cache (array/SSD), RAID, disk pods, NVMe/Flash storage, multipathing.

**Configuration:** Storage I/O Control (SIOC), disk shares, disk reservations, multipathing policy, cache settings, queue depth.

**Commands:**
```powershell
# Check VM disk latency in vCenter: Performance tab → Disk → Latency (ms)
# PowerCLI — get latency stats:
Get-VM VMName | Get-Stat -Stat disk.connectedLatency.average -Start (Get-Date).AddHours(-1) -IntervalMins 5 | Select-Object Timestamp, Value
Get-VM VMName | Get-Stat -Stat disk.totalLatency.average -Start (Get-Date).AddHours(-1) -IntervalMins 5 | Select-Object Timestamp, Value
Get-VM VMName | Get-Stat -Stat disk.readLatency.average -Start (Get-Date).AddHours(-1) -IntervalMins 5 | Select-Object Timestamp, Value
Get-VM VMName | Get-Stat -Stat disk.writeLatency.average -Start (Get-Date).AddHours(-1) -IntervalMins 5 | Select-Object Timestamp, Value
# Check host storage latency:
Get-VMHost VMHost01 | Get-Stat -Stat storageLatency.total -Start (Get-Date).AddHours(-1) | Select-Object Timestamp, Value
# Check SIOC:
Get-Datastore Datastore1 | Select-Object Name, @{N='SIOCEnabled';E={$_.ExtensionData.StorageIOControlConfig.Enable}}, IopsThreshold, CongestionThreshold
# Check multipathing:
Get-VMHost VMHost01 | Get-VMHostMultipathInfo | Select-Object VMHost, @{N={'Paths';E={$_.MultipathInfo.Paths.Count}}}
# On ESXi SSH:
esxcli storage core device list  # shows all paths, latency, state
esxcli storage core path list  # shows all paths
# Check HBA/adapter stats:
esxcli storage core adapter list
# Check datastore IOPS:
esxcli storage stats volume list -m vmfs6  # or vmfs5
# Check VM kernel disk latency:
# vmkernel.log entries with "scsi" or "I/O" errors
```

**Real-World Example:** All VMs on a datastore had 50ms+ latency — array was in a degraded RAID state (single disk failure, degraded array). Replaced disk, array rebuilt, latency returned to <5ms.

**Common Failures:**
- Storage array degraded (RAID failure, rebuild in progress)
- Network congestion (iSCSI/ NFS storage network saturated)
- HBA card issue (firmware, driver, path failure)
- Too many VMs on one datastore (contention)
- SIOC not configured or misconfigured
- Multipathing misconfigured (asymmetric paths)
- Cache battery low (array write cache disabled → all writes go to disk)
- Old/slow disks (HDD instead of SSD)
- VM has excessive IOPS (noisy neighbor)
- Snapshot delta disk generating heavy I/O

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | High disk latency on VMs? All VMs or specific? Specific datastore? |
| 2. Determine scope | One VM? All on one datastore? All datastores? One host or cluster? |
| 3. Check recent changes | New workload, storage maintenance, array firmware update, HBA change, network change. |
| 4. Check monitoring | vCenter: VM disk latency (ms) — normal <10ms, concerning >20ms, critical >40ms. Host storage latency. Array monitoring (controller health, RAID status, cache battery). |
| 5. Validate connectivity | Storage network connectivity (iSCSI: port 3260; FC: FLOGI; NFS: port 2049). Multipathing active. |
| 6. Check OS | On ESXi: `esxcli storage core device list` (check latency, paths, state). `esxcli storage stats volume list`. On VM: disk latency in Performance tab. |
| 7. Check dependencies | Storage array healthy? RAID not degraded? Cache battery OK? Network to storage not saturated? |
| 8. Check logs | ESXi: `/var/log/vmkernel.log` for I/O errors, timeouts, path failures. vCenter: vpxd.log. Array controller logs. Event IDs for path failures. |
| 9. Identify root cause | Array degraded? Network saturated? Noisy VM? HBA issue? Cache disabled? |
| 10. Implement fix | Replace failed disk/repair RAID, move noisy VM to different datastore, configure SIOC (set IOPS threshold), fix multipathing, upgrade HBA firmware, enable write cache (with battery), add HBA ports, migrate to faster storage tier, adjust VM disk shares, reduce VM snapshots (I/O overhead). |
| 11. Validate service | Latency <10ms sustained. VM application performance restored. |
| 12. Monitor | Array health, latency metrics, path states. |
| 13. Document RCA | Root cause, resolution, and storage health check schedule. |

**Interview Questions:**
- "High storage latency — what's your approach?"
- "What's Storage I/O Control and how does it help?"
- "How do you differentiate between VM-level and datastore-level latency issues?"

---

### 4.11 — VM Network Failure

**Concept:** A VM loses network connectivity — cannot ping, cannot communicate, or has no network at all.

**Architecture:** VM virtual NIC (vNIC) → virtual switch (vSwitch) → physical NIC (pNIC) → physical network → destination. VMs connect to port groups on vSwitches.

**Components:** vNIC (VMXNET3 recommended), vSwitch, port group, VLAN ID, physical NIC (pNIC), NIC teaming, uplink, driver, VM tools, firewall, security groups (NSX).

**Configuration:** vSwitch port group, VLAN tagging (external/transparent/ virtual), NIC teaming policy, security policies (promiscuous mode, forged transmits, MAC changes), VM network adapter settings.

**Commands:**
```powershell
# Check VM network:
Get-VM VMName | Get-NetworkAdapter | Select-Object Name, NetworkName, StartConnected, Type, MacAddress
# Check vSwitch:
Get-VirtualSwitch -VMHost VMHost01 | Select-Object Name, NumPorts, Mtu, NicCount
Get-VirtualSwitch -VMHost VMHost01 | Get-VirtualPortGroup | Select-Object Name, VLANId
# Check physical NICs:
Get-VMHostNetworkAdapter -VMHost VMHost01 | Select-Object Name, IPAddress, Status, Speed
Get-VMHost VMHost01 | Get-VMHostNetworkAdapter | Where-Object {$_.Type -eq '物理NIC'}  # Get-PhysicalNic
# Check NIC teaming:
Get-NicTeamingPolicy -VMHost VMHost01 -VirtualSwitchName vSwitch0 | Select-Object *
# Check if VM can network:
Test-Connection -ComputerName 8.8.8.8 -Source (Get-VM IP)  # from VM
# Check connectivity from host to VM:
ping VM_IP
# PowerCLI connectivity test:
Test-VMNetworkConnectivity -VM VMName -DestinationIP 8.8.8.8  # if module available
# Check VM tools (needed for some network features):
Get-VM VMName | Select-Object Name, ExtensionData.Guest.ToolsRunningStatus, ExtensionData.Guest.ToolsVersionStatus
# Check network adapter on host:
# ESXi SSH:
esxcfg-vswitch -l  # list vSwitches and ports
esxcfg-vlan -l  # list VLANs
esxcli network nic list  # list physical NICs
esxcli network ip interface list  # list VMkernel interfaces
# Check VM port on vSwitch:
# vCenter → VM → Edit Settings → Network Adapter → Port ID → check on vSwitch port group
# Check for network alerts:
Get-Cluster ClusterName | Get-VIEvent -Types Warning | Where-Object {$_.FullFormattedMessage -like "*network*" -or $_.FullFormattedMessage -like "*NIC*"} | Select-Object CreatedTime, FullFormattedMessage -First 30
```

**Real-World Example:** VM lost network after host reboot — vSwitch uplinks not active because NIC drivers hadn't loaded yet. Resolved by setting proper boot order (NIC before other services).

**Common Failures:**
- vNIC disconnected (StartConnected = $false)
- VM tools not running (some network features require tools)
- Port group/VLAN mismatch
- Physical NIC down/failed
- vSwitch uplink misconfigured
- NIC teaming all ports failed
- VLAN tagging wrong (trunk vs access mismatch)
- Firewall/NSX security rules blocking
- IP conflict
- DHCP issue (no IP → no network)
- VM network adapter type incompatible (E1000 vs VMXNET3)

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | VM has no network? Slow network? Specific VM or all VMs on host? Intermittent? |
| 2. Determine scope | One VM? All on same vSwitch? All on host? Specific VLAN? |
| 3. Check recent changes | Host reboot, vSwitch config change, NIC failure, VLAN change, security policy change. |
| 4. Check monitoring | vCenter network alerts, pNIC status, vSwitch port count, VM network utilization. |
| 5. Validate connectivity | VM can ping host gateway? `ping` from VM to gateway, external IP, DNS. `Test-NetConnection` from host to VM IP on port 445/80/3389. |
| 6. Check OS | VM: `ipconfig`/`ifconfig`. Check vNIC connected status. `Get-VMNetworkAdapter`. vCenter: vSwitch port group, VLAN. ESXi: `esxcfg-vswitch -l`, `esxcli network nic list`. |
| 7. Check dependencies | vSwitch has uplink? pNIC link up? Port group correct? VM tools running? |
| 8. Check logs | vCenter: Events (network-related). ESXi: `vmkernel.log` (link up/down events, NIC errors). `/var/log/syslog.log`. |
| 9. Identify root cause | vNIC disconnected? Physical NIC down? Port group wrong? VLAN mismatch? Security policy blocking? |
| 10. Implement fix | Reconnect vNIC (`Set-NetworkAdapter -StartConnected $true`), fix port group, fix NIC teaming/replace failed NIC, correct VLAN config, check/replace NIC driver, enable required security flags (promiscuous/forged/MAC for certain use cases like NSX/monitoring), restart VM, reinstall VM tools. |
| 11. Validate service | VM has network connectivity (ping, application access). Bandwidth normal. |
| 12. Monitor | Network utilization, packet loss, connectivity monitoring. |
| 13. Document RCA | Root cause, fix, and preventive action. |

**Interview Questions:**
- "A VM has no network — what's your checklist?"
- "What security policy settings are required for promiscuous mode (for tools like Wireshark/NSX)?"
- "How do you troubleshoot VM network issues when you can't RDP into the VM?"
- "What NIC types are recommended and why?"

---

## 🔷 DOMAIN 5: HYPER-V (⭐⭐⭐⭐)

---

### 5.1 — VM Won't Start

**Concept:** A Hyper-V VM fails to start, showing an error state or stopping immediately.

**Architecture:** Hyper-V VM starts: VM worker process (vmwp.exe) launches, reads VM configuration (XML/VMCX), initializes virtual devices, boots from virtual firmware (UEFI/BIOS), loads OS from virtual disk (VHD/VHDX).

**Components:** VM configuration (XML/VMCX), virtual hard disk (VHD/VHDX), virtual switch, VM worker process (vmwp.exe), Hyper-V Virtual Machine Management Service (VMMS), Hyper-V host.

**Configuration:** VM startup memory, dynamic memory settings, processor count, boot order, generation (1 or 2), virtual hardware version, integration services.

**Commands:**
```powershell
# Check VM state:
Get-VM -Name VMName | Select-Object Name, State, Id, Generation, Path, HasDynamicMemory, Status
# Check VM health:
Get-VM -Name VMName | Get-VMHealth -ErrorAction SilentlyContinue
# Check VM integration services:
Get-VMIntegrationService -VMName VMName | Select-Object VMName, Name, Enabled, PrimaryStatusDescription, HealthStatus
# Try to start VM:
Start-VM -Name VMName
# Check VM configuration:
Get-VM -Name VMName | Select-Object * | Format-List
# Check VM log file:
Get-ChildItem "C:\ProgramData\Microsoft\Windows\Hyper-V\Virtual Machines\$VMID" -ErrorAction SilentlyContinue
# VM configuration path:
Get-VM -Name VMName | Select-Object -ExpandProperty Path
# Check VMMS service:
Get-Service VMMS
# Check Hyper-V:
Get-WindowsFeature Hyper-V
Get-WmiObject Win32_ComputerSystem | Select-Object -ExpandProperty HypervisorPresent
# Check VM boot failure details:
# Check Event Viewer: Applications and Services Logs → Microsoft → Windows → Hyper-V-VMMS → Administrative
Get-WinEvent -LogName "Microsoft-Windows-Hyper-V-VMMS-Admin" -MaxEvents 30
# Also: Hyper-V-Worker
Get-WinEvent -LogName "Microsoft-Windows-Hyper-V-Worker-Admin" -MaxEvents 30
# Check if VM is saved/ paused:
Get-VM | Where-Object {$_.State -in @('Saved','Paused','Frozen')}
# Check VM configuration file directly:
# For Gen2 VMs: .vmcx file (XML-based)
# For Gen1: .xml file
# Can sometimes edit to fix configuration issues
```

**Real-World Example:** VM won't start after host update — integration services version mismatch. Updated integration services from Hyper-V Manager → VM settings → Insert Integration Services Setup Disk.

**Common Failures:**
- VM configuration corrupted (.vmcx/.xml)
- VHD/VHDX file missing or corrupt
- VM in Saved/Paused/Failed state
- Insufficient host resources (memory, CPU)
- Integration services outdated
- Generation mismatch (trying to boot Gen2 from VHD instead of VHDX)
- VM hardware version incompatible with host
- VMMS service stopped
- Virtual switch misconfigured
- Boot device not found (wrong boot order, no bootable disk)

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Error message? Fails immediately or hangs? Specific VM? |
| 2. Determine scope | One VM? All VMs on host? After specific change? |
| 3. Check recent changes | Host update, VM configuration change, VHD moved, Hyper-V role updated. |
| 4. Check monitoring | VM state in Hyper-V Manager. Host resources. Hyper-V health. |
| 5. Validate connectivity | If VM needs network, check virtual switch. |
| 6. Check OS | `Get-VM` state. `Get-VMIntegrationService` status. `Get-Service VMMS` running. VMMS event logs. |
| 7. Check dependencies | VHD/VHDX file exists? Virtual switch exists? Host resources sufficient? VM hardware version compatible? |
| 8. Check logs | Hyper-V-VMMS-Admin log, Hyper-V-Worker-Admin log. Event IDs for VM start failures. `vmwp.exe` process status. |
| 9. Identify root cause | Config corrupt? VHD missing? Integration services outdated? Service stopped? |
| 10. Implement fix | Repair VM config (re-import, edit XML), replace/repair VHD, restart VMMS service (`Restart-Service VMMS`), update integration services, change VM hardware version, free host resources, change boot order, remove saved state, set VM state to Running. |
| 11. Validate service | VM starts successfully, OS boots, network/IO functional. |
| 12. Monitor | VM uptime, resource usage. |
| 13. Document RCA | Root cause, fix, preventive action. |

**Interview Questions:**
- "A Hyper-V VM won't start — walk me through your approach."
- "What's the difference between Generation 1 and Generation 2 VMs?"
- "How do you check Hyper-V VM logs?"

---

### 5.2 — Live Migration Failure

**Concept:** Live migration of a running VM between Hyper-V hosts fails.

**Architecture:** Live migration transfers VM memory (RAM pages) over network between hosts. Uses SMB 3.0 (recommended), TCP, or WSD. VM stays running during migration (brief pause during memory transfer).

**Components:** Source and destination Hyper-V hosts, VM memory, VM state, network between hosts (high bandwidth, low latency), SMB 3.0 file share (for SMB-based migration), VM configuration, virtual hard disks.

**Configuration:** Live migration enabled, authentication (Kerberos/cleartext), migration network (SMB or TCP), compression enabled, VLAN for migration network, SMB share with proper permissions.

**Commands:**
```powershell
# Check if Live Migration enabled:
Get-VMHost -ComputerName Host01 | Select-Object Name, @{N='LiveMigrationEnabled';E={$_.HostSupport.LiveMigrationEnabled}}
# Enable Live Migration:
Enable-VMMigration -ComputerName Host01
Set-VMHost -ComputerName Host01 -LiveMigrationEnabled $true
# Configure migration network:
Set-VMHost -ComputerName Host01 -VirtualMachineMigrationNetworkAdapter "Ethernet 2"
# Check migration settings:
Get-VMHostMigration -ComputerName Host01
# Test migration (validate):
Move-VM -Name VMName -DestinationHost Host02 -IncludeStorage -PlanOnly  # simulate
# Perform migration:
Move-VM -Name VMName -DestinationHost Host02 -IncludeStorage
Move-VM -Name VMName -DestinationHost Host02 -IncludeStorage -Force
# For SMB migration (recommended):
Move-VM -Name VMName -DestinationHost Host02 -StorageMoveType SMB
# Check SMB share:
Get-SmbShare | Where-Object {$_.Name -like "*migration*" -or $_.Name -like "*vm*"}
# Check SMB permissions:
Get-SmbShareAccess -Name "VMMigrationShare"
# Check network between hosts:
Test-NetConnection -ComputerName Host02 -Port 445  # SMB
Test-NetConnection -ComputerName Host02 -Port 6600  # WSD migration
# Check VM health before migration:
Get-VM VMName | Select-Object Name, State, Health, IntegrationServicesStatus
# Check Hyper-V event logs:
Get-WinEvent -LogName "Microsoft-Windows-Hyper-V-VMMS-Admin" -MaxEvents 30
Get-WinEvent -LogName "Microsoft-Windows-Hyper-V-Worker-Admin" -MaxEvents 30
# Check cluster (if cluster):
Get-ClusterNode | Select-Object Name, State, DynamicWeight
Get-ClusterGroup | Where-Object {$_.GroupType -eq 'VirtualMachine'} | Select-Object Name, State, OwnerNode
```

**Real-World Example:** Live migration fails with "migration target unreachable" — firewall blocking port 6600 (WSD) or SMB 445. Enabled SMB 3.0 migration with Kerberos auth, opened SMB port — resolved.

**Common Failures:**
- Network issue between hosts (firewall, VLAN, bandwidth)
- SMB share not configured or permissions wrong
- SMB 3.0 not enabled or encryption not matching
- Authentication mismatch (Kerberos not configured)
- VM has too much memory (migration timeout)
- VM checkpoint in progress
- VM uses iSCSI-attached storage (migration issues)
- Destination host insufficient resources
- Version mismatch between hosts (different Hyper-V versions)
- VM hardware version incompatible with destination
- Compression not configured for slow links

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Migration fails immediately or times out? Error message? Which hosts? |
| 2. Determine scope | One VM? All VMs between these hosts? Specific migration type (SMB/TCP)? |
| 3. Check recent changes | Network change, host update, SMB share change, VM configuration change. |
| 4. Check monitoring | Network throughput between hosts, host resources, VM health. |
| 5. Validate connectivity | `Test-NetConnection Host02 -Port 445` (SMB) or port 6600 (WSD). Network latency and bandwidth between hosts. SMB 3.0 encryption capability. |
| 6. Check OS | `Get-VMHost | Select-Object LiveMigrationEnabled, VirtualMachineMigrationNetworkAdapter`. Check SMB share exists and is accessible. |
| 7. Check dependencies | SMB share accessible? Correct permissions? Network bandwidth sufficient? Authentication configured? |
| 8. Check logs | Hyper-V-VMMS-Admin and Hyper-V-Worker-Admin event logs. SMB server logs. `Move-VM` error output. |
| 9. Identify root cause | Firewall blocking? SMB permissions? Timeout due to large memory? Network latency? |
| 10. Implement fix | Open firewall ports (445 for SMB, 6600 for WSD), configure SMB 3.0 migration share with proper permissions, enable Kerberos authentication, enable migration compression for slow links, increase migration timeout, ensure VM hardware version compatibility, check/fix iSCSI storage access, upgrade integration services on VM. |
| 11. Validate service | VM migrates successfully between hosts. VM functional after migration (network, storage, applications). |
| 12. Monitor | Migration success rate, migration time, network utilization during migration. |
| 13. Document RCA | Root cause, fix, and migration best practices documentation. |

**Interview Questions:**
- "What are the requirements for Hyper-V Live Migration?"
- "SMB vs. TCP migration — when do you use each?"
- "What ports does Live Migration use?"
- "How do you troubleshoot migration timeout?"

---

### 5.3 — Cluster Node Failure

**Concept:** A node in a Hyper-V failover cluster goes down, and the cluster must handle VM failover.

**Architecture:** Failover cluster: nodes communicate via heartbeat (UDP), cluster database (Clusdb), and witness (cloud/file/witness disk). VMs are clustered resources with preferred owners and failover policies.

**Components:** Cluster nodes, cluster network, witness (cloud/file/disk), cluster shared volumes (CSV), VM Cluster roles, failover policies, Quorum, node weights.

**Configuration:** Cluster quorum configuration, failover thresholds, VM restart policies (failover, restart on same node), preferred owners, node voting weights, cluster network names, CSV configuration.

**Commands:**
```powershell
# Check cluster health:
Get-Cluster | Select-Object Name, State, ClustersVersion
Get-ClusterNode | Select-Object Name, State, NodeWeight, NodeFailing
Test-Cluster -Node Node01, Node02  # comprehensive validation
# Check cluster network:
Get-ClusterNetwork | Select-Object Name, State, Address, IPv4Subnet
# Check cluster quorum:
Get-ClusterQuorum | Select-Object Cluster, QuorumSource, QuorumType, VoteCount
# Check cluster resources:
Get-ClusterResource | Select-Object Name, State, OwnerNode, ResourceType
# Check VM cluster roles:
Get-ClusterGroup -Cluster ClusterName | Where-Object {$_.GroupType -eq 'VirtualMachine'} | Select-Object Name, State, OwnerNode, PreferredOwner, FailoverLevel
# Check node status:
Get-ClusterNode -Cluster ClusterName | Select-Object Name, State, LastStateChanged, FailoverCount
# If node failed:
# Check if cluster can see node:
(Get-ClusterNode Node01).State  # Up, Down, Paused
# Force node removal (if permanently down):
Remove-ClusterNode -Name Node01 -ForceRemoval
# Drain node (graceful VM migration):
Stop-ClusterNode -Name Node01 -Drain
# Check cluster logs:
Get-ClusterLog -Node Node01 -TimeSpan 24  # cluster log from node
# Or: C:\Windows\Cluster\Reports\*.html for cluster reports
# Check CSV:
Get-ClusterSharedVolume | Select-Object Name, State, OwnedNode
# Check witness:
Get-ClusterQuorumWitness | Select-Object Name, Type, State
```

**Real-World Example:** Node crashed due to BSOD, cluster lost quorum because witness was also offline — reconfigured cloud witness, established quorum.

**Common Failures:**
- Node hardware failure (BSOD, power loss, memory corruption)
- Cluster network partition (split-brain)
- Quorum lost (odd number of votes becomes even)
- CSV unavailable on failing node
- Witness offline/inaccessible
- VM fails to fail over (resource dependency issue)
- Node not gracefully drained (VMs in saved state)
- Network heartbeat loss (not actual node failure — network issue)

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Cluster node down? Cluster lost quorum? VMs not failing over? |
| 2. Determine scope | Single node? Multiple nodes? All VMs affected? |
| 3. Check recent changes | Node hardware issue, network change, cluster config change, witness change. |
| 4. Check monitoring | Cluster health dashboard, node status, quorum status, CSV status. |
| 5. Validate connectivity | All cluster nodes can reach each other on cluster network. Witness accessible. `ping`, `Test-NetConnection` on cluster network ports. |
| 6. Check OS | `Get-ClusterNode` for node state. `Get-Cluster` health. `Test-Cluster` validation report. |
| 7. Check dependencies | Quorum available? CSV accessible? Witness online? Cluster network healthy? |
| 8. Check logs | Cluster logs (`C:\Windows\Cluster\Reports\` and `Get-ClusterLog`). Event Viewer: Failover Clustering events (IDs 1000-1300 range). VM logs. |
| 9. Identify root cause | Node hardware failed? Quorum lost? CSV path issue? Network partition? |
| 10. Implement fix | Reboot/replace failed node, reconfigure quorum witness, drain failed node (`Stop-ClusterNode -Drain`), re-add node to cluster, fix CSV, ensure proper node voting weights, fix network connectivity between nodes. |
| 11. Validate service | All VMs running on remaining nodes. Cluster has quorum. `Test-Cluster` passes. New VMs can fail over. |
| 12. Monitor | Cluster health, node status, quorum status, CSV health. |
| 13. Document RCA | Root cause, fix, and cluster resilience review. |

**Interview Questions:**
- "What happens when a cluster node fails?"
- "How does cluster quorum work and what happens when you lose it?"
- "What's the difference between Drain and ForceRemove?"
- "How do you configure a cloud witness?"

---

### 5.4 — CSV Issue (Cluster Shared Volume)

**Concept:** Cluster Shared Volumes (CSV) become unavailable, preventing VMs from accessing storage or failing over.

**Architecture:** CSV is a shared NTFS volume accessible by all cluster nodes simultaneously, managed by CSV Volume Manager (CSVVM). Uses block-level redirect (BDR) orSMB-based redirect (SBR) for non-owner node access.

**Components:** CSV volume, CSV metadata file (`*` files), CSV VM worker, CSV file system driver (csvfs.sys), NTFS, LDM, storage multipathing.

**Configuration:** CSV block size (64KB default), CSV direct I/O, CSV metadata logging, SMB redirect (for non-owner).

**Commands:**
```powershell
# Check CSV status:
Get-ClusterSharedVolume | Select-Object Name, State, OwnedNode, PhysicalPaths, @{N='Status';E={$_.State}}
# Get CSV info:
Get-ClusterSharedVolume -Name "CSV1" | Select-Object *
# Check CSV health:
Get-ClusterResource -Name "CSV1" | Get-ClusterParameter
# Force CSV online:
# From failing node or any node:
Get-ClusterSharedVolume "CSV1" | Resume-ClusterSharedVolume  # if paused
# Stop-ClusterGroup "CSV1" then Start-ClusterGroup "CSV1"  # restart CSV resource
# Check CSV blocking:
# Sometimes CSV gets blocked by a process holding a handle:
# On all nodes, check for open handles on CSV:
handle.exe -a C:\  # Sysinternals tool
# Check CSV events:
Get-WinEvent -LogName "Microsoft-Windows-FailoverClustering/Operational" | Where-Object {$_.Id -eq 5120 -or $_.Id -eq 5125 -or $_.Id -eq 5139} -MaxEvents 30
# Check CSV LUN mapping:
Get-ClusterSharedVolume | Select-Object -ExpandProperty PhysicalPaths
# Rescan CSV:
Get-Disk | Where-Object {$_.FriendlyName -like "*CSV*"} | Rescan
# Check CSV metadata:
# CSV metadata files are in root of CSV (e.g., C:\ClusterStorage\Volume1\...)
dir C:\ClusterStorage\Volume1\*  # check for metadata files
# If CSV offline on all nodes (critical):
# Check: Is the underlying LUN accessible from all nodes?
# Check multipath:
Get-Multipath | Select-Object *
# If metadata corrupt:
# Sometimes need to force CSV online from a node that can see the LUN:
# From Hyper-V Manager or Failover Cluster Manager → Right-click CSV → Bring Online
# If that fails, manually from node that can access LUN:
mountvol C:\ClusterStorage\Volume1 \\?\GLOBALROOT\Device\CdRom0...  # manual mount (advanced)
```

**Real-World Example:** CSV offline after host crash — metadata file locked by a VM on another node. Used `Resume-ClusterSharedVolume` from a healthy node with proper LUN mapping.

**Common Failures:**
- CSV offline on all nodes (LUN path lost)
- CSV metadata corruption
- CSV blocked by process on non-owner node (handle leak)
- LUN mapping not visible on one node (storage MPIO issue)
- Node in CSV dump mode (CSVVM worker hung)
- Antivirus locking CSV files
- CSV snapshot/SCVMM issue
- CSV out of space

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | VMs can't access storage? CSV offline? One node or all? |
| 2. Determine scope | One CSV? All CSVs? One node can see it or none? |
| 3. Check recent changes | Storage change, host crash, node restart, AV policy change. |
| 4. Check monitoring | CSV state in Failover Cluster Manager. LUN health from storage array. |
| 5. Validate connectivity | All nodes can access LUN? Check multipathing. `Get-Disk | Where-Object FriendlyName -like "*CSV*"` on each node. |
| 6. Check OS | `Get-ClusterSharedVolume` status. CSV state (Online/Offline/Blocked/Lost). On each node: Can you see the LUN? Can you mount it? |
| 7. Check dependencies | LUN accessible? MPIO/NMP working? All nodes have same storage path? |
| 8. Check logs | Failover Clustering/Operational log (Event IDs 5120, 5125, 5139, 5146). CSV VM worker logs. `vmwp.exe` process for CSV worker. |
| 9. Identify root cause | LUN path failure? Metadata corruption? CSVVM worker hung? Antivirus locking files? |
| 10. Implement fix | Resume CSV, restart CSV resource group, fix multipathing, restart VMMS/CSVVM process, exclude CSV from AV scanning, fix LUN mapping, replace failed storage path, recover metadata from backup, in worst case: temporarily move VMs off CSV, format and remount. |
| 11. Validate service | CSV online on all nodes. VMs running on CSV. VHD/X files accessible. |
| 12. Monitor | CSV health monitoring, LUN path monitoring. |
| 13. Document RCA | Root cause, resolution, and prevention (e.g., AV exclusion for CSVs). |

**Interview Questions:**
- "What is CSV and why is it important?"
- "CSV offline on all nodes — what do you check?"
- "How do you recover a corrupted CSV metadata file?"
- "What causes CSV blocking?"

---

### 5.5 — Quorum Problem

**Concept:** The cluster loses quorum and VMs stop running or cannot fail over.

**Architecture:** Quorum: Cluster requires a majority of votes to operate. Each node gets a vote (by default), plus witness (file/cloud/disk) adds additional vote to maintain majority.

**Components:** Cluster nodes, node weights, witness (file/cloud/witness disk/disk), Quorum configuration, dynamic quorum, cluster database.

**Configuration:** Quorum type (Node Majority, Node and File Share Majority, Node and Disk Majority, No Majority: Disk Only), dynamic quorum enabled/disabled, dynamic witness enabled/disabled.

**Commands:**
```powershell
# Check current quorum:
Get-ClusterQuorum | Select-Object Cluster, QuorumType, QuorumSource, VoteCount, QuorumState
# Check cluster nodes and votes:
Get-ClusterNode | Select-Object Name, State, NodeWeight, NodeFailing, HasVote, IsLastNodeStanding
# Check if cluster has quorum:
(Get-Cluster).ClusterQuorumState  # Quorum, NoQuorum
# Dynamic quorum settings:
Get-Cluster | Select-Object -ExpandProperty DynamicQuorumEnabled
Get-Cluster | Select-Object -ExpandProperty DynamicWitnessEnabled
# Enable dynamic quorum (recommended):
Set-Cluster -DynamicQuorumEnabled $true
Set-Cluster -DynamicWitnessEnabled $true
# Configure cloud witness:
Set-ClusterQuorum -CloudWitness -AccountName "storageaccount" -AccessKey "key"
# Configure file share witness:
Set-ClusterQuorum -FileShareWitness "\\server\share"
# Check node voting:
Get-ClusterNode | Where-Object {$_.HasVote -eq $false} | Select-Object Name  # nodes without votes
# Set node weight (give or remove vote):
Set-ClusterNode -Name Node01 -NodeWeight 1  # has vote
Set-ClusterNode -Name Node01 -NodeWeight 0  # no vote (recommended for stretched clusters with even nodes)
# Check quorum history:
Get-WinEvent -FilterHashtable @{LogName='System'; Id=1177, 1178, 1179, 1180, 1181, 1182} -MaxEvents 30
# Cluster reports (HTML):
C:\Windows\Cluster\Reports\Cluster.html  # comprehensive cluster report
# Check cluster database:
Get-ClusterDatabase | Select-Object Name, State
```

**Real-World Example:** 2-node cluster lost quorum after one node crashed — no witness configured. Added cloud witness via Azure storage account → quorum restored, dynamic quorum enabled so future single-node loss doesn't cause issues.

**Common Failures:**
- No witness configured in 2-node cluster
- Witness inaccessible (file share down, cloud account key expired)
- Even number of nodes without witness (tie)
- Node weights misconfigured (all nodes have vote, even-node cluster)
- Dynamic quorum disabled
- Network partition between nodes
- Witness disk/export failure (for disk witness)
- Cluster split-brain (nodes isolated from each other but both running)

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Cluster stopped running VMs? VMs failed over and won't come back? "Cluster has no quorum" message? |
| 2. Determine scope | Cluster-wide? All VMs? After specific event? |
| 3. Check recent changes | Node removed/added, witness change, node failure, network partition. |
| 4. Check monitoring | Cluster quorum status dashboard. Node status. Witness availability. |
| 5. Validate connectivity | Nodes can communicate? Witness accessible? Network between nodes healthy? |
| 6. Check OS | `Get-ClusterQuorum`. `(Get-Cluster).ClusterQuorumState`. `Get-ClusterNode` for node states and votes. |
| 7. Check dependencies | Witness online? Nodes have votes? Dynamic quorum enabled? |
| 8. Check logs | Event IDs 1177 (quorum lost), 1178 (quorum regained), 1179, 1180. Cluster log. `C:\Windows\Cluster\Reports\` for detailed analysis. |
| 9. Identify root cause | No witness? Witness down? Node weight issue? Network partition? |
| 10. Implement fix | Add/configure witness (cloud witness recommended), enable dynamic quorum and dynamic witness, fix node weights, fix network partition, restore witness access, force quorum (last resort, only if you know exactly which partition has the majority: `Set-ClusterQuorum -NodeAndFileShareMajority`). |
| 11. Validate service | Cluster has quorum. VMs running on appropriate nodes. `Test-Cluster` passes. VMs can fail over. |
| 12. Monitor | Quorum health, node count, witness availability. |
| 13. Document RCA | Root cause, fix, and cluster design improvement (e.g., always use cloud witness, enable dynamic quorum). |

**Interview Questions:**
- "What's quorum and why does it matter?"
- "How do you fix a cluster that lost quorum?"
- "What's dynamic quorum?"
- "Why is cloud witness preferred over disk witness?"
- "What happens in a 2-node cluster with no witness when one node fails?"

---

## 🔷 DOMAIN 6: PKI / CERTIFICATES (⭐⭐⭐⭐)

---

### 6.1 — Certificate Expired

**Concept:** A certificate has expired, causing authentication failures, TLS errors, and service outages.

**Architecture:** PKI hierarchy: Root CA → Subordinate/Issuing CA → End-entity certificates. Certificates have validity period (Not Before / Not After).

**Components:** CA server, certificate templates, certificate database, CRL/Delta CRL, AIA (Authority Information Access), OCSP, certificate stores, auto-enrollment, NTAuth certificates.

**Configuration:** CA validity period, certificate template validity period, auto-enrollment policy, CRL publication interval, AIA/CRL distribution points.

**Commands:**
```powershell
# Check certificate on local machine:
Get-ChildItem Cert:\LocalMachine\My | Where-Object {$_.NotAfter -lt (Get-Date)} | Select-Object Subject, Thumbprint, NotAfter, NotBefore
# Check a specific certificate:
Get-ChildItem Cert:\LocalMachine\My\THUMBPRINT | Select-Object *
# Check CA server certificates:
Get-ChildItem Cert:\LocalMachine\CA | Select-Object Subject, NotAfter, NotBefore
# Check Enterprise CA:
Get-CertificateAuthority | Select-Object *  # requires RSAT-ADCS-PowerShell
# Check pending certs:
Get-CertificateAuthority | Get-CertificateRequest | Where-Object {$_.RequestStatus -eq "Pending"}
# Check CRL:
certutil -CRL http://crl.domain.com/crl.pem
# Check certificate chain:
certutil -verify -urlfetch C:\path\to\cert.cer
# Check OCSP:
certutil -ocsp http://ocsp.domain.com
# Renew CA certificate:
certutil -caRenewCert  # on CA server
# Request new certificate (auto):
certutil -pulse  # trigger auto-enrollment
certutil -cainfo -autoenrollment
# On Windows Server CA:
# Check CA cert validity:
Get-WmiObject -Class Win32-CertificationAuthority | Select-Object *
# Check CRL publication:
certutil -crlgen
# Schedule CRL auto-publish:
certutil -setreg CA\CRLPeriodDays 7
certutil -setreg CA\CRLDeltaPeriod 1
# Force CRL publication:
certutil -crl
# Renew all expired certificates:
# For web servers (IIS):
Import-Module WebAdministration
Get-ChildItem IIS:\SslBindings | ForEach-Object {
    $cert = Get-ChildItem Cert:\LocalMachine\My | Where-Object {$_.Thumbprint -eq $_.Thumbprint}
    if ($cert.NotAfter -lt (Get-Date)) { Write-Host "$($_.Thumbprint) EXPIRED" }
}
```

**Real-World Example:** All RDP connections failed — RDP certificate expired on all servers. Used auto-enrollment with a GPO to auto-renew RDP certificates 30 days before expiry.

**Common Failures:**
- Certificate expired (not renewed in time)
- CA server offline (can't issue new certs)
- Auto-enrollment not configured
- CRL not published/accessible (OCSP/CRL down → chain validation fails)
- AIA pointer missing
- Certificate chain broken (missing intermediate)
- Private key lost (can't renew)
- Template expired
- NTAuth certificate expired (enterprise PKI doesn't trust CAs)

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | TLS errors? Authentication failures? "Certificate has expired" message? Which service? |
| 2. Determine scope | Single cert? All certs from one CA? Multiple servers? |
| 3. Check recent changes | CA change, cert template change, certificate not renewed, CA certificate expired. |
| 4. Check monitoring | Certificate expiration monitoring (Zabbix, SCOM, PowerShell script scanning cert stores). |
| 5. Validate connectivity | CA accessible? CRL/OCSP accessible? Client can reach CA? |
| 6. Check OS | Check cert stores: `Cert:\LocalMachine\My`, `Cert:\LocalMachine\Root`, `Cert:\LocalMachine\CA`. `certutil -viewstore -user my`. Check CA: `Get-CertificateAuthority`. |
| 7. Check dependencies | CA operational? CRL published? Auto-enrollment configured? Template valid? |
| 8. Check logs | Event Viewer: CA events (Applications and Services Logs → CA), Security log (certificate request events, 4705-4711), System log for auto-enrollment. |
| 9. Identify root cause | Certificate expired? CA expired? CRL not accessible? Template expired? |
| 10. Implement fix | Renew certificate (`certutil -pulse` for auto-enrollment, `certreq` for manual request), renew CA certificate (`certutil -caRenewCert`), fix CRL/AIA accessibility, reissue cert templates, fix NTAuth cert, manually install new cert. |
| 11. Validate service | New certificate in place, valid dates, chain valid (`certutil -verify`), TLS handshake works, service functional. |
| 12. Monitor | Certificate expiration monitoring (alert at 30/15/7 days). CA CRL publication monitoring. |
| 13. Document RCA | Root cause, fix, and certificate lifecycle management process. |

**Interview Questions:**
- "A server certificate expired — how do you renew it?"
- "What's the difference between CRL and OCSP?"
- "How do you check the certificate chain?"
- "What are NTAuth certificates and why do they matter?"

---

### 6.2 — Certificate Chain Failure

**Concept:** Certificate chain validation fails — client cannot build a trust chain from the end-entity certificate to a trusted root CA.

**Architecture:** Chain building: End-entity cert → Intermediate CA cert(s) → Root CA cert. Trust requires: root in trusted store, all intermediates available (AIA/CRL), valid signatures at each level.

**Components:** Root CA certificate, intermediate CA certificates, AIA pointers, CRL distribution points, OCSP responder, certificate stores (Root, CA, Intermediate), cross-certification.

**Configuration:** AIA and CDP URLs in certificate extensions, trust settings, auto-enrollment for intermediate certs, cross-certification between PKIs.

**Commands:**
```powershell
# Check certificate chain:
Get-ChildItem Cert:\LocalMachine\My\THUMBPRINT | Select-Object Subject, Thumbprint, NotAfter
# Use certutil to verify chain:
certutil -verify -urlfetch https://server/cert.cer
# Or:
certutil -verify C:\path\to\certificate.cer
# Check certificate details (including AIA/CDP):
certutil -v -dump C:\path\to\certificate.cer
# Check if intermediate certs are installed:
Get-ChildItem Cert:\LocalMachine\CA | Select-Object Subject, Thumbprint, NotAfter
# Check if root is trusted:
Get-ChildItem Cert:\LocalMachine\Root | Where-Object {$_.Subject -like "*CA Name*"} | Select-Object Subject, Thumbprint
# Check certificate chain on server:
# For IIS:
Get-WebBinding | Where-Object {$_.Protocol -eq "https"} | Select-Object *
# Check SSL certificate binding:
Get-ChildItem IIS:\SslBindings | Select-Object *, @{N='CertChain';E={
    $cert = Get-ChildItem Cert:\LocalMachine\My\$_.Thumbprint
    $chain = New-Object System.Security.Cryptography.X509Certificates.X509Chain
    $chain.Build($cert)
    $chain.ChainStatus | Select-Object Status, StatusInformation
}}
# For individual client/server:
# OpenSSL test (if available):
openssl s_client -connect server.domain.com:443 -showcerts
# Or PowerShell:
$tls = [System.Net.Security.SslStream]::new([System.Net.Sockets.TcpClient]::new("server.domain.com", 443).Client, $false, (Callback)) 
# Simpler:
$req = [System.Net.Http.HttpClient]::new()
$req.GetAsync("https://server.domain.com")  # check TLS errors
# Check AIA/CRP retrieval:
# Use certutil:
certutil -urlfetch -verify cert.cer
# Check certificate chain on specific endpoint:
Test-NetConnection -ComputerName server.domain.com -Port 443
# Force chain building:
Get-ChildItem Cert:\LocalMachine\My | ForEach-Object {
    $chain = New-Object System.Security.Cryptography.X509Certificates.X509Chain
    $chain.Build($_) | Out-Null
    $_.Subject | ForEach-Object {
        $chain.ChainStatus | ForEach-Object {
            if ($_.Status -ne 'NoError') {
                [PSCustomObject]@{ Subject = $_.Subject; Status = $_.Status; Info = $_.StatusInformation }
            }
        }
    }
}
```

**Real-World Example:** Internal app users got "untrusted connection" — intermediate CA certificate was missing from servers. Imported intermediate CA cert to `Intermediate Certification Authorities` store on all servers.

**Common Failures:**
- Intermediate CA certificate missing
- Root CA not in trusted root store
- AIA URL unreachable (can't download intermediate)
- CRL/OCSP unreachable (chain validation requires CRL fetch)
- Root CA certificate removed from trust store
- Firewall blocking AIA/CRL URLs
- Cross-certification missing (PKI trust between forests)
- Certificate signed with unsupported algorithm (SHA-1 disabled)
- Self-signed cert used without adding to trusted store

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | "Certificate chain error"? "Untrusted"? "Certification chain not trusted"? Specific application or all HTTPS? |
| 2. Determine scope | One server? Multiple? Specific cert? All HTTPS on server? |
| 3. Check recent changes | CA hierarchy change, intermediate cert renewal, root CA change, trust settings changed. |
| 4. Check monitoring | Certificate chain validation monitoring, AIA/CRL accessibility monitoring. |
| 5. Validate connectivity | Client can reach AIA URL? Can reach CRL? Can reach OCSP? `Test-NetConnection` to AIA/CDP URLs. |
| 6. Check OS | Check `Root`, `CA` (intermediate), `My` stores. `certutil -verify -urlfetch cert.cer`. OpenSSL `openssl s_client` for live chain check. |
| 7. Check dependencies | All intermediate certs available? Root trusted? AIA/CRL accessible? |
| 8. Check logs | Event Viewer: Security (cert validation events), System (cert chain errors), Application. `certutil` chain validation output. |
| 9. Identify root cause | Intermediate missing? Root not trusted? CRL unreachable? Firewall blocking? |
| 10. Implement fix | Import intermediate cert to `Intermediate Certification Authorities` store, add root to `Trusted Root Certification Authorities`, publish AIA/CDP on accessible HTTP URL (not file://), enable CRL caching, fix firewall rules, configure `CertificateTrustList` via GPO, disable CRL checking (not recommended for production but as workaround), configure proper chain. |
| 11. Validate service | `certutil -verify` passes. HTTPS connection succeeds. Application works. `openssl s_client` shows full chain. |
| 12. Monitor | Chain validation success rate, AIA/CRL reachability. |
| 13. Document RCA | Root cause, fix, and CA administration process update. |

**Interview Questions:**
- "How do you troubleshoot a broken certificate chain?"
- "What's the difference between AIA and CDP?"
- "How do you handle CRL in disconnected environments?"
- "How do you enable certificate chain logging?"

---

### 6.3 — Auto-Enrollment Failure

**Concept:** Certificates that should be automatically requested and renewed via GPO auto-enrollment are not being issued/renewed.

**Architecture:** Auto-enrollment: GPO → Certificate Services Client Extension (certutil / Pulse) → Enrollment agent (certenroll.dll) → CA → certificate issued → stored in user/machine store. Runs via SYSTEM account.

**Components:** GPO certificate settings, Certificate Services Client, DCOM, enrollment agent, CA, template, certificate stores, NTAuth certs.

**Configuration:** GPO → Computer Config → Windows Settings → Security Settings → Public Key Policies → Certificate Services Client → Auto-enrollment (enabled/disabled, renewal, certificate trust list). Certificate templates (auto-enroll enabled).

**Commands:**
```powershell
# Force auto-enrollment:
certutil -pulse  # triggers auto-enrollment
certutil -cainfo -autoenrollment  # shows auto-enrollment info
certutil -cainfo -ca  # CA info
certutil -store My  # check personal store
certutil -store Root  # check root store
certutil -store CA  # check intermediate store
# Check auto-enrollment results:
certutil -cainfo -autoenrollment 2
# Check GPO certificate settings:
# gpresult /h C:\gpreport.html
# Check in report: Computer Configuration → Policies → Windows Settings → Security Settings → Public Key Policies
# Check if auto-enrollment is configured:
Get-GPRegistryValue -Key "HKLM\SOFTWARE\Policies\Microsoft\SystemCertificates\AuthRoot" -Name AutoCertificateImport  # for root CAs
Get-GPRegistryValue -Key "HKLM\SOFTWARE\Policies\Microsoft\SystemCertificates\My" -Name AutoEnrollment  # for personal certs
# Re-enroll manually:
certreq -submit -config "CAName\TemplateName" C:\request.inf C:\cert.cer
# Check pending requests:
Get-CertificateRequest | Where-Object {$_.RequestStatus -eq "Pending"}  # on CA server
# On CA server:
certutil -view -restrict "Request Disp;" -out RequestId, RequestStatus, RequesterName, CertificateTemplate
# Check certificate template:
certutil -template -v TemplateName  # on CA server
# Check if template is configured for auto-enrollment:
# Template: In CA console → Certificate Templates → Properties → Request Handling → "Allow private key to be exported" + "Enrollment" rights on security tab
# Check AutoEnrollment on template: "Allow Auto-enrollment" or "Read" permission on "Enroll" for authenticated users
# Troubleshoot auto-enrollment with detailed logging:
# Enable cert enrollment debug:
certutil -setreg ca\AuditFilter 0xff  # enable CA audit logging
# Set registry key for cert client logging:
# HKLM\SOFTWARE\Microsoft\Cryptography\Debug\OID
certutil -setreg chain\ChainCacheResyncFiletime @now
# Event Viewer for auto-enrollment:
# Applications and Services Logs → Microsoft → Windows → CertificationServicesClient → Enrollment (Event ID 44, 45, 46, 47, 48)
Get-WinEvent -LogName "Microsoft-Windows-CertificationServicesClient-Enrollment/Operational" -MaxEvents 50
# CA server events:
Get-WinEvent -LogName "Microsoft-Windows-CertificationServices-CA-Admin" -MaxEvents 50
```

**Real-World Example:** Computer certificates not auto-enrolling — GPO was modified and auto-enrollment node was set to "Disabled." Re-enabled and ran `certutil -pulse` on clients.

**Common Failures:**
- Auto-enrollment GPO disabled/removed
- Certificate template expired or deleted
- Template "Auto-enroll" flag disabled
- Insufficient permissions on template (Enroll/AutoEnroll)
- CA not operational or full
- DCOM issues (enrollment uses DCOM)
- NTAuth certificates expired
- Client unable to reach CA
- Template security ACL wrong (Enroll/Read permission missing)
- Certificate expiration not configured for auto-renewal
- Group Policy not applied (client not in correct OU)
- Template not published to CA
- Template version mismatch

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Auto-enrollment expected but not happening? Specific template? All clients or some? |
| 2. Determine scope | All domain clients? Specific OU? Specific template? |
| 3. Check recent changes | GPO change, template change, CA change, certificate renewal. |
| 4. Check monitoring | Enrollment event logs, CA queue monitoring, certificate expiration dashboard. |
| 5. Validate connectivity | Client can reach CA (RPC/Http on port 443/135/4444). Can reach DC (for GPO and policy). Can reach CRL/AIA URLs. |
| 6. Check OS | `certutil -pulse` results (errors?). `certutil -cainfo -autoenrollment`. Check `Cert:\LocalMachine\My` for expected certs. `gpresult /r` (GPO applied?). GPO registry settings for auto-enrollment. |
| 7. Check dependencies | GPO correct? Template published and valid? CA operational? NTAuth valid? Template permissions? |
| 8. Check logs | Enrollment operational log (Event IDs 44, 45, 46, 47, 48). CA admin log. `certutil` output. |
| 9. Identify root cause | GPO disabled? Template expired? Template permissions? CA down? NTAuth expired? |
| 10. Implement fix | Enable auto-enrollment in GPO, renew/republish template, fix template ACLs, restart CA, add/repair NTAuth certificate, run `certutil -pulse` on clients, fix GPO links/permissions. |
| 11. Validate service | Certificates enroll correctly via `certutil -pulse`. Check cert stores. Verify certificate validity period. |
| 12. Monitor | Enrollment event logs, certificate expiration tracking. |
| 13. Document RCA | Root cause, fix, and certificate lifecycle process improvement. |

**Interview Questions:**
- "Auto-enrollment not working — how do you troubleshoot?"
- "What Event IDs relate to certificate enrollment?"
- "How do certificate templates affect auto-enrollment?"
- "What is NTAuth and why does it matter for auto-enrollment?"

---

### 6.4 — TLS Failure

**Concept:** TLS handshake fails between client and server, preventing secure communication.

**Architecture:** TLS 1.2/1.3: Client sends ClientHello with supported cipher suites → Server selects cipher suite and sends Certificate → Key exchange → Handshake complete → encrypted session.

**Components:** TLS protocol versions, cipher suites, certificates, private keys, SPN/hostname matching, CipherSuite order, registry TLS settings, SCHANNEL, .NET TLS configuration.

**Configuration:** Registry: `HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols` (TLS 1.0/1.1/1.2/1.3), `HKLM\SOFTWARE\Policies\Microsoft\.NETFramework\v4.0.30319` (SchUseStrongCrypto), cipher suite order, certificate bindings (SNI).

**Commands:**
```powershell
# Check TLS protocol settings:
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Server" -ErrorAction SilentlyContinue
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Client" -ErrorAction SilentlyContinue
# Check TLS 1.3:
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.3\Server" -ErrorAction SilentlyContinue
# Check cipher suites:
Get-TlsCipherSuite | Select-Object Name, CipherSuite, HashAlgorithm, KeyExchangeAlgorithm | Format-Table
# Test TLS connection:
Test-NetConnection -ComputerName server.domain.com -Port 443  # basic connectivity
# PowerShell TLS test:
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
$req = [Net.HttpWebRequest]::new("https://server.domain.com")
try { $req.GetResponse() } catch { $_.Exception.Message }
# Or with PowerShell 7+:
Invoke-WebRequest -Uri https://server.domain.com -UseBasicParsing
# Detailed TLS info:
# Use OpenSSL (if available on server):
openssl s_client -connect server.domain.com:443 -tls1_2
openssl s_client -connect server.domain.com:443 -tls1_3
# Or with Python:
python3 -c "import ssl; print(ssl.OPENSSL_VERSION)"
# Windows: Use Test-TlsConnection (if module available) or Network Monitor/ Wireshark
# Check IIS TLS settings:
Get-WebBinding | Where-Object {$_.Protocol -eq "https"} | Select-Object *
# Check certificate on IIS binding:
Get-ChildItem IIS:\SslBindings | Select-Object Port, IPAddress, Thumbprint
Get-ChildItem Cert:\LocalMachine\My | Where-Object {$_.NotAfter -gt (Get-Date)} | Select-Object Subject, Thumbprint, NotAfter
# Check TLS registry (full settings):
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols" -ErrorAction SilentlyContinue
# Registry for strong cryptography (32-bit and 64-bit):
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\.NETFramework\v4.0.30319" -Name SchUseStrongCrypto -ErrorAction SilentlyContinue
Get-ItemProperty "HKLM:\SOFTWARE\WOW6432Node\Policies\Microsoft\.NETFramework\v4.0.30319" -Name SchUseStrongCrypto -ErrorAction SilentlyContinue
# Check SChannel events:
Get-WinEvent -LogName System | Where-Object {$_.Id -eq 36871 -or $_.Id -eq 36872 -or $_.Id -eq 36873 -or $_.Id -eq 36887 -or $_.Id -eq 36888 -or $_.Id -eq 36889} -MaxEvents 30
# Event IDs:
# 36871: TLS 1.2 not supported
# 36872: TLS 1.2 disabled
# 36873: TLS 1.2 handshake failed
# 36887: Remote server terminated handshake
# 36888: Remote server does not support TLS 1.2
# 36889: Fatal alert
# Application log: Schannel errors
```

**Real-World Example:** App stopped connecting to API — TLS 1.0 disabled on server but app was using TLS 1.0. Enabled TLS 1.2, app upgraded, connectivity restored.

**Common Failures:**
- TLS version mismatch (server only allows 1.2, client only supports 1.0)
- Cipher suite mismatch (no common cipher suite)
- Certificate expired or chain issue (not TLS-specific but TLS handshake fails)
- Certificate CN/SAN doesn't match hostname
- SNI not configured (multiple sites on same IP:port)
- TLS 1.2 not enabled (registry not configured, common after server build)
- .NET Framework app using old TLS (need SchUseStrongCrypto registry)
- Mixed HTTP/HTTPS proxy issues
- CRL/OCSP check timing out during TLS handshake
- Weak cipher suites disabled on server, client doesn't support strong ones
- TLS 1.3 incompatibility
- FIPS mode enabled restricting cipher suites

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | TLS handshake failed? Connection refused? Specific app? All HTTPS? Error message ("TLS not supported", "no common cipher suite")? |
| 2. Determine scope | One app? All apps? One server? All servers? One client OS version? |
| 3. Check recent changes | TLS settings changed, certificate renewed, app updated, server patched (may disable TLS 1.0/1.1). |
| 4. Check monitoring | TLS handshake success/failure rate. Certificate expiration. |
| 5. Validate connectivity | Port 443 open. CA accessible (for CRL checks during handshake). |
| 6. Check OS | TLS protocol enabled? `Get-ItemProperty` for TLS 1.2/1.3 in Server/Client registry. TLS cipher suites: `Get-TlsCipherSuite`. App .NET TLS settings: SchUseStrongCrypto. |
| 7. Check dependencies | TLS 1.2+ enabled on both? Common cipher suite? Certificate valid and chain trusted? |
| 8. Check logs | Schannel events (36871, 36872, 36873, 36887-36889). Application log for .NET TLS errors. `openssl s_client` output for detailed handshake diagnostics. |
| 9. Identify root cause | TLS 1.0/1.1 disabled and client doesn't support 1.2? Cipher suite mismatch? Expired cert? .NET app not configured for strong crypto? |
| 10. Implement fix | Enable appropriate TLS versions via registry + reboot, configure cipher suite order, set SchUseStrongCrypto for .NET apps, renew certificate, fix SNI configuration, configure CRL caching (to avoid handshake delays), update app to support modern TLS. |
| 11. Validate service | TLS handshake succeeds. App connects via HTTPS. `openssl s_client` shows valid chain and acceptable cipher. |
| 12. Monitor | TLS handshake success rate. Protocol version usage. |
| 13. Document RCA | Root cause, fix, TLS configuration standard for all servers. |

**Interview Questions:**
- "TLS handshake failing — what do you check?"
- "How do you enable TLS 1.2 on Windows Server?"
- "What's SchUseStrongCrypto and when do you use it?"
- "How do you debug a TLS issue between client and server?"

---

## 🔷 DOMAIN 7: AZURE IaaS / ARCT (⭐⭐⭐⭐)

---

### 7.1 — Azure VM Unreachable

**Concept:** An Azure VM cannot be reached from the internet or from within the Azure VNet.

**Architecture:** Azure VM → vNIC → Azure Virtual Network → Subnet → NSG → Load Balancer/Public IP → Internet. Azure manages the physical hypervisor (hypervisor hidden from user).

**Components:** Azure VM, OS disk, data disk, NIC, NSG, subnet, VNet, public IP, load balancer, availability set, VMSS, Azure Bastion, Jumpbox.

**Configuration:** NSG rules (inbound/outbound), public IP allocation, NIC NSG association, VM OS firewall, boot diagnostics, serial console, Azure Monitor agent.

**Commands:**
```powershell
# Azure PowerShell:
# Check VM status:
Get-AzVM -ResourceGroupName RGName -Name VMName | Select-Object Name, ProvisioningState, PowerState, Location
# Check VM instance view (agent status, boot diagnostics):
Get-AzVM -ResourceGroupName RGName -Name VMName -Status | Select-Object -ExpandProperty Statuses | Where-Object {$_.Code -like "*health*"}
# Check VM agent:
Get-AzVM -ResourceGroupName RGName -Name VMName -InstanceView | Select-Object -ExpandProperty Statuses
# Check NSG rules:
Get-AzNetworkSecurityGroup -ResourceGroupName RGName -Name NSGName | Select-Object -ExpandProperty SecurityRules | Format-Table Name, Direction, Access, Protocol, SourcePortRange, DestinationPortRange, SourceAddressPrefix, DestinationAddressPrefix, Priority
# Check NIC:
Get-AzNetworkInterface -ResourceGroupName RGName -Name VMName-NIC | Select-Object *
# Check NSG association on NIC:
Get-AzNetworkInterface -ResourceGroupName RGName -Name VMName-NIC | Select-Object -ExpandProperty NetworkSecurityGroup
# Check public IP:
Get-AzPublicIpAddress -ResourceGroupName RGName -Name PIPName | Select-Object Name, IpAddress, AllocationMethod, Sku
# Check if VM is reachable:
Test-NetConnection -ComputerName VM_PUBLIC_IP -Port 3389
Test-NetConnection -ComputerName VM_PUBLIC_IP -Port 80
# Check route table:
Get-AzRouteTable -ResourceGroupName RGName | Select-Object *
Get-AzVirtualNetwork -ResourceGroupName RGName -Name VNetName | Select-Object -ExpandProperty Subnets | Select-Object Name, AddressPrefix, NetworkSecurityGroup, RouteTable
# Check VNet peering:
Get-AzVirtualNetworkPeering -ResourceGroupName RGName -VirtualNetworkName VNetName | Select-Object *
# Check boot diagnostics (serial console):
Get-AzVM -ResourceGroupName RGName -Name VMName -ShowDiagnosticsProfile
# Check VM boot diagnostics:
Get-AzBootDiagnostics -ResourceGroupName RGName -Name VMName
# If VM is running but unreachable:
# Check OS firewall from within VM (via Azure Bastion or serial console):
# On VM: Get-NetFirewallRule | Where-Object {$_.Enabled -eq 'True'} | Select-Object DisplayName, Direction, Action, RemotePort
# Check Azure service health:
Get-AzServiceHealth -Warning
Get-AzServiceHealth -Event
```

**Real-World Example:** VM became unreachable after NSG rule was added with priority 100 (higher priority) denying all inbound traffic. Deleted the NSG rule (priority 65000 was the original allow).

**Common Failures:**
- NSG rule blocking traffic (port 3389/22/80 closed)
- Public IP not allocated or reassigned
- VM OS firewall blocking traffic
- VM not running (deallocated)
- Subnet NSG conflicting with VM NSG
- Route table misconfigured (UDR sending traffic elsewhere)
- VM agent not responding (extensions not working)
- Public IP SKU mismatch (Standard vs Basic with Load Balancer)
- Port exhaustion on VM
- VM host issue (Azure-level, contact support)
- Accelerated networking disabled causing performance issues
- IP address conflict (static IP configured wrong)

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | VM unreachable from where? Internet? Same VNet? Different VNet? All ports or specific? |
| 2. Determine scope | One VM? Multiple in same subnet? All in same NSG? |
| 3. Check recent changes | NSG rule changed, IP changed, VM restarted, OS update, extension deployed. |
| 4. Check monitoring | Azure VM health. Azure Service Health. VM agent status. Network monitoring. |
| 5. Validate connectivity | `Test-NetConnection` from public internet and from other Azure VMs on different ports. Ping (ICMP may be blocked by NSG). |
| 6. Check OS | Azure Portal: VM → Boot diagnostics (serial console output). Azure Portal: VM → Networking (effective NSG rules). From within VM (via Bastion/serial): `ipconfig`, `Get-NetFirewallRule`, `netstat -an`. |
| 7. Check dependencies | VM running? Public IP associated? NSG rules allow traffic? VM firewall allows traffic? VM agent responding? |
| 8. Check logs | Azure Activity Log for VM operations. VM boot diagnostics. NSG flow logs (if enabled). OS firewall logs. Event logs on VM. `Get-AzVM -Status` for agent health. |
| 9. Identify root cause | NSG blocking? OS firewall? VM agent down? NIC issue? Public IP issue? |
| 10. Implement fix | Add NSG rule (allow specific port from desired source), fix OS firewall rules, restart VM, restart Azure VM agent (`Restart-AzVM -Agent`), fix public IP association, fix NIC configuration, redeploy VM (last resort), contact Microsoft support if host-level issue. |
| 11. Validate service | VM reachable on required ports from expected sources. `Test-NetConnection` passes. |
| 12. Monitor | Azure VM availability, NSG flow logs, Azure Monitor alerts. |
| 13. Document RCA | Root cause, fix, and NSG change management process. |

**Interview Questions:**
- "An Azure VM is unreachable — what's your approach?"
- "How do you check effective NSG rules?"
- "What's the difference between subnet NSG and VM NIC NSG?"
- "How do you connect to a VM with no network access?"

---

### 7.2 — NSG Blocking Traffic

**Concept:** Network Security Group rules are preventing expected traffic flow.

**Architecture:** NSG contains security rules (priority 100-4096) evaluated in order by priority. First matching rule determines Allow/Deny. Default rules exist at priorities 65500-65533.

**Components:** NSG rules (inbound/outbound), priorities, source/destination IPs/ports, service tags, application security groups, NSG association (subnet, NIC, or both).

**Configuration:** Rule priority, action (Allow/Deny), source/destination, protocol, port range, service tags (AzureLoadBalancer, VirtualNetwork, Internet, etc.), description.

**Commands:**
```powershell
# List all NSG rules:
$NSG = Get-AzNetworkSecurityGroup -ResourceGroupName RGName -Name NSGName
$NSG.SecurityRules | Format-Table Name, Priority, Direction, Access, Protocol, SourcePortRange, DestinationPortRange, SourceAddressPrefix, DestinationAddressPrefix
# Check effective NSG rules for a specific VM/port:
Get-AzNetworkInterface -ResourceGroupName RGName -Name VMName-NIC | Select-Object -ExpandProperty NetworkSecurityGroup
# Use "Effective Security Rules" in portal: VM → Networking → Effective NSG Rules
# Check if traffic is being allowed/denied:
# Check flow logs (if configured):
Get-AzNetworkWatcherConfigFlowLog -ResourceGroupName RGName -NetworkWatcherName NWName -Location LocationName
# Start packet capture:
New-AzNetworkWatcherPacketCapture -ResourceGroupName RGName -NetworkWatcherName NWName -TargetResourceId $VM.Id -StorageAccount SAName -CapacityInGB 10
# Add NSG rule:
New-AzNetworkSecurityRuleConfig -Name "Allow-RDP" -Protocol Tcp -Direction Inbound -Priority 1000 -SourceAddressPrefix Internet -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 3389 -Access Allow
# Update NSG:
Get-AzNetworkSecurityGroup -ResourceGroupName RGName -Name NSGName | Set-AzNetworkSecurityGroup
# Check service tags:
Get-AzNetworkSecurityGroup -ResourceGroupName RGName -Name NSGName | Select-Object -ExpandProperty SecurityRules | Where-Object {$_.SourceAddressPrefix -like "*Cloud*" -or $_.SourceAddressPrefix -like "*Azure*"}
# Check NSG association:
Get-AzVirtualNetwork -ResourceGroupName RGName -Name VNetName | Select-Object -ExpandProperty Subnets | Select-Object Name, NetworkSecurityGroup
Get-AzNetworkInterface -ResourceGroupName RGName | Where-Object {$_.NetworkSecurityGroup.Id -like "*NSGName*"} | Select-Object Name
# Remove a rule:
$NSG | Get-AzNetworkSecurityRuleConfig -Name "BadRule" | Remove-AzNetworkSecurityRuleConfig
$NSG | Set-AzNetworkSecurityGroup
```

**Real-World Example:** Web app couldn't receive traffic — NSG rule for port 443 was missing on VM NIC NSG (subnet NSG had it, but VM also had its own NSG attached). Added port 443 rule to VM NSG.

**Common Failures:**
- Rule missing for required port/protocol
- Rule has wrong priority (lower priority deny rule exists above it)
- Rule has wrong source/destination prefix (too restrictive)
- Service tag used incorrectly (e.g., using "Internet" tag on internal traffic)
- Deny rule more specific than allow rule (higher priority)
- NSG not associated correctly (attached to subnet when it should be on NIC or vice versa)
- Multiple NSGs with conflicting rules
- Default NSG rules (Allow Internet Inbound was disabled — common in Azure policy)
- Application Security Group misconfiguration

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | What traffic blocked? Port, protocol, source, destination? |
| 2. Determine scope | Specific VM? All VMs in subnet? All VMs in NSG? |
| 3. Check recent changes | NSG rule change, VM NSG association changed, new VM deployed with different NSG. |
| 4. Check monitoring | NSG flow logs (if enabled). Azure Monitor. Network Watcher. |
| 5. Validate connectivity | From source, `Test-NetConnection VM_IP -Port PortNumber`. From VM, `Test-NetConnection` to expected destination. |
| 6. Check OS | Azure Portal: VM → Networking → Effective NSG Rules (shows which rule applies to traffic). Check all NSGs associated with VM (subnet + NIC). `Get-AzNetworkSecurityGroup -Name ... | Select-Object -ExpandProperty SecurityRules`. |
| 7. Check dependencies | NSG rules in correct order? Appropriate service tags? Correct association? |
| 8. Check logs | NSG flow logs (if enabled). Network Watcher packet capture. Activity log for NSG changes. |
| 9. Identify root cause | NSG rule missing? Wrong priority? Conflicting NSGs? Wrong association? |
| 10. Implement fix | Add correct NSG rule with appropriate priority (above any deny rules), correct source/destination, verify NSG association (subnet vs NIC), remove conflicting deny rules, adjust priority order. |
| 11. Validate service | Traffic flows successfully through required ports. `Test-NetConnection` passes. Network Watcher verifies correct NSG rule match. |
| 12. Monitor | NSG flow logs for blocked connections. Alert on unexpected deny patterns. |
| 13. Document RCA | Root cause, fix, and NSG change documentation (review process, naming conventions). |

**Interview Questions:**
- "How do you troubleshoot NSG blocking?"
- "How do NSG priorities work?"
- "Difference between subnet NSG and NIC NSG?"
- "How do you test if a specific NSG rule is blocking traffic?"

---

### 7.3 — Azure Arc Disconnected

**Concept:** An Azure Arc-connected on-premises server or cluster loses connection to Azure.

**Architecture:** Azure Arc agent (`arc-agent`) runs on the machine, communicates with Azure via HTTPS (port 443). Arc extends Azure management to on-prem resources (non-Azure servers, Kubernetes clusters, data services).

**Components:** Arc agent, Azure Arc resource provider, ARC data controller (for data services), connected machine (HM), Kubernetes extension, Azure Resource Graph, hybrid identity (Azure AD).

**Configuration:** Arc resource group, connected machine resource in Azure, resource ID on machine, proxy settings (if applicable), outbound internet connectivity.

**Commands:**
```powershell
# Check Arc agent status (on Windows):
Get-Service "AzureConnectedMachineAgent" | Select-Object Name, Status, StartType
Get-Service "AzureArcSetup" | Select-Object Name, Status, StartType
# Check Arc agent status (on Linux):
systemctl status arcagent
systemctl status azure-arc-agent
# Check machine connectivity in Azure:
Get-AzConnectedMachine -ResourceGroupName RGName -Name MachineName | Select-Object Name, Status, ResourceId, LastConnectivityTime
# Check Arc agent logs (Windows):
Get-Content "C:\Program Files\Microsoft Azure Arc\AzureConnectedMachineAgent\Logs\*.log" -Tail 100
Get-Content "C:\ProgramData\Microsoft\AzureArc\`*.log" -Tail 100
# Check Arc agent logs (Linux):
/var/lib/azure/azcmagent/logs/*.log
# Reconnect (restart agent):
Restart-Service AzureConnectedMachineAgent
# On Linux:
sudo systemctl restart arcagent
# Check machine identity:
Get-AzConnectedMachine -ResourceGroupName RGName -Name MachineName | Select-Object Name, Identity, Status
# Check proxy configuration (if applicable):
# Arc agent config file (Linux): /etc/azure/azcmagent/config.toml
# Arc agent config (Windows): registry or environment variables
# Check if machine can reach Azure:
Test-NetConnection -ComputerName management.azure.com -Port 443
Test-NetConnection -ComputerName *.azure.com -Port 443
# Check Azure Arc resource in portal:
Get-AzResource -ResourceGroupName RGName | Where-Object {$_.ResourceType -like "*azureArc*"}
# Re-onboard (if needed):
# Remove machine from Arc:
az connectedmachine remove --name MachineName --resource-group RGName
# Then re-onboard using:
# https://docs.microsoft.com/en-us/azure/arc/servers/onboard
# or via portal: Azure Arc → Servers → Add → Onboard
# Check for Arc agent updates:
Get-AzConnectedMachine -ResourceGroupName RGName -Name MachineName | Select-Object *, LastConnectivityTime
```

**Real-World Example:** Arc server went offline after proxy server was changed — updated agent config with new proxy settings, service restarted, reconnected.

**Common Failures:**
- Internet connectivity lost (firewall, proxy, gateway)
- Agent service stopped/crashed
- Agent outdated (version mismatch with Azure)
- Proxy not configured or misconfigured
- Machine identity expired or changed
- DNS resolution of Azure endpoints failing
- Proxy certificate not trusted by agent
- Time skew (Kerberos/authentication failure)
- Arc resource deleted from Azure
- Resource group locked/blocked
- Insufficient disk space for agent logs

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Machine shows "Disconnected" in Arc portal? All Arc features failing? Intermittent? |
| 2. Determine scope | One machine? Multiple on-prem? Multiple Arc clusters? |
| 3. Check recent changes | Network change, proxy change, agent update, machine restart, certificate change. |
| 4. Check monitoring | Azure Arc connectivity monitoring. Agent service health. Network monitoring. |
| 5. Validate connectivity | Machine can reach `management.azure.com:443`, `*.azure.com:443`. DNS resolution works. Proxy configured correctly. |
| 6. Check OS | Agent service running? `Get-Service AzureConnectedMachineAgent`. Agent version current? Check logs for errors. |
| 7. Check dependencies | Internet reachability? Proxy configured? DNS works? Agent healthy? |
| 8. Check logs | Arc agent logs (`C:\Program Files\Microsoft Azure Arc\AzureConnectedMachineAgent\Logs\`). Windows Event Log: Arc-related events. Linux: `/var/lib/azure/azcmagent/logs/`. |
| 9. Identify root cause | Agent crashed? Network issue? Proxy wrong? DNS broken? Agent outdated? |
| 10. Implement fix | Restart agent service, update agent to latest version, fix proxy configuration, fix DNS, check network connectivity, check proxy certificates, fix machine identity, re-onboard if necessary. |
| 11. Validate service | Machine shows "Connected" in Azure Arc portal. Extensions reporting. Logs show successful communication. |
| 12. Monitor | Arc connectivity alerting, agent version monitoring, log size monitoring. |
| 13. Document RCA | Root cause, fix, and prevention (e.g., agent auto-update policy, network monitoring for Azure endpoints). |

**Interview Questions:**
- "Azure Arc disconnected — what do you check?"
- "What ports does Azure Arc need for connectivity?"
- "How do you troubleshoot Arc behind a proxy?"
- "When would you re-onboard vs. just restarting the agent?"

---

### 7.4 — Azure VM Performance Issue

**Concept:** Azure VM exhibits slow performance, high CPU/memory/disk usage, or network saturation.

**Architecture:** Azure VM = virtual hardware on Azure hypervisor. Resources depend on VM size (SKU). Storage depends on managed disk type (Premium SSD, Standard SSD, HDD). Network depends on VM size bandwidth tier and bandwidth allocation.

**Components:** VM size (CPU/RAM), managed disk type/IOPS/throughput, vNIC bandwidth, accelerated networking, NSG, Azure Load Balancer, Application Gateway, OS inside VM.

**Configuration:** VM SKU, disk type and size, caching (Read/Write/None), network security rules, Azure Monitor, VM insights, disk IOPS limits.

**Commands:**
```powershell
# Check VM size and current utilization:
Get-AzVM -ResourceGroupName RGName -Name VMName | Select-Object Name, HardwareProfile, StorageProfile, OsProfile
# Get VM metrics from Azure:
Get-AzMetric -ResourceId $VM.Id -MetricName "Percentage CPU" -TimeGrain 00:05:00 -TimeWindow 01:00:00
Get-AzMetric -ResourceId $VM.Id -MetricName "Available Memory Bytes" -TimeGrain 00:05:00 -TimeWindow 01:00:00
Get-AzMetric -ResourceId $VM.Id -MetricName "Disk Read Bytes" -TimeGrain 00:05:00 -TimeWindow 01:00:00
Get-AzMetric -ResourceId $VM.Id -MetricName "Disk Write Bytes" -TimeGrain 00:05:00 -TimeWindow 01:00:00
Get-AzMetric -ResourceId $VM.Id -MetricName "Network In" -TimeGrain 00:05:00 -TimeWindow 01:00:00
Get-AzMetric -ResourceId $VM.Id -MetricName "Network Out" -TimeGrain 00:05:00 -TimeWindow 01:00:00
# Check VM SKU limits:
Get-AzVMSize -Location "East US" | Where-Object {$_.Name -like "*Standard_D*"} | Select-Object Name, NumberOfCores, MemoryInMb, MaxDataDiskCount, ResourceDiskSize, ResourceDiskIOPS, ResourceDiskMBps
# Check disk type and IOPS:
Get-AzDisk -ResourceGroupName RGName -Name DiskName | Select-Object Name, DiskSizeGB, Sku, DiskIOPSReadWrite, DiskMBpsReadWrite, ProvisioningState
# Check if accelerated networking enabled:
Get-AzNetworkInterface -ResourceGroupName RGName -Name VMName-NIC | Select-Object EnableAcceleratedNetworking, EnableIPForwarding
# Check VM agent and extensions:
Get-AzVM -ResourceGroupName RGName -Name VMName -InstanceView | Select-Object -ExpandProperty Statuses
# Inside VM (via Bastion):
# Check within guest:
Get-Counter '\Processor(_Total)\% Processor Time'
Get-Counter '\Memory\Available MBytes'
Get-Counter '\PhysicalDisk(_Total)\Avg. Disk sec/Read', '\PhysicalDisk(_Total)\Avg. Disk sec/Write'
Get-NetAdapter | Select-Object Name, LinkSpeed, ReceiveBytes, SentBytes
```

**Real-World Example:** VM CPU consistently >90% — application had unbounded thread creation. Right-sized VM (upgraded from Standard_D2s_v3 to Standard_D4s_v3) and fixed application thread pooling.

**Common Failures:**
- VM size too small for workload
- Disk type insufficient IOPS/throughput (Standard HDD vs Premium SSD)
- Disk bursting exhausted (B-series burst credits depleted)
- Network bandwidth limit reached (VM size tier limit)
- NSG throttling (misconfigured)
- Guest OS issue (inside VM: memory leak, runaway process)
- Insufficient disk caching
- OS disk too small (event logs, temp files)
- Temp disk/ephemeral OS disk too small

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Slow VM? What metrics high? CPU? Memory? Disk? Network? |
| 2. Determine scope | One VM? Multiple? After scaling change? |
| 3. Check recent changes | VM SKU changed? Application deployed? Traffic increased? Disk type changed? |
| 4. Check monitoring | Azure Monitor metrics (CPU, Memory, Disk IOPS, Network). VM Insights. Application performance inside VM. Disk burst balance. |
| 5. Validate connectivity | VM reachable? VM network throughput vs. SKU limit? |
| 6. Check OS | Inside VM (via Bastion/serial console): performance counters, Task Manager, `Get-Counter`. VM size vs. workload requirements. |
| 7. Check dependencies | Disk type sufficient for IOPS? VM SKU has network bandwidth? Caching appropriate? |
| 8. Check logs | Azure Activity Log. VM Guest OS logs (inside VM). VM Insights logs. Azure Diagnostics logs. |
| 9. Identify root cause | VM too small? Disk IOPS limit? Memory leak? Network saturation? Disk burst exhausted? |
| 10. Implement fix | Resize VM (stop/deallocate → change SKU → start), change disk type (unsupported resize requires create new disk and attach), increase disk size (more IOPS/throughput proportional), enable accelerated networking, fix application issue, configure disk caching properly, enable VM Insights. |
| 11. Validate service | Performance within SLA. Metrics normalized. |
| 12. Monitor | Azure Monitor alerts for VM metrics. VM Insights for deep monitoring. |
| 13. Document RCA | Root cause, scaling decision, and capacity planning. |

**Interview Questions:**
- "An Azure VM is slow — how do you diagnose?"
- "How does VM size affect network bandwidth?"
- "What's disk burst and how do you monitor it?"
- "How do you resize a VM without downtime?" (can't — must deallocate for SKU change)

---

### 7.5 — Hybrid Connectivity Failure

**Concept:** Connectivity between on-premises and Azure (or between Azure VNets) fails.

**Architecture:** Hybrid connectivity options: VPN Gateway (IPsec/IKE), ExpressRoute (private fiber), VNet Peering (Azure backbone), Azure Bastion (jumpbox), Azure Private Link (Private Endpoints).

**Components:** VPN Gateway (VNet gateway), ExpressRoute circuit, BGP peering, local network gateway, VPN tunnel, VNet peering connection, DNS resolution (Azure Private DNS), traffic routing (UDR).

**Configuration:** VPN gateway type (Basic/HighPerformance/ZoneRedundant), VPN tunnel (IKEv2/IPsec), BGP ASNs, ExpressRoute circuits, peering configurations, UDR (User Defined Routes).

**Commands:**
```powershell
# Check VPN Gateway:
Get-AzVirtualNetworkGateway -ResourceGroupName RGName -Name GWName | Select-Object Name, GatewayType, VpnType, EnableBgp, GatewaySku, IpConfigurations
# Check VPN tunnel status:
Get-AzVirtualNetworkGatewayConnection -ResourceGroupName RGName -Name ConnectionName | Select-Object Name, ConnectionType, Status, PeerAddress, SharedKey, Ipv4PeerAddress, TunnelConnectionStatus
# Check connection shared key:
Get-AzVirtualNetworkGatewayConnection -ResourceGroupName RGName -Name ConnectionName | Select-Object -ExpandProperty SharedKey
# Reset VPN gateway (reset VPN to re-establish tunnel):
Restart-AzVirtualNetworkGateway -ResourceGroupName RGName -Name GWName
# Reset VPN tunnel:
Reset-AzVirtualNetworkGatewayConnection -ResourceGroupName RGName -Name ConnectionName -GatewayName GWName
# Check ExpressRoute:
Get-AzExpressRouteCircuit -ResourceGroupName RGName -Name ERCircuitName | Select-Object *
Get-AzExpressRouteCircuitPeering -ResourceGroupName RGName -CircuitName ERCircuitName -PeeringType AzurePrivatePeering | Select-Object *
# Check peering health (ExpressRoute):
Get-AzExpressRouteCircuitConnection -ResourceGroupName RGName -CircuitName ERCircuitName -PeeringName AzurePrivatePeering -Name PeerName | Select-Object *
# Check VNet Peering:
Get-AzVirtualNetworkPeering -ResourceGroupName RGName -VirtualNetworkName VNet1Name | Select-Object Name, PeeringState, RemoteVirtualNetwork, AllowVirtualNetworkAccess, AllowForwardedTraffic, AllowGatewayTransit
# Check local network gateway:
Get-AzLocalNetworkGateway -ResourceGroupName RGName -Name LNGName | Select-Object *
# Check BGP peers:
Get-AzVirtualNetworkGatewayBGPPeer -ResourceGroupName RGName -Name GWName | Select-Object *
# Test connectivity from Azure VM to on-prem:
Test-NetConnection -ComputerName OnPremIP -Port 80
Test-NetConnection -ComputerName OnPremIP -Port 443
# Test routing:
Test-NetConnection -ComputerName OnPremIP -Port 80 -Source VM_IP
# Check effective route:
Get-AzEffectiveRouteTable -ResourceGroupName RGName -NetworkInterfaceID $NIC.Id | Select-Object -ExpandProperty Routes
# Check DNS resolution from Azure:
Resolve-DnsName onprem.domain.com -Server 168.63.129.16  # Azure internal DNS
# Check Azure Private DNS:
Get-AzPrivateDnsZone -ResourceGroupName RGName -Name "privatelink.*.azure.com" | Select-Object *
# Network Watcher connection troubleshoot:
Test-AzNetworkWatcherConnectionIPFlow -ResourceGroupName RGName -NetworkWatcherName NWName -TargetResourceId $VM.Id -Direction Inbound -Protocol TCP -LocalPort 80 -RemotePort 8080 -RemoteIP OnPremIP
# Or from portal: Network Watcher → Connection Troubleshoot
```

**Real-World Example:** VPN tunnel dropped — BGP peering reset after maintenance. BGP session came back up automatically; for IKEv2 tunnels, tunnel reconnected after 10 seconds of no keepalives. Resolved by verifying BGP neighbor config.

**Common Failures:**
- VPN tunnel down (IKE time mismatch, shared key mismatch, gateway restart, idle timeout)
- BGP peering down (ASN mismatch, peer IP wrong, BGP disabled)
- ExpressRoute circuit down (provider-side issue, circuit deactivated, MAC locked)
- VNet peering broken (cross-tenant, DNS resolution failure, AllowForwardedTraffic disabled)
- UDR misconfigured (traffic not routed to gateway)
- NSG blocking traffic between VNets/on-prem
- DNS resolution failing (no Azure Private DNS, split-brain DNS)
- Gateway SKU too small (throughput exceeds capacity)
- Asymmetric routing (return traffic takes different path)
- NAT/translation issue

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | On-prem ↔ Azure failing? Specific protocol? All traffic or specific? Intermittent? |
| 2. Determine scope | One VPN tunnel? All tunnels? ExpressRoute? VNet peering? Cross-tenant? |
| 3. Check recent changes | Gateway restart, circuit maintenance, IP change, BGP config change, DNS change. |
| 4. Check monitoring | VPN tunnel status, BGP peer status, ExpressRoute circuit health, Network Watcher. |
| 5. Validate connectivity | `Test-NetConnection` on-prem to Azure and Azure to on-prem on specific ports. Check VPN tunnel status. |
| 6. Check OS/Config | `Get-AzVirtualNetworkGatewayConnection -Status`. BGP peers up? Tunnel state? `Get-AzVirtualNetworkGatewayBGPPeer`. UDR routing? NSG rules? |
| 7. Check dependencies | Gateway running? Circuit active? BGP configured correctly? Routes in place? DNS resolving? NSG allowing? |
| 8. Check logs | Activity Log. VPN Gateway diagnostics logs. BGP session logs. ExpressRoute logs. Network Watcher flow logs and connection troubleshoot. On-prem device logs. |
| 9. Identify root cause | Tunnel down? BGP peering issue? UDR missing? DNS broken? NSG blocking? Circuit provider issue? |
| 10. Implement fix | Restart VPN gateway (`Restart-AzVirtualNetworkGateway`), reset tunnel, verify BGP peering, fix UDR, fix DNS (Azure Private DNS, split-brain resolution), fix NSG rules, reset ExpressRoute circuit (if provider issue), fix asymmetric routing (check both directions), enable gateway diagnostics logging. |
| 11. Validate service | Tunnel up. BGP peers established. Connectivity from Azure to on-prem and vice versa on required ports. DNS resolution works in both directions. |
| 12. Monitor | Tunnel status monitoring (up/down alerts). BGP session monitoring. Network Watcher flow logs. Latency and throughput monitoring. |
| 13. Document RCA | Root cause, fix, and hybrid network architecture documentation with troubleshooting procedures. |

**Interview Questions:**
- "VPN tunnel down — how do you troubleshoot?"
- "What's the difference between VPN and ExpressRoute?"
- "How do you check routing in Azure?"
- "How does BGP help with VPN/ExpressRoute?"
- "How do you troubleshoot DNS resolution in hybrid environments?"

---

## 🔷 DOMAIN 8: BACKUP/DR (⭐⭐⭐⭐)

---

### 8.1 — Backup Failure

**Concept:** Backup job fails to complete, resulting in no backup or incomplete backup.

**Architecture:** Backup architecture: Backup agent (Windows Server Backup, Veeam, Commvault, Azure Backup/MARS agent, DPM) → backup server → storage (backup repository, offsite/cloud, tape). Backup types: Full, Incremental, Differential, Snapshot-based.

**Components:** Backup agent, backup catalog, backup repository (disk/tape/cloud), scheduling, VSS (Volume Shadow Copy Service), writer, network, storage encryption.

**Configuration:** Backup schedule, retention policy, backup type, VSS writers, repository location, encryption, compression, bandwidth throttling, backup window.

**Commands:**
```powershell
# Windows Server Backup:
wbadmin get versions  # list backup versions
wbadmin get status  # current backup status
wbadmin get disks  # disks available for backup
wbadmin get backup targets  # configured backup targets
wbadmin catalog  # list backup catalog entries
wbadmin checkhealth  # check WSB health (VSS writers, etc.)
wbadmin start backup -backupTarget:\\server\share -include:C: -allCritical -systemState -quiet  # trigger backup
# Check VSS writers:
vssadmin list writers
# Check VSS service:
Get-Service VSS
Get-Service SWPRemote
# Check specific writer status:
vssadmin list writers | Select-String -Pattern "State|Error"
# Check last backup time:
Get-WinEvent -LogName "Application" | Where-Object {$_.Id -eq 100 or $_.Id -eq 517} | Select-Object TimeCreated, Message -First 20
# For Veeam (if installed):
# Veeam.Backup.API PowerShell module
# Check via Veeam console or backup server
# For Azure Backup (MARS agent):
Get-AzRecoveryServicesVault -ResourceGroupName RGName -VaultName VaultName
Get-AzRecoveryServicesBackupJob -VaultId $Vault.ID -WorkloadType AzureIaasVM | Select-Object -First 20
Get-AzRecoveryServicesBackupProperty -VaultId $Vault.ID
# Check DPM:
Get-DPMServer | Select-Object *
# Check backup logs:
# WSB: C:\Windows\Logs\WindowsServerBackup\
# VSS: Event Viewer → Application → VSS events (Event ID 8193, 8194, 8195, 8196, 8200+)
Get-WinEvent -FilterHashtable @{LogName='Application'; Id=8193,8194,8195,8196,8200,8201,8202,8203,8204,8205,8206,8207,8208,8209,8210,8211,8212} -MaxEvents 30
# Check Application log for backup errors:
Get-WinEvent -FilterHashtable @{LogName='Application'; Id=517, 518} -MaxEvents 20
```

**Real-World Example:** Daily backup failed for 3 days — backup repository disk was full (C: drive where backup was stored hit 98%). Moved backup to network share and automated repository cleanup.

**Common Failures:**
- Repository storage full
- VSS writer hanging/failing
- Network timeout (backup window too short)
- Backup server offline
- Credentials expired (backup service account)
- Ransomware prevention blocking backup (Windows Defender/Conti)
- VSS service stopped
- Disk identifier changed (volume GUID changed after disk replace)
- Backup job conflict (multiple jobs at same time)
- Deduplication data corruption
- Snapshot creation timeout
- Backup application bug/corrupt catalog

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Backup job failed? Specific error? Backup not starting? Backup incomplete? |
| 2. Determine scope | One server? All servers? One backup job type? |
| 3. Check recent changes | Repository disk changed, backup server maintenance, VSS update, credential change, network change. |
| 4. Check monitoring | Backup job monitoring (Veeam dashboard, WSB status, SCOM, custom alerts). Backup success rate. |
| 5. Validate connectivity | Backup server reachable? Repository (network share) accessible? Network bandwidth sufficient? |
| 6. Check OS | `wbadmin get status`. `wbadmin get versions`. VSS writer status (`vssadmin list writers`). Backup application dashboard. |
| 7. Check dependencies | VSS service running? Repository space? Network throughput? Credentials valid? |
| 8. Check logs | WSB logs (`C:\Windows\Logs\WindowsServerBackup\`). VSS events (Event IDs 8193+). Application log (517, 518). Backup application logs. `wbadmin` output. |
| 9. Identify root cause | Repository full? VSS writer hanging? Network timeout? Credential expired? |
| 10. Implement fix | Free repository space, restart VSS service (`Restart-Service VSS`), fix credentials, increase backup window, optimize backup (exclude unnecessary data), change backup target, update backup application, fix disk identifier (`wbadmin delete backup` for old entries), check for ransomware blocking, configure Windows Defender exclusion for backup process. |
| 11. Validate service | Backup completes successfully. Recovery test successful (manual file restore). |
| 12. Monitor | Backup job success rate. Repository capacity. VSS writer health. |
| 13. Document RCA | Root cause, fix, and backup operational process (repository sizing, monitoring, testing). |

**Interview Questions:**
- "Backup failed — what's your approach?"
- "What are VSS writers and how do you check their health?"
- "How do you validate backup integrity?"
- "What's the difference between wbadmin checkhealth and wbadmin get status?"

---

### 8.2 — Restore Failure

**Concept:** Restoring data from backup fails or produces incomplete/corrupt data.

**Architecture:** Restore: Backup catalog → locate backup version → read backup data → reconstruct to target volume/machine → verify data integrity.

**Components:** Backup catalog, backup media (disk/tape/cloud), restore tool, target volume, system state components, AD (for AD restore), VSS (for point-in-time restore).

**Configuration:** Restore options (bare metal, system state, file/folder, VM), target location, alternate server restore, restore priority.

**Commands:**
```powershell
# List available backup versions for restore:
wbadmin get versions | Select-Object *
# List backup catalog:
wbadmin catalog 2>/dev/null | Select-Object *
# Restore system state:
wbadmin start systemstaterecovery -version:01/01/2024-12:00 -quiet
# Restore files:
wbadmin start recovery -version:01/01/2024-12:00 -itemType:File -items:C:\Important -restoreTarget:D:\Restore -quiet
# Restore to specific location (bare metal):
wbadmin start recovery -version:01/01/2024-12:00 -allCritical -restoreTarget:D:\ -quiet
# Bare metal restore from WinRE:
# Boot to WinRE → Troubleshoot → System Image Recovery → select backup
# In Veeam:
# Veeam.Backup.PowerShell module:
Get-VBRBackup | Select-Object Name, Id, LastBackupDate, Status
Get-VBRBackupSession -BackupId $Backup.Id | Select-Object *
# Restore from Veeam:
Restore-VBRBackup -BackupSessionId $Session.Id -RestoreTo OriginalOrNewLocation -TargetServer "Destination"
# For Azure Backup:
Get-AzRecoveryServicesBackupItem -VaultId $Vault.ID -WorkloadType AzureIaasVM -Name VMName | Select-Object *
Resume-AzRecoveryServicesBackupItemRestoreJob -Job $Job -TargetDiskStorageType Premium_LRS -StorageAccountName SAName
# Check backup integrity (catalog check):
# Windows: wbadmin starts backup -backupTarget:... -checkIntegrity ...
# Veeam: SureBackup / SureReplica functionality
# Event IDs for restore:
Get-WinEvent -FilterHashtable @{LogName='Application'; Id=517,518} -MaxEvents 30
# Application: Restore events from backup app logs
# VSS: 8193+ (for VSS-related restore issues)
```

**Real-World Example:** System state restore failed — DNS partition restore worked but SYSVOL was corrupt. Restored SYSVOL using Authoritative restore (`ntdsutil` → authoritative restore for AD objects), then non-authoritative for SYSVOL.

**Common Failures:**
- Backup version not available (retention expired)
- Backup catalog corrupted (can't find backups)
- Target volume insufficient space
- Restore to original server — name conflict
- AD restore: non-authoritative vs. authoritative confusion
- DIT (Directory Information) file in use (can't restore AD offline)
- Corrupt backup (backup completed but data corrupt)
- Restore tool version mismatch (newer backup, older restore tool)
- Permissions insufficient
- Network timeout (large restore over network)

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | Restore fails? Incomplete data? Error message? Data corrupt after restore? |
| 2. Determine scope | One server? AD restore? VM restore? Specific files? |
| 3. Check recent changes | Backup changed, storage replaced, tool updated, AD changes since backup. |
| 4. Check monitoring | Backup catalog integrity check. Backup verification (if configured). Storage health. |
| 5. Validate connectivity | Backup repository accessible? Backup media (tape/cloud) available? |
| 6. Check OS | `wbadmin catalog` — can it see backup versions? `wbadmin get versions`. Check disk space on target. |
| 7. Check dependencies | Target volume large enough? Backup catalog readable? Restore tool compatible? |
| 8. Check logs | Backup logs for original failure. Restore logs (`wbadmin` output). Application log (517, 518). VSS logs. Backup tool logs (Veeam, etc.). |
| 9. Identify root cause | Backup version missing? Catalog corrupt? Target full? Restore from corrupt backup? |
| 10. Implement fix | Select different backup version, restore to alternate location first (test), clear target space, repair backup catalog (`wbadmin catalog`), clean DIT (`ntdsutil` for AD), authoritative restore for specific AD objects (`ntdsutil` → `authoritative restore`), restore from different backup source, verify backup integrity first. |
| 11. Validate service | Restored data verified. Applications functional. For AD: `dcdiag`, `repadmin /replsummary`. For file restore: file checksums/integrity. |
| 12. Monitor | Post-restore monitoring for data integrity and application function. |
| 13. Document RCA | Root cause, restoration method used, and backup verification process improvement. |

**Interview Questions:**
- "How do you restore AD from backup?"
- "What's the difference between authoritative and non-authoritative restore?"
- "How do you verify backup integrity?"
- "Restore fails because catalog is corrupt — what do you do?"

---

### 8.3 — DR Failover

**Concept:** Disaster Recovery failover is triggered, and systems must transition from primary to DR site.

**Architecture:** DR architecture: Primary site → Replication (log shipping, storage replication, VM replication, database replication) → DR site → Failover orchestration (scripts, SCVMM, Azure Site Recovery, Zerto). RPO/RTO targets define acceptable data loss and downtime.

**Components:** Primary site, DR site, replication mechanism, DNS failover, load balancer, network replication, DR runbook/playbook, Orchestrator (SCVMM, ASR, Zerto, scripts).

**Configuration:** Replication frequency (RPO), failover policy (automatic/manual), DR site resources, DNS TTL, network configuration, testing schedule, failback procedure.

**Commands:**
```powershell
# Azure Site Recovery:
Get-AzRecoveryServicesVault -ResourceGroupName RGName -VaultName VaultName
Get-AzRecoveryServicesBackupProtectionContainer -VaultId $Vault.ID | Select-Object *
Get-AzRecoveryServicesBackupWorkloadProtectedItem -VaultId $Vault.ID -WorkloadType AzureIaasVM | Select-Object Name, ProtectionState, LastBackupTime, RecoveryPointCount
# Test DR failover (non-production):
Start-AzRecoveryServicesBackupFailoverTest -Item $Item -RestorePoint $RP -TargetResourceGroupName TestRG -TargetVMName TestVM -UseManagedDisk
# Failover:
Start-AzRecoveryServicesBackupFailover -Item $Item -RestorePoint $RP -TargetResourceGroupName DRRG -TargetVMName DRVM -UseManagedDisk
# Check ASR health:
Get-AzRecoveryServicesSite -VaultId $Vault.ID | Select-Object Name, FriendlyName, OperationalStatus
# For Hyper-V SCVMM DR:
# Check VM replication status:
Get-SCVMReplication -VMMServer VMMServer01 | Where-Object {$_.HealthState -ne "Ok"} | Select-Object VMName, HealthState, ReplicationPolicyName
# Check replication direction:
Get-SCVMReplication -VMMServer VMMServer01 | Select-Object VMName, SourceVMName, TargetVMName, HealthState, LastAppliedTime, LastReplicatedTime
# For on-premises replication (Storage Replica):
Get-SRPartnership | Select-Object Name, SourceComputerName, DestinationComputerName, Health, ReplicationStatus, LastReplicaFlushTime
# Check SR health:
Get-SRGroup | Select-Object Name, State, Health
Test-SRGroup -Name "SRGroup1"
# Database replication (SQL Always On):
Get-SqlAvailabilityGroup -Path "SQLSERVER:\Sql\PrimaryServer\Default\AvailabilityGroups\AG1" | Select-Object *
Get-SqlAvailabilityReplica | Select-Object Name, Role, SynchronizationState, SynchronizationHealth
# For DNS failover:
# Check TTL on DNS records:
Resolve-DnsName primary.domain.com | Select-Object Ttl, IPAddress
# Lower TTL (set well before DR):
# 300 seconds (5 min) recommended for DR scenarios
```

**Real-World Example:** DR failover tested during DR drill — VM failed over but DNS still pointed to primary. DNS TTL was 3600 seconds, clients cached DNS for 1 hour. Resolved by reducing TTL to 300 seconds 24 hours before production failover.

**Common Failures:**
- Replication lag exceeds RPO (data loss)
- DR site resources insufficient
- DNS not updated (clients still point to primary)
- DR network not configured (different subnets, VLANs)
- Certificate mismatch between sites
- DR VM not updated in last replication
- Licensing not at DR site (Windows, SQL, etc.)
- DR test not performed → unexpected failures
- Split-brain (both sites serving simultaneously)
- Orchestration tool misconfigured

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | DR failover triggered. What disaster? Planned failover? Actual disaster? What systems? |
| 2. Determine scope | Full DR site failover? Single VM? Single application? Multi-tier? |
| 3. Check recent changes | DR test done? Replication configuration changed? Network change? Certificate expired? |
| 4. Check monitoring | Replication health, RPO metrics, DR readiness dashboard, DR site capacity. |
| 5. Validate connectivity | DR site VMs have network? DNS resolution? Storage accessible? Network routes between sites? |
| 6. Check OS | Check replication health (`Get-SRPartnership`, `Get-SCVMReplication`, ASR health). DR site VM status. |
| 7. Check dependencies | DR site resources provisioned? DNS updates? Network routes? Certificates valid? |
| 8. Check logs | Replication logs. DR orchestration logs. Activity logs. Network logs. DNS logs for TTL. |
| 9. Identify root cause | Replication failed? DNS stale? Resources insufficient? Certificates expired? Network issue? |
| 10. Implement fix | Fix replication (resume, resync), update DNS to DR site IP (low TTL first), provision DR resources, update certificates, fix network routes, validate DR plan step by step. |
| 11. Validate service | All DR VMs running. Applications functional at DR site. DNS resolves to DR IPs. Clients can connect. Data loss within RPO. |
| 12. Monitor | DR site stability. Replication resume back to primary when ready (failback). |
| 13. Document RCA | Root cause, failover actions taken, DR plan improvements, and lessons learned. |

**Interview Questions:**
- "What's RPO and RTO?"
- "How do you validate DR readiness without impacting production?"
- "What happens if replication is behind during failover?"
- "How do you handle DNS during DR failover?"
- "What's the most common DR failure you've seen?"

---

### 8.4 — Recovery Validation Failure

**Concept:** After restore/failover, systems don't function correctly — data corrupt, services don't start, applications fail.

**Architecture:** Recovery validation: automated and manual checks to ensure restored/failed-over systems function correctly and data integrity is maintained.

**Components:** Application health checks, database integrity checks, AD integrity, DNS resolution, network connectivity, service status, data consistency checks, checksums, transaction log review.

**Configuration:** Health check scripts, application-specific validation queries, monitoring baselines, alert thresholds, backup verification settings.

**Commands:**
```powershell
# AD validation after restore:
dcdiag /v /e /c  # comprehensive DC validation
repadmin /replsummary  # replication status
nltest /dsgetdc:domain.com  # DC discovery
Get-ADDomainController -Filter * | Select-Object Name, OperationalStatus
# Check AD database integrity:
ntfsutil resource setintegritylevel C:\Windows\NTDS\ntds.dit (if applicable)
# On DC:
esentutl /g %SystemRoot%\NTDS\ntds.dit  # ESE database integrity check (if ESE errors)
# Database validation:
sqlcmd -S ServerName -Q "DBCC CHECKDB('DatabaseName')"
sqlcmd -S ServerName -Q "SELECT name, state_desc FROM sys.databases"
# Application health:
Test-NetConnection ServerName -Port 80
Test-NetConnection ServerName -Port 443
Invoke-WebRequest -Uri https://localhost/healthcheck -UseBasicParsing
curl http://localhost:8080/actuator/health  # Java Spring Boot health endpoint
# Service validation:
Get-Service | Where-Object {$_.Status -ne 'Running'} | Select-Object Name, Status, StartType
Get-WinEvent -LogName System | Where-Object {$_.Id -eq 7000 -or $_.Id -eq 7031} -MaxEvents 20  # service failures
# Network validation:
Test-NetConnection ServerName -Port 80,443,3389,53,139,445
Test-Connection ServerName -Count 10
# Disk validation:
chkdsk C: /f  # if recommended
Get-PhysicalDisk | Select-Object FriendlyName, HealthStatus, OperationalStatus
# Application-specific data validation:
# SQL: SELECT COUNT(*) FROM critical tables
# Compare row counts to expected values
# Email test (if mail server restored):
Send-MailMessage -From test@domain.com -To test@domain.com -Subject "DR Test" -Body "Testing" -SmtpServer localhost
# Custom recovery validation script:
$Tests = @{
    "Service Running" = (Get-Service Spooler).Status -eq 'Running'
    "AD Accessible" = [bool](Get-ADDomainController -Filter * -ErrorAction SilentlyContinue)
    "Disk Healthy" = (Get-PhysicalDisk).HealthStatus -eq 'Healthy'
    "DNS Resolves" = [bool](Resolve-DnsName domain.com -ErrorAction SilentlyContinue)
    "Web App Responds" = [bool](Invoke-WebRequest -Uri https://localhost -TimeoutSec 5 -UseBasicParsing -ErrorAction SilentlyContinue)
}
$Tests | Format-Table Name, @{N='Pass';E={$_.Value}}
```

**Real-World Example:** DR failover completed — VMs running, but SQL database had transaction log corruption because the replication hadn't flushed the log. Ran DBCC CHECKDB, found corruption, restored database from latest clean backup on DR site.

**Common Failures:**
- Application not configured for DR (connection strings point to primary)
- Database in suspect mode after restore
- AD objects missing or stale
- DNS records stale (pointing to primary site)
- Services configured for manual start but need to be running
- Application data not consistent (replication lag)
- Certificates not available at DR (TLS failures)
- Missing dependencies (shared services, load balancer, etc.)
- Testing not thorough enough

**Troubleshooting Steps:**

| Step | Action |
|------|--------|
| 1. Understand symptom | System running but not functioning? Application errors? Data inconsistency? |
| 2. Determine scope | Specific service? Application? Database? AD? Network? |
| 3. Check recent changes | Recovery just completed. Failover just happened. |
| 4. Check monitoring | Application health dashboards. Database integrity checks. Service status. |
| 5. Validate connectivity | Network connectivity restored? DNS correct? Applications reachable? |
| 6. Check OS | Service status. `dcdiag`. `DBCC CHECKDB`. Application logs. DNS resolution. |
| 7. Check dependencies | Application dependencies (DB, cache, message queue, AD) all functional? |
| 8. Check logs | Application logs, database error logs, System/Application event logs, AD replication logs, DNS logs. |
| 9. Identify root cause | Service not started? DB corrupt? DNS stale? Missing dependency? |
| 10. Implement fix | Start services, restore database from last clean checkpoint, update DNS, update application config, fix certificates, install missing dependencies on DR site. |
| 11. Validate service | All recovery validation checks pass. Application fully functional. RTO/RPO confirmed. |
| 12. Monitor | Full application monitoring at DR site. Compare to primary site baseline. |
| 13. Document RCA | Validation failures found and resolved. Update DR validation runbook. |

**Interview Questions:**
- "What's your recovery validation checklist?"
- "How do you validate database integrity after a restore?"
- "What do you do if recovery validation fails?"
- "How do you validate AD after a restore?"

---

# ✅ COMPLETE: PART 35 & 36 — Summary Coverage Matrix

| Domain | Scenario | Covered |
|--------|----------|---------|
| Windows | Server won't boot | ✅ |
| Windows | Server extremely slow | ✅ |
| Windows | Service keeps stopping | ✅ |
| Windows | High CPU | ✅ |
| Windows | High memory | ✅ |
| Windows | Disk 100% | ✅ |
| Windows | Disk full | ✅ |
| Windows | RDP unavailable | ✅ |
| Windows | Windows Update failure | ✅ |
| Windows | Blue screen | ✅ |
| AD | User cannot log in | ✅ |
| AD | Account repeatedly locks | ✅ |
| AD | Trust relationship broken | ✅ |
| AD | GPO not applying | ✅ |
| AD | Replication failure | ✅ |
| AD | New user not syncing | ✅ |
| AD | DC unavailable | ✅ |
| AD | SYSVOL problem | ✅ |
| AD | Kerberos failure | ✅ |
| DNS/DHCP | DNS resolution failure | ✅ |
| DNS/DHCP | Reverse lookup failure | ✅ |
| DNS/DHCP | AD clients wrong DNS | ✅ |
| DNS/DHCP | DHCP scope exhausted | ✅ |
| DNS/DHCP | Client receives APIPA | ✅ |
| VMware | VM inaccessible | ✅ |
| VMware | Host disconnected | ✅ |
| VMware | Datastore full | ✅ |
| VMware | vMotion failure | ✅ |
| VMware | Storage vMotion failure | ✅ |
| VMware | HA failover | ✅ |
| VMware | DRS issue | ✅ |
| VMware | Snapshot consolidation | ✅ |
| VMware | High CPU Ready | ✅ |
| VMware | High storage latency | ✅ |
| VMware | VM network failure | ✅ |
| Hyper-V | VM won't start | ✅ |
| Hyper-V | Live migration failure | ✅ |
| Hyper-V | Cluster node failure | ✅ |
| Hyper-V | CSV issue | ✅ |
| Hyper-V | Quorum problem | ✅ |
| PKI | Certificate expired | ✅ |
| PKI | Certificate chain failure | ✅ |
| PKI | Auto-enrollment failure | ✅ |
| PKI | TLS failure | ✅ |
| Azure | VM unreachable | ✅ |
| Azure | NSG blocking traffic | ✅ |
| Azure | Arc disconnected | ✅ |
| Azure | VM performance | ✅ |
| Azure | Hybrid connectivity | ✅ |
| Backup/DR | Backup failure | ✅ |
| Backup/DR | Restore failure | ✅ |
| Backup/DR | DR failover | ✅ |
| Backup/DR | Recovery validation | ✅ |

---

> **All scenarios follow the 13-step Interview-Focused Troubleshooting Framework.** Each includes Concept → Architecture → Components → Configuration → Commands → Real-World Example → Common Failures → Troubleshooting Steps → Root Cause → Fix → Validation → Interview Questions — structured for L3 enterprise operations credibility.