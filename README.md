# Honeypot-TI-LLM-Splunk

Live honeypot threat intelligence and automated triage lab. A public-facing T-Pot honeypot captures real internet attack traffic, Splunk centralizes the telemetry, Canarytokens add a deception layer, and a Python classifier uses an LLM to map captured events to MITRE ATT&CK before writing enriched results back into Splunk.

**Stack:** T-Pot 24.04.1 · Splunk Enterprise · Ubuntu 22.04 · RamNode VPS · Docker · Canarytokens · Python · Anthropic Claude API · Splunk REST API · Splunk HEC · MITRE ATT&CK

**Full write-up:** [Honeypot_TI_Lab_Writeup.pdf](<Honeypot_TI_Lab_Writeup.pdf>) covers the architecture, troubleshooting, deception layer, and LLM enrichment pipeline in depth.

---

## Overview

My earlier labs focused on Active Directory telemetry, endpoint forensics, and cloud detection engineering. This project extends that work into live threat intelligence by exposing a controlled honeypot environment to the public internet and capturing real attacker behavior instead of only simulated activity.

The goal was to build a small SOC-style pipeline: collect raw attack telemetry, index it in Splunk, add high-confidence deception signals, and automate first-pass triage by enriching events with MITRE ATT&CK context.

## Architecture

```text
Internet Attackers
        ↓
T-Pot Honeypots + Honeytokens
        ↓
Splunk SIEM
        ↓
Python LLM Classifier
        ↓
MITRE ATT&CK-Enriched Events
        ↓
Splunk HEC
```

The environment runs on a single RamNode VPS. T-Pot provides the honeypot services, Splunk acts as the central SIEM, Canarytokens create deception alerts, and the Python classifier pulls recent Splunk events, classifies them with Claude, and writes the enriched output back into Splunk.

## 1. VPS Deployment

The lab started on a RamNode Ubuntu 22.04 VPS with 8GB RAM, 4 vCPUs, and 180GB storage. This was enough to begin the build, but not enough to run Splunk and the full honeypot stack reliably.

<p align="center"><img src="files/Screenshot%202026-07-12%20190708.png" width="720"></p>

After T-Pot installation, SSH was moved away from port `22` to port `64295`, allowing the honeypot services to occupy common ports as bait while keeping administration separate.

<p align="center"><img src="files/Screenshot%202026-07-12%20194337.png" width="720"></p>

## 2. T-Pot Honeypot Deployment

T-Pot 24.04.1 was deployed in Hive mode with multiple containerized honeypots, including Cowrie, Dionaea, Honeytrap, Conpot, Suricata, Sentrypeer, Adbhoney, Ciscoasa, and others.

<p align="center"><img src="files/Screenshot%202026-07-13%20020046.png" width="620"></p>

The honeypot began receiving traffic within hours of being exposed to the internet. The first dashboard capture showed 97 attacks across multiple honeypots and source countries.

<p align="center"><img src="files/Screenshot%202026-07-13%20020411.png" width="900"></p>

Most early activity was automated scanning and opportunistic probing, which matches what an unadvertised public VPS typically attracts.

## 3. Splunk SIEM Setup

Splunk Enterprise was installed directly on the VPS and used as the main SIEM for the project. During setup, Splunk’s default web port `8000` conflicted with Honeytrap, which was already listening on that port.

<p align="center"><img src="files/Screenshot%202026-07-13%20021958.png" width="760"></p>

I confirmed the conflict with `lsof` and moved Splunk Web to a different port instead of disrupting the honeypot service. This was one of the first reminders that honeypot platforms intentionally bind to common ports, so normal service assumptions do not always apply.

## 4. Log Ingestion into Splunk

T-Pot log sources were added into Splunk with custom sourcetypes so each honeypot could be searched separately.

Main sourcetypes:

```text
tpot:cowrie
tpot:honeytrap
tpot:dionaea
tpot:suricata
tpot:conpot
```

Ingestion was verified with:

```spl
index=main sourcetype=tpot:*
```

<p align="center"><img src="files/Screenshot%202026-07-13%20024255.png" width="900"></p>

A sourcetype breakdown confirmed that Splunk was indexing events from multiple honeypot sources.

```spl
index=main sourcetype=tpot:*
| stats count by sourcetype
```

<p align="center"><img src="files/Screenshot%202026-07-13%20031131.png" width="900"></p>

At this point, Splunk had indexed over 1,100 events across Conpot, Dionaea, and Honeytrap.

## 5. Resource Contention and Resize

Running Splunk, T-Pot, and the internal T-Pot stack on 8GB RAM caused instability. I disabled T-Pot’s redundant internal ELK stack since Splunk was serving as the SIEM, then resized the VPS to 16GB RAM, 8 vCPUs, and 260GB storage.

<p align="center"><img src="files/Screenshot%202026-07-13%20030615.png" width="720"></p>

The resize stabilized the lab and made it possible to run Splunk and the honeypot services together on one host.

## 6. Honeytoken Deception Layer

Raw honeypot logs are useful, but they include a lot of scanner noise. To add a higher-confidence signal, I deployed Canarytokens where an attacker or automated post-exploitation tool might look.

| Honeytoken | Purpose |
|---|---|
| Fake backup credentials | Detects interaction with a bait credential file |
| Fake AWS credentials | Detects attempted use of fake cloud keys |

<p align="center"><img src="files/Screenshot%202026-07-13%20165302.png" width="760"></p>

The token values should be redacted before publishing. Even fake credentials and Canarytoken URLs should not be exposed publicly because it can burn the bait and create false alerts.

## 7. LLM MITRE ATT&CK Classifier

The final stage was a Python script called `honeypot_llm_classifier.py`. The script pulls recent honeypot events from Splunk, sends them to Claude for classification, and returns structured fields:

```text
attack_category
attack_summary
mitre_technique_id
mitre_technique_name
severity
```

The classifier was first tested on sample events in dry-run mode.

<p align="center"><img src="files/Screenshot%202026-07-13%20163759.png" width="820"></p>

The test events mapped activity such as brute force, malware delivery, exploitation attempts, and reconnaissance to ATT&CK techniques including T1110.001, T1105, T1210, and T0846.

After test mode worked, I ran the script against live Splunk events.

<p align="center"><img src="files/Screenshot%202026-07-13%20164713.png" width="820"></p>

The live results classified real honeypot activity as brute force and reconnaissance with reasonable severity ratings, rather than treating every scanner hit as critical.

## 8. Writing Enriched Events Back to Splunk

The last step was closing the loop by writing the enriched results back into Splunk through the HTTP Event Collector.

```spl
index=main sourcetype=tpot:llm_enriched
```

<p align="center"><img src="files/Screenshot%202026-07-13%20164948.png" width="900"></p>

This produced analyst-ready events containing the original honeypot fields plus the LLM-generated ATT&CK mapping, summary, and severity.

## 9. Final Attack Volume

After additional runtime, the honeypot dashboard showed much higher attack volume.

<p align="center"><img src="files/Screenshot%202026-07-13%20181300.png" width="900"></p>

The dashboard reached approximately 39k total honeypot attacks, including:

| Honeypot | Events |
|---|---:|
| Honeytrap | 19k |
| Dionaea | 12k |
| Sentrypeer | 3k |
| Cowrie | 2k |
| Ciscoasa | 533 |
| RDPHoneypot | 476 |
| Conpot | 381 |
| Adbhoney | 179 |
| Tanner | 164 |
| Miniprint | 125 |

This reinforced the core lesson of the project: exposed public infrastructure is found quickly, even when it is never advertised.

## Key Decisions & Lessons

- **Splunk over T-Pot’s internal ELK stack.** I used Splunk as the system of record because it better matches SOC workflows and connects to my previous detection engineering work.
- **Resize instead of forcing 8GB to work.** Splunk plus a full honeypot suite needed more memory. Moving to 16GB RAM made the lab stable.
- **Port conflicts are expected in honeypot labs.** Honeytrap occupying port `8000` was not a random issue. It was part of the honeypot attack surface.
- **Honeytokens create higher-confidence alerts.** Raw honeypot hits show scanning volume, but token interaction suggests deeper attacker behavior.
- **LLM enrichment is useful when bounded.** The classifier used structured output and ATT&CK mappings to turn raw events into analyst-ready summaries.
- **Secrets review matters.** Screenshots, scripts, API keys, HEC tokens, and honeytoken values need to be checked before publishing.

## What This Demonstrates

End-to-end threat intelligence and automated incident response: live honeypot deployment, real attacker telemetry collection, Splunk SIEM ingestion, deception-based alerting, Python automation, LLM-assisted triage, and MITRE ATT&CK enrichment.

The result is a small but complete SOC-style workflow: raw internet attacks come in, Splunk indexes them, the classifier enriches them, and the final events are written back as structured intelligence.
