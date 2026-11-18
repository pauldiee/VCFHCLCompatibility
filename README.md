# HCLCompatibilityReport

A PowerShell script that inventories all hardware in a vSphere cluster and validates each device against the [Broadcom Hardware Compatibility Guide (HCL)](https://compatibilityguide.broadcom.com/) — outputting per-host CSVs, a cluster summary CSV, and a dark-mode HTML report with tabs per host and collapsible driver/firmware detail.

> **Original idea:** [Kabir Ali](https://whatkabirwrites.nl/)  
> **Author:** Paul van Dieen — [hollebollevsan.nl](https://www.hollebollevsan.nl)

---

## What it does

For every ESXi host in the target cluster the script collects:

| Category | Data collected |
|---|---|
| **CPU** | Model, vendor, socket count, core count |
| **NIC** | Vendor, model, PCI IDs (VID/DID/SVID/SDID), driver name, driver version, firmware version |
| **HBA / Storage** | Vendor, model, PCI IDs, driver name, driver version, firmware version |
| **GPU** | Vendor, model, PCI IDs, driver name (discrete GPUs only — iLO/BMC display chips are skipped) |

When `-TargetESXiVersion` is supplied, each unique device is looked up against the Broadcom Compatibility Guide API. Results show HCL status (Supported / Not Supported / Not Found) and the required driver + firmware combinations for the target release.

---

## Requirements

- **PowerCLI**: VCF.PowerCLI for vCenter 9, VMware.PowerCLI for vCenter 7 / 8 ([KB 428547](https://knowledge.broadcom.com/external/article/428547)). The script only needs `VMware.VimAutomation.Core`, which both provide.

```powershell
# vCenter 9 / VCF 9
Install-Module VCF.PowerCLI -Scope CurrentUser

# vCenter 7 / 8
Install-Module VMware.PowerCLI -Scope CurrentUser
```

- **PowerShell 5.1 or later** with VMware.PowerCLI; **PowerShell 7** with VCF.PowerCLI 9.x ([KB 409981](https://knowledge.broadcom.com/external/article/409981))

Setting up PowerShell 7 and VCF.PowerCLI on a jump host, with or without internet access, and keeping one PowerCLI version per host: [Jump Host: PowerShell 7 and VCF PowerCLI](https://vcf-planning.hollebollevsan.nl/docs/42-jump-host-powershell/).
- **Network access** to `compatibilityguide.broadcom.com` from the machine running the script
- vCenter credentials with at least read access to hosts and clusters

---

## Usage

### Inventory only (no HCL check)

```powershell
.\HCLCompatibilityReport.ps1 -vCenterServer vcenter.lab.local -ClusterName "Prod-Cluster-01"
```

### Inventory + HCL validation

```powershell
.\HCLCompatibilityReport.ps1 `
    -vCenterServer vcenter.lab.local `
    -ClusterName "Prod-Cluster-01" `
    -TargetESXiVersion "ESXi 9.0" `
    -OutputPath "C:\HCL_Reports"
```

### Scan all hosts in vCenter (no cluster filter)

```powershell
.\HCLCompatibilityReport.ps1 -vCenterServer vcenter.lab.local -TargetESXiVersion "ESXi 8.0 U3"
```

### Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| `-vCenterServer` | Yes | — | FQDN or IP of vCenter Server |
| `-ClusterName` | No | *(all hosts)* | vSphere cluster to inventory; omit to scan all hosts |
| `-TargetESXiVersion` | No | *(skip HCL check)* | Target ESXi release string, e.g. `"ESXi 9.0"` or `"ESXi 8.0 U3"` |
| `-OutputPath` | No | Current directory | Directory where output files are saved |
| `-Credential` | No | *(prompted)* | PSCredential for vCenter login |

---

## Output files

All files are written to `-OutputPath` (defaults to the current directory) and include a timestamp in the filename.

| File | Description |
|---|---|
| `HCL_{hostname}_{timestamp}.csv` | Per-host hardware inventory |
| `HCL_ClusterSummary_{timestamp}.csv` | All hosts combined |
| `HCL_Check_{version}_{timestamp}.csv` | HCL results with required driver/fw per device |
| `HCL_Check_{version}_{timestamp}.html` | Dark-mode HTML report (see below) |

### HTML report

The HTML report is self-contained (no external dependencies) and includes:

- **Summary cards** — total devices checked, supported (green), not supported (red), not found (amber), errors
- **Tab per host** — each tab shows the hostname with a colour-coded count of supported / not-supported / not-found devices
- **Device table** — type badge, vendor, model, PCI IDs, installed driver + version, installed firmware, HCL status badge
- **Collapsible driver/firmware detail** — click "▶ N combination(s)" to expand the required driver name, version, firmware, and type for the target release

---

## How HCL lookup works

The script queries the same REST API used by the Broadcom Compatibility Guide portal.

1. A session is initialised against `compatibilityguide.broadcom.com` to obtain cookies required by the detail endpoint.
2. Devices are deduplicated by `VID + DID` — the API is called once per unique combination across all hosts.
3. The search uses VID + DID only; SVID narrowing is done client-side to avoid excluding chip-vendor entries that carry the correct driver data.
4. If the first matching HCL entry has no driver/firmware data for the installed driver, the script walks through all candidate UUIDs for that VID/DID until it finds a match.
5. CPUs are looked up separately via keyword search (e.g. "Xeon Gold") and matched to the correct HCL series by model number prefix.

> **Note:** The BCG API is unofficial and undocumented. It may change without notice.

---

## Known limitations

- **Embedded/management GPUs** (iLO VGA, BMC display chips) are skipped. Only NVIDIA, AMD, and Intel discrete GPUs are checked against the HCL.
- **HCL results reflect unique VID+DID combinations.** Multiple identical cards in a cluster (e.g. 4 × ConnectX-4 Lx) share a single lookup result — the installed driver and firmware shown per row come from the inventory, but the HCL result is looked up once.
- **The BCG API requires outbound HTTPS** to `compatibilityguide.broadcom.com`. If your jump host has no direct internet access, HCL checking will fail gracefully (inventory CSVs are still produced).
- **Certificate warnings** are suppressed at the PowerCLI session level for lab/self-signed vCenter environments.

---

## Changelog

| Version | Changes |
|---|---|
| 1.4.2 | `Get-Help` shows the full help again: changelog moved out of the help block into its own comment block. |
| 1.4.1 | PowerCLI loading: import with `-ErrorAction Stop`, stop on mixed PowerCLI versions already loaded, clear error with loaded modules and jump-host guide link (#2). |
| 1.4.0 | HTML report: tab per host, collapsible driver/fw combos. Fixed HBA PCI lookup to exact DeviceName match. Removed SVID from BCG API filters (client-side only). Detail API walks all candidate UUIDs. Fixed PowerShell 5 array unwrapping bug. Skip embedded GPU/display chips. |
| 1.3.0 | CPU HCL check via BCG `cpu` program. CPU series matched by model number prefix. CPU rows rendered separately in HTML. |
| 1.2.1 | PowerShell 5 compatibility fixes. Fixed `viewDetails` to POST with query-string params. BCG session cookie initialisation. Driver/fw combos filtered to installed driver. |
| 1.2.0 | Fixed BCG API endpoint. Resolved `0x` prefix in PCI ID lookups. Added driver/firmware detail lookup. Added dark-mode HTML report. |
| 1.1.0 | Broadcom HCL check via BCG API. |
| 1.0.0 | Initial release — hardware inventory (CPU, NIC, HBA, GPU) from vCenter. |

---

## Related

- [Broadcom Compatibility Guide](https://compatibilityguide.broadcom.com/)
- [VMware PowerCLI documentation](https://developer.vmware.com/powercli)
