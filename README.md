# Threat Intelligence & Automated Incident Response Lab

## Live Honeypot, Splunk SIEM, Honeytokens, and LLM-Driven MITRE ATT&CK Classification

This project is an end-to-end threat intelligence and automated incident response lab built around a live internet-facing honeypot. The goal was to capture real attacker behavior, centralize the telemetry in Splunk, add a deception layer using honeytokens, and automate first-pass triage by mapping captured activity to the MITRE ATT&CK framework with an LLM-powered classifier.

Unlike simulated-only labs, this environment was exposed to the public internet and captured real scanning, probing, and attack traffic from external sources.

---

## Project Overview

The lab was built on a cloud VPS running a T-Pot honeypot suite and a self-hosted Splunk instance. Honeypot logs were ingested into Splunk using separate sourcetypes for each data source. A deception layer was added with Canarytokens, and a Python script was developed to pull recent attack events from Splunk, classify them with an LLM, and write the enriched results back into Splunk through the HTTP Event Collector.

The final result is a small-scale SOC-style pipeline:

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
