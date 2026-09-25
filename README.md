# Azure Network Traffic & Network Security Groups

### Wireshark · Windows & Linux · Protocol troubleshooting

A hands-on networking lab using Windows and Ubuntu virtual machines in Microsoft Azure to explore packet capture, network protocols, and traffic controls.

[Portfolio](https://github.com/justinmccuff) · [Repository](https://github.com/justinmccuff/network-protocols)

> **Documentation status:** The original walkthrough covers VM setup and ICMP, DHCP, DNS, and RDP observations. This revision adds explicit capture instructions, safety guidance, and proposed NSG, SSH, and HTTP/S exercises. The revised procedure has not been executed against a live lab. Expected observations are not verified results. Original screenshot links are retained below but have not been independently inspected or validated.

## Objectives

- Capture traffic on a Windows VM using Wireshark.
- Distinguish ICMP, DHCP, DNS, RDP, SSH, HTTP, and TLS traffic.
- Compare successful connectivity with an intentionally blocked ICMP test.
- Explain the limits of endpoint capture and encrypted-traffic inspection.
- Collect sanitized evidence that another learner can reproduce.

## Software & Lab Design

| Component | Purpose |
| :--- | :--- |
| Microsoft Azure | VM hosting, virtual networking, and NSGs |
| Windows VM | Wireshark capture workstation and test client |
| Ubuntu VM | Private-address ping and SSH target |
| Wireshark + Npcap | Windows packet capture and analysis |
| PowerShell / Command Prompt | `ping`, `ipconfig`, `nslookup`, `ssh`, and `curl.exe` |
| OpenSSH Server on Ubuntu | SSH endpoint for the proposed exercise |

The original lab used Windows 10 21H2 and Ubuntu Server 20.04. Those are historical versions, not recommendations for a new deployment. Use supported, patched, appropriately licensed images and verify their support status before deployment.

### Illustrative topology

```text
Dedicated lab resource group
└── VNet: 10.20.0.0/16
    └── Subnet: 10.20.1.0/24
        ├── WIN-LAB     10.20.1.4  → Wireshark and test commands
        └── LINUX-LAB   10.20.1.5  → ICMP and SSH target
                                    NSG associated with Ubuntu NIC
```

All names and addresses above are examples. Substitute the actual addresses assigned in Azure. A resource group's location does not require all contained resources to use that region. For this simple lab, place both VMs in the same VNet and region; confirm subnet membership and routing rather than relying on the resource group alone.

## 1. Prepare a Safe Environment

1. Create a dedicated lab resource group, VNet, and subnet, then deploy both VMs.
2. Record private addresses and inspect any NSGs on both NICs and the subnet. Both subnet and NIC rules can affect connectivity.
3. Prefer Bastion or VPN/private access for administration. If temporary public RDP is necessary, restrict TCP 3389 to your trusted public IP, retain Network Level Authentication, and remove access when finished. Do not expose SSH or RDP to the entire internet.
4. Use SSH keys for Ubuntu access and keep keys and passwords out of the repository.
5. Keep guest firewalls enabled; add narrowly scoped rules where needed.
6. Confirm explicit outbound connectivity if installing packages or testing internet destinations. Do not assume a new Azure deployment automatically has internet egress. Any NAT, Bastion, disks, and other resources can incur charges.
7. Set a budget alert and a cleanup plan.

Capture only traffic on systems you own or are authorized to test. Packet captures can expose hostnames, usernames, DNS queries, and application data.

## 2. Install Wireshark & Start Capture

1. Download Wireshark from [wireshark.org](https://www.wireshark.org/download.html).
2. Install the Windows capture driver, Npcap, when prompted. Follow your organization's software policy.
3. Open Wireshark and select the active Ethernet interface associated with the Windows VM's private address. Confirm the interface is receiving packets.
4. Start the capture before running a test.
5. Enter the filters below in Wireshark's **display filter** bar.

A display filter hides unrelated packets in an existing capture; it does not prevent those packets from being recorded. Capture filters use different syntax. This lab uses display filters throughout.

An endpoint capture shows traffic visible to that VM's interface, not every flow in the Azure VNet. Promiscuous mode does not turn a VM into a VNet-wide traffic monitor.

## 3. ICMP — Test Private Connectivity

On Windows, replace the address with your Ubuntu VM's private IP:

```powershell
ping -4 -n 4 10.20.1.5
```

**Display filter:**

```text
icmp && ip.addr == 10.20.1.5
```

**Expected:** Echo Requests from Windows and Echo Replies from Ubuntu, if routing, NSGs, and host firewalls permit them. Record source/destination IPs, ICMP type, sequence number, and reply behavior.

A failed ping does not prove the target is offline. Public destinations can block ICMP, and resolving a hostname is a separate DNS operation.

## 4. NSG — Compare Allow, Deny & Recovery

**New exercise: not demonstrated in the original README.** Perform only on the isolated lab NIC/NSG.

1. Establish a successful private ping baseline first. Save a short capture and command output.
2. Locate the NSG attached to the Ubuntu NIC. If creating one for the exercise, associate it with that NIC and preserve your management-access rules.
3. Review effective security rules, including any subnet NSG and higher-priority rules.
4. Add a narrowly scoped inbound deny rule:

| Setting | Example |
| :--- | :--- |
| Name | `Deny-ICMP-From-WIN-LAB` |
| Priority | `200`, only if unused and ahead of any matching allow rule |
| Source | Windows private IP, `10.20.1.4/32` |
| Source port | Any (`*`) |
| Destination | Ubuntu private IP, `10.20.1.5/32` |
| Destination port | Any (`*`) |
| Protocol | ICMP |
| Action | Deny |

Lower numeric priority is evaluated first. ICMP does not use TCP/UDP port numbers; the port fields remain Any.

5. Allow time for the rule to apply, stop the old ping, and run a fresh short test. NSGs are stateful; an established tracked flow can persist after a rule change. If results remain unchanged, allow the previous flow to expire and retry rather than concluding the rule failed.
6. Compare the capture and command output with the baseline. Expected after the deny applies: outgoing requests without successful echo replies. Timeouts alone do not uniquely identify the NSG as the cause.
7. Remove only the temporary deny rule, wait for propagation, and run a fresh test to confirm recovery.

**Save evidence:** Rule settings and association, effective rules, baseline output, denied-test output, and restored connectivity. Do not block all protocols or change RDP/SSH management rules for this exercise.

## 5. DNS — Inspect a Lookup

Run on Windows:

```powershell
nslookup example.com
```

**Display filter:**

```text
dns
```

**Expected:** A DNS query and response if a conventional DNS exchange is visible. Inspect the query name, record type, resolver address, response code, and returned records. DNS commonly uses UDP 53 but can use TCP 53 as well.

Browser traffic may use encrypted DNS; it will not necessarily appear under this filter. `nslookup` is a more controlled exercise than opening a browser and assuming a fresh visible lookup occurs.

## 6. DHCP — Observe Lease Renewal Carefully

Run this optional exercise only when you have an alternate recovery path if remote access is interrupted. Start capture first, then use:

```powershell
ipconfig /renew
```

**Display filter:**

```text
udp.port == 67 || udp.port == 68
```

**Expected:** A renewal may show DHCPREQUEST and DHCPACK. A renewal does not guarantee a new IP address or the full Discover → Offer → Request → Acknowledge exchange. Packet visibility and timing can vary in Azure.

Do not run `ipconfig /release` on a remotely managed VM for this exercise; it can interrupt connectivity. Do not manually change guest IP settings to force a capture. Record exactly what was visible rather than assuming all four DHCP messages occurred.

## 7. RDP — Observe the Management Session

While capturing on the Windows VM, interact briefly with the desktop through the authorized remote-access path.

**Display filter:**

```text
tcp.port == 3389 || udp.port == 3389
```

**Expected:** Traffic on the default RDP port if that transport is present at the capture point. RDP can use TCP and UDP; gateways, alternate ports, and connection paths affect what is visible.

Inspect endpoint addresses, transport, packet timing, and traffic volume. Encrypted RDP does not expose readable screen contents or keystrokes in a normal capture. A port match is an indication, not definitive proof of application identity.

## 8. SSH — Inspect an Encrypted Session

**New exercise: not demonstrated in the original README.**

1. Confirm that Ubuntu has OpenSSH Server installed and running. From its console, `systemctl status ssh` can help check the service.
2. Permit TCP 22 only from the Windows VM's private IP through the relevant NSGs and guest firewall.
3. On Windows, confirm an OpenSSH client is available. Start capture, then connect using your actual username, private IP, and key:

```powershell
ssh -i "C:\path\to\lab_key" labuser@10.20.1.5
```

Verify the server host-key fingerprint through a trusted path before accepting a new key. Do not bypass host-key checking.

**Display filter:**

```text
tcp.port == 22 && ip.addr == 10.20.1.5
```

Run a harmless command such as `hostname`, then `exit`.

**Expected:** TCP establishment, SSH identification/key exchange, and encrypted session traffic. Ordinary packet capture cannot reveal encrypted commands or passwords.

## 9. HTTP & HTTPS — Compare Visibility

**New exercise: not demonstrated in the original README.** Requires working DNS and permitted internet egress.

Run from Windows while capturing:

```powershell
curl.exe --http1.1 --head http://example.com
curl.exe --http1.1 --head https://example.com
```

**Display filters:**

```text
http
```

```text
tls || tcp.port == 443
```

**Expected:** The HTTP request and response headers may be readable, including a redirect if returned. HTTPS normally shows TLS traffic rather than readable HTTP headers in Wireshark. The HTTP/1.1 option avoids a browser's HTTP/3/QUIC path for this controlled test; browser HTTPS may instead use UDP 443.

Do not transmit credentials over HTTP. Do not disable certificate validation or attempt to bypass encryption. A proxy or managed inspection service can change the observed traffic; document that if present.

## Filter Quick Reference

| Test | Wireshark display filter |
| :--- | :--- |
| IPv4 ping | `icmp` |
| DNS | `dns` |
| DHCPv4 | `udp.port == 67 || udp.port == 68` |
| Default RDP transports | `tcp.port == 3389 || udp.port == 3389` |
| Default SSH port | `tcp.port == 22` |
| Decoded HTTP | `http` |
| TLS / TCP 443 | `tls || tcp.port == 443` |
| A specific IPv4 endpoint | `ip.addr == 10.20.1.5` |

These are display filters, not capture filters. Port-based matches alone do not establish protocol identity.

## Results — Complete After Rerunning

| Test | Evidence to collect | Current status |
| :--- | :--- | :--- |
| ICMP | Requests, replies, command output | Original walkthrough present; revision not verified |
| NSG deny/recovery | Rule details and before/during/after results | Proposed extension |
| DNS | Query and response details | Original walkthrough present; revision not verified |
| DHCP renewal | Messages actually observed | Original walkthrough present; revision not verified |
| RDP | Transport and endpoint observations | Original walkthrough present; revision not verified |
| SSH | Connection and encryption observations | Proposed extension |
| HTTP/S | Plaintext versus TLS comparison | Proposed extension |

Capture short sessions, stop recording promptly, and sanitize screenshots before sharing. Keep raw `.pcapng` files private unless you have reviewed them carefully; a display filter does not remove hidden packets from a saved capture.

## Original Screenshot References

The original URLs use the older `azure-network-protocols` repository path. They are preserved as supplied rather than guessing replacement attachment URLs. If they no longer load, re-upload reviewed screenshots to this repository and update the links.

| Original section | Screenshot links |
| :--- | :--- |
| Resource setup | [1](https://github.com/justinmccuff/azure-network-protocols/assets/143865133/a4344f8f-1f11-4894-ae19-38967276734f) · [2](https://github.com/justinmccuff/azure-network-protocols/assets/143865133/3a5a91a7-3820-4f42-b311-73329b8f947e) · [3](https://github.com/justinmccuff/azure-network-protocols/assets/143865133/7fd697ff-56c7-42b2-a780-d4148a5d40ab) |
| VM creation | [4](https://github.com/justinmccuff/azure-network-protocols/assets/143865133/61ce7de1-5b48-467e-9b77-e4f8fcc9d31c) · [5](https://github.com/justinmccuff/azure-network-protocols/assets/143865133/61985342-2f64-4295-8b11-ae28b40ce632) |
| Remote access and setup | [6](https://github.com/justinmccuff/azure-network-protocols/assets/143865133/16afc86f-f8a4-4915-b1ac-d2e66c1780b9) · [7](https://github.com/justinmccuff/azure-network-protocols/assets/143865133/0c1bea19-bd04-4fc3-8532-31c11c4de1ce) |
| ICMP | [8](https://github.com/justinmccuff/azure-network-protocols/assets/143865133/f76c9ed2-2e8d-46b3-9504-d6b8255ccac4) |
| DHCP | [9](https://github.com/justinmccuff/azure-network-protocols/assets/143865133/91b7ffb5-3b70-4be4-b985-4a66ecbdf2de) |
| DNS | [10](https://github.com/justinmccuff/azure-network-protocols/assets/143865133/afd52767-8659-4436-8a17-44ec25eeefe6) |
| RDP | [11](https://github.com/justinmccuff/azure-network-protocols/assets/143865133/ac7246e5-8302-4fa2-a3b1-c9e10889de43) · [12](https://github.com/justinmccuff/azure-network-protocols/assets/143865133/8bf6201f-4fa7-444d-b065-8083cc0be0ef) |

## Troubleshooting & Cleanup

- **No packets:** Verify the selected interface, Npcap installation, capture permissions, and active capture. Clear the display filter to check for other traffic.
- **Ping fails:** Check the private destination address, routing, effective NSG rules, and guest firewall. Do not disable all security controls.
- **SSH fails:** Check OpenSSH Server, TCP 22 reachability, username, key, and key permissions.
- **No DNS packets:** Confirm a fresh query occurred and that the resolver exchange is visible at this interface.
- **No internet traffic:** Check DNS, routes, explicit outbound connectivity, and proxy settings.
- **NSG test unchanged:** Verify rule association, priority, exact endpoints, propagation, and stateful flow behavior.

After exporting sanitized evidence, remove temporary rules and delete only the dedicated lab resources you intend to remove. Deallocation alone does not eliminate disk and other resource charges. Verify cleanup in Azure.
