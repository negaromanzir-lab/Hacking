# honeypot

## Overview

Notes on honeypots: what the different types actually give you, how to stand one up safely, and how to read the data it produces. The defensive value is the focus here, because a honeypot earns its keep by turning "no evidence" into "confirmed evidence" and by showing which misconfiguration is actually reachable.

## What a Honeypot Is

A honeypot is a deliberately exposed system or service that exists to be attacked. It has no legitimate users, so every interaction is either a false positive, a misconfiguration, or a real attacker. That last property is what makes the data high-signal compared to ordinary logs, where almost everything is background noise.

The related term is a **honeypot token** or **honey credential**: a planted account, API key, or file that appears valid and is not. It is useful when you cannot expose a whole host, and it turns "we were probably never on the login page" into a checkable fact. Treat honey tokens as production change requests, though, not as something to drop into a live domain unannounced.

## Taxonomy

| Type | What the attacker gets | Data captured | Cost / risk |
| --- | --- | --- | --- |
| **Low-interaction** | An emulated surface only | Protocol commands, payloads, connection metadata | Low cost, very little fidelity |
| **High-interaction** | A real (isolated) machine | Full post-exploitation activity | Expensive, real containment risk |
| **Honeytoken** | Nothing, or a decoy file | Use of the credential or file | Cheap, no attack surface added |
| **Honeypot network** | Multiple emulated hosts | Scanning and lateral-movement patterns | Medium, needs segmentation |
| **Honeypot-as-a-service** | Vendor-hosted | Shared or isolated telemetry | Subscription, less raw control |

**Low-interaction** honeypots emulate the service rather than run it, so there is nothing real to exploit even if the attacker is thorough. That is the main reason to prefer them on a network you do not fully control.

**High-interaction** honeypots run a real OS, so they record behaviour that no emulation would predict. The cost is that you have now agreed to maintain a machine that is actively being compromised, and it must never be able to reach anything of value.

## Common Implementations

| Tool | Focus |
| --- | --- |
| **Cowrie** | SSH/Telnet emulator with a full shell emulation layer, file transfer, and command replay |
| **Dionaea** | Multi-protocol service emulation (FTP, SMB, HTTP, and more) driven by a template system |
| **Honeytables** | Backdoored web server and honeypot filesystem, for realistic web app responses |
| **Suricata / Zeek** | Network-level detection: scans, exploits, and C2 traffic patterns |
| **OpenCanary** | High-interaction Windows and Linux canaries, with per-user alerting |
| **Tarpit** | Deliberate slow-down of connections to exhaust attacker resources cheaply |

Cowrie and Dionaea cover the common case of internet background noise. Suricata and Zeek sit one layer lower and catch the scanning and exploitation traffic that precedes the connection a honeypot would see. For an internal lab, a plain VM with a fake service and full logging is often more informative than a packaged tool.

## Standing One Up Safely

The containment rules are the whole job. A honeypot that can reach production is a liability, not a sensor.

1. **Isolate it completely.** No route to production, no shared credentials, no trust relationship to the domain, no production DNS, no reachable data stores. Use a physically or logically separate VLAN with a deny-all policy toward everything that matters.
2. **Egress-control it.** If it does not need outbound, block outbound. Outbound C2 traffic is the main channel for exfiltration and for the attacker learning what they caught.
3. **Give it away freely.** Advertise it: an unauthenticated banner, an open port, a plausible hostname. A honeypot nobody finds produces nothing.
4. **Log centrally and immediately.** Ship logs off the host in near real time. Attacker-controlled machines are exactly the ones that get wiped, and post-compromise log loss is the normal case, not the exception.
5. **Monitor the honeypot itself.** Expect the logs to fill up, expect disk to run out, expect the clock to drift. A silent sensor is an unmonitored sensor.
6. **Reimage on a schedule.** The machine will be owned. Rebuild it from a known-good image and reset the honey credential after every confirmed interaction.

Never expose a honeypot on a network where someone else might mistake it for a production system. The label it publicly, and get the engagement scope in writing first.

## Reading the Data

What the logs actually give you, in rough order of usefulness:

- **Source addresses and ports.** Volume and rate tell you whether you are seeing commodity scanning or a targeted campaign. A single IP doing a full port sweep is a different event from thousands of IPs each trying one port.
- **The command sequence.** Credential attempts, then `uname`, then a download command, is a recognisable pattern, and a shell that downloads and executes a payload in one line tells you the tooling in use.
- **Payloads.** Dropped files, requested URLs, and command strings feed indicator lists and let you write detection rules.
- **Timing and TTPs.** Map what you observe onto [MITRE ATT&CK](https://attack.mitre.org/) so the activity can be compared against other sources and shared in a readable form.

The most important habit is deduplicating before you report. Background scanning produces the same handful of sources and payloads every day, and a report that does not separate them from a deliberate attempt overstates the result.

## Using It Alongside Other Signals

A honeypot is a complement, not a substitute.

- **IDS/IPS** for volumetric and signature-based network detection.
- **SIEM** for correlating the honeypot's events with authentication, DNS, and EDR telemetry from the real estate.
- **EDR** on the actual estate, which is where a real incident actually lands.
- **Threat intel** to check whether the sources and payloads you collected are already known.

The combination that works best: a honey credential shows an account was used, an EDR agent shows what it did, and the honeypot shows what the attacker does next against a target that was not real.

## Legal and Ethical Notes

- **Authorisation first.** Running a honeypot that is publicly reachable is active engagement, not passive monitoring. Have it in writing, and define the traffic you are permitted to collect and for how long you retain it.
- **Data handling.** Honeypot data is attacker-supplied and frequently contains third-party IP addresses, credentials, and personal data collected from elsewhere. Define retention and access rules before you start collecting.
- **Do not retaliate.** Do not scan back, do not exploit the access you gained, and do not publish an attacker's infrastructure without considering who else is exposed to that information.
- **No honeypots in shared or third-party infrastructure.** A honeypot in a provider's network can consume resources you are not paying for and can land you in an abuse report.
- **Protect the defenders.** Access to honeypot logs is itself a target. A honeypot's SSH key is a real credential and belongs in the same secret-management path as everything else.

## Related Material

- [`../About Linux/`](../About%20Linux/) and [`../About Windows/`](../About%20Windows/) for the platform fundamentals
- [`../About Networking/`](../About%20Networking/) for the protocol-level view
- [`../tryHackmeNotes-main/MITRE/`](../tryHackmeNotes-main/MITRE/) for mapping observations to techniques
- [`../tryHackmeNotes-main/Splunk/`](../tryHackmeNotes-main/Splunk/) for correlating the resulting events
- [`../tryHackmeNotes-main/Yara/`](../tryHackmeNotes-main/Yara/) for writing rules against collected samples
- [`../packet_sniffer-main/`](../packet_sniffer-main/) and [`../network_scanner-main/`](../network_scanner-main/) for the network-level view

## Planned Content

This directory is a placeholder. Intended additions:

- A Cowrie deployment walkthrough with a hardened `cowrie.conf` and systemd unit.
- A Dionaea template showing how to emulate an additional service.
- Example Suricata rules and a Zeek script for post-compromise behaviour.
- A honey credential deployment plan, including the abort criteria for rotating it.
- A log-processing script that tags events by ATT&CK technique and deduplicates sources.

## Contribution Guidelines

1. Record containment settings alongside any setup steps; a honeypot config without its firewall rules is incomplete.
2. Redact attacker infrastructure before committing captures.
3. Keep captured payloads out of the repository and store a hash plus a reference instead.
4. State the retention period for any logs a script writes.
5. Note the source of any rule or script you add, and test it before including it.
