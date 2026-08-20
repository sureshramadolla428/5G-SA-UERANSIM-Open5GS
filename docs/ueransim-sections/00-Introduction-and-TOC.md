# UERANSIM + Open5GS — Setup Guide
> Part of the complete guide. See `UERANSIM-OPEN5GS-SETUP-GUIDE.md` for the full document.

# UERANSIM + Open5GS 5G Core — Complete Setup, Test & Stress Guide

> **Audience:** Linux administrators building a software-only 5G lab  
> **Target OS:** Ubuntu 22.04+ inside VMware  
> **Scope:** Build UERANSIM, configure gNB/UE for Open5GS, verify, capture traffic, multi-UE load, break-test

---

## About This Guide

This guide walks through **UERANSIM** — open-source 5G gNB and UE simulator — connected to **Open5GS** 5G Core. It covers source builds, configuration aligned with upstream `config/open5gs-gnb.yaml` and `config/open5gs-ue.yaml`, signaling flows, PCAP analysis, and fault injection.

**Pre-built configs in this project:**

- `private-5g/ueransim/config/open5gs/open5gs-gnb.yaml`
- `private-5g/ueransim/config/open5gs/open5gs-ue.yaml`

**Default PLMN:** MCC **999** / MNC **70** (Open5GS + UERANSIM standard test network)

---

## Table of Contents

1. [What Is UERANSIM and Why Use It](#1-what-is-ueransim-and-why-use-it)
2. [Complete 5G Signaling Flow Diagrams](#2-complete-5g-signaling-flow-diagrams)
3. [Prerequisites & System Requirements](#3-prerequisites--system-requirements)
4. [Install All Build Dependencies](#4-install-all-build-dependencies)
5. [Clone and Build UERANSIM](#5-clone-and-build-ueransim)
6. [Complete gNB Configuration File](#6-complete-gnb-configuration-file)
7. [Complete UE Configuration File](#7-complete-ue-configuration-file)
8. [Provision the Subscriber in Open5GS](#8-provision-the-subscriber-in-open5gs)
9. [Start and Verify the gNB](#9-start-and-verify-the-gnb)
10. [Start and Verify the UE](#10-start-and-verify-the-ue)
11. [End-to-End Connectivity Tests](#11-end-to-end-connectivity-tests)
12. [PCAP Capture for gNB and UE Traffic](#12-pcap-capture-for-gnb-and-ue-traffic)
13. [Multiple UE Simulation](#13-multiple-ue-simulation)
14. [Break Testing & Stress Testing](#14-break-testing--stress-testing)
15. [Troubleshooting Common Issues](#15-troubleshooting-common-issues)

---
