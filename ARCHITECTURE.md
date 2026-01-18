# 🏗️ SOC Lab Architecture

This document describes the **complete architecture** as envisioned, including both implemented and planned components. For implementation status, see [PROJECT_STATUS.md](PROJECT_STATUS.md).

---

## 📐 Complete System Architecture

![Complete SOC Architecture](images/labwazuh.png)

The architecture implements a comprehensive Security Operations Center with:
- **Detection Layer**: Endpoint (Wazuh) + Network (Security Onion)
- **Orchestration Layer**: SOAR automation with n8n
- **Enrichment Layer**: Threat intelligence via Cortex
- **Response Layer**: Analyst decision-making + automated endpoint isolation
- **Communication Layer**: Multi-channel notifications (Slack, Email)

---

## 🌐 Network Topology

### VirtualBox NAT Network Configuration

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VirtualBox NAT Network                            │
│                      10.2.15.0/24                                   │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │ Kali Linux   │  │ Windows 10   │  │ Security     │             │
│  │ 10.2.15.60   │─>│ 10.2.15.50   │  │ Onion        │             │
│  │ (Attacker)   │  │ +Wazuh Agent │  │ 10.2.15.10   │             │
│  └──────────────┘  └──────┬───────┘  └──────┬───────┘             │
│                           │                  │                      │
│                    ┌──────┴──────────────────┴─────┐               │
│                    │                                │               │
│  ┌──────────────┐  │  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Wazuh        │<─┘  │ n8n SOAR     │  │ TheHive + Cortex     │  │
│  │ Manager      │     │ 10.2.15.40   │─>│ 10.2.15.30           │  │
│  │ 10.2.15.20   │     │ :5678        │  │ :9000 (TheHive)      │  │
│  └──────────────┘     └──────────────┘  │ :9001 (Cortex)       │  │
│                                          └──────────────────────┘  │
│                                                                      │
│  External Services:                                                 │
│  ├─ Slack (Webhooks)                                               │
│  ├─ Email (SMTP)                                                   │
│  └─ Threat Intel APIs (VirusTotal, AbuseIPDB, MaxMind)            │
└─────────────────────────────────────────────────────────────────────┘

Host Machine:
├─ Dell Precision 8920 Tower
├─ Ubuntu 20.04
├─ VirtualBox 6.1
└─ 64GB RAM (recommended)
```

---

## 🔄 Data Flow Analysis

### Flow 1: Endpoint Detection Pipeline (✅ Implemented)

```
┌─────────────┐
│ Kali Linux  │ 1. Simulate attack (SSH brute force)
└──────┬──────┘
       │ Attack traffic
       ▼
┌─────────────┐
│ Windows 10  │ 2. Wazuh agent monitors system logs
│ +Agent      │    - Detects failed SSH login attempts
└──────┬──────┘    - Matches Rule 5710 (non-existent user)
       │ Sysmon/Windows logs
       ▼
┌─────────────┐
│ Wazuh       │ 3. Alert generation
│ Manager     │    - Rule: 5710 (level 5)
└──────┬──────┘    - Format: JSON with full alert data
       │ custom-w2n8n integration script
       │ HTTP POST
       ▼
┌─────────────┐
│ n8n         │ 4. Webhook receives alert
│ Webhook     │    - Endpoint: /webhook/wazuh-alert
└──────┬──────┘    - Method: POST
       │
       ▼
┌─────────────┐
│ n8n         │ 5. Data transformation
│ Edit Fields │    - Extract: data.srcip, data.srcuser
│ Node        │    - Format: rule.description, rule.level
└──────┬──────┘    - Prepare observables array
       │
       ├──────────────────┬─────────────────┐
       │                  │                 │
       ▼                  ▼                 ▼
┌─────────────┐    ┌─────────────┐   ┌─────────────┐
│ TheHive     │    │ Slack       │   │ Email       │
│ Create Case │    │ Notify      │   │ (Planned)   │
└──────┬──────┘    └─────────────┘   └─────────────┘
       │
       │ Case created with ID
       ▼
┌─────────────┐
│ TheHive     │ 6. Observable extraction
│ Alert #XYZ  │    - Type: IP
└─────────────┘    - Data: 8[.]8[.]8[.]8 (from data.srcip)
                   - Tags: wazuh, n8n, automated, level-5
```

**Status**: ✅ Fully functional

**Data Format at Each Stage**:

```json
// Wazuh Alert (Input)
{
  "rule": {
    "description": "sshd: Attempt to login using a non-existent user",
    "level": 5,
    "id": "5710"
  },
  "data": {
    "srcip": "8.8.8.8",
    "srcuser": "admin"
  },
  "agent": {
    "name": "wazuh-manager",
    "id": "000"
  }
}

// TheHive Alert (Output)
{
  "title": "Wazuh Alert 5710: sshd: Attempt to login using a non-existent user",
  "description": "Alert Level: 5\nSource IP: 8.8.8.8\nSource User: admin\nAgent: wazuh-manager",
  "severity": 2,
  "tags": ["wazuh", "n8n", "automated", "level-5"],
  "source": "n8n-automation",
  "type": "wazuh",
  "observables": [
    {
      "dataType": "ip",
      "data": "8.8.8.8",
      "message": "Source IP from Wazuh alert"
    }
  ]
}
```

---

### Flow 2: Threat Intelligence Enrichment (✅ Implemented - Manual)

```
┌─────────────┐
│ TheHive     │ 1. Alert contains observables
│ Case #2     │    - Observable: IP 8[.]8[.]8[.]8
└──────┬──────┘
       │ Analyst clicks "Run Analyzers"
       ▼
┌─────────────┐
│ Cortex      │ 2. Trigger analyzers
│ 10.2.15.30  │    - Job type: analyzer
└──────┬──────┘    - Data type: ip
       │
       ├───────────┬────────────┬──────────────┐
       │           │            │              │
       ▼           ▼            ▼              ▼
┌──────────┐ ┌──────────┐ ┌──────────┐  ┌──────────┐
│VirusTotal│ │AbuseIPDB │ │ MaxMind  │  │  Other   │
│ API      │ │ API      │ │  GeoIP   │  │Analyzers │
└────┬─────┘ └────┬─────┘ └────┬─────┘  └────┬─────┘
     │            │            │              │
     │            │            │              │
     ▼            ▼            ▼              ▼
┌─────────────────────────────────────────────────┐
│           Cortex Job Processing                  │
│  - Docker container per analyzer                 │
│  - Job directory: /home/thehive/cortex-jobs     │
│  - Results in JSON format                        │
└────────────────────┬────────────────────────────┘
                     │
                     ▼
            ┌─────────────┐
            │ TheHive     │ 3. Results displayed
            │ Observable  │    - Tags added
            │ Enriched    │    - TLP classification
            └─────────────┘    - Reputation data
```

**Status**: ✅ Manual trigger functional  
**Future**: 🚧 Automatic trigger from n8n planned

**Enrichment Results Example**:

```
IP: 8[.]8[.]8[.]8

VirusTotal_GetReport_3_1:
  Status: Success
  Result: Clean reputation
  
AbuseIPDB_1_1:
  Status: Success
  Abuse Confidence Score: 0
  Total Reports: 2
  Is Whitelisted: True
  Usage Type: Content Delivery Network
  
MaxMind_GeoIP_4_0:
  Status: Success
  Country: United States
  Continent: North America
  ISP: Google LLC
```

---

### Flow 3: Network Detection (🚧 Documented, Not Implemented)

```
┌─────────────┐
│ Network     │ All VM traffic mirrored
│ Traffic     │
└──────┬──────┘
       │ Packet capture
       ▼
┌─────────────┐
│ Security    │ 1. Network monitoring
│ Onion       │    - Zeek: Protocol analysis
│ 10.2.15.10  │    - Suricata: IDS rules
└──────┬──────┘    - Elasticsearch: Storage
       │
       ├──────────┬───────────┬──────────────┐
       │          │           │              │
       ▼          ▼           ▼              ▼
  ┌────────┐ ┌────────┐ ┌────────┐    ┌────────┐
  │ Port   │ │ DNS    │ │ C2     │    │ Data   │
  │ Scan   │ │ Tunnel │ │ Beacon │    │ Exfil  │
  └───┬────┘ └───┬────┘ └───┬────┘    └───┬────┘
      │          │          │              │
      └──────────┴──────────┴──────────────┘
                     │
                     ▼
            ┌─────────────┐
            │ Elasticsearch│ 2. Alert storage
            │ Index        │
            └──────┬───────┘
                   │ n8n scheduled query (planned)
                   ▼
            ┌─────────────┐
            │ n8n         │ 3. Parse network alerts
            │ (Planned)   │    - Filter by severity
            └──────┬──────┘    - Deduplicate
                   │
                   ▼
            ┌─────────────┐
            │ TheHive     │ 4. Create network case
            │ (Planned)   │    - Correlate with Wazuh
            └─────────────┘    - Unified timeline
```

**Status**: 🚧 Security Onion deployed, integration not implemented

**Planned Alert Types**:
- Port scanning (Nmap, Masscan)
- DNS tunneling detection
- C2 beaconing patterns
- Lateral movement (SMB, RDP)
- Data exfiltration (large uploads)

---

### Flow 4: Decision Tree & Interactive Response (🚧 Documented, Not Implemented)

```
            ┌─────────────┐
            │ Cortex      │ Enrichment complete
            │ Analysis    │
            │ Complete    │
            └──────┬──────┘
                   │
                   ▼
            ┌─────────────┐
            │ n8n         │ 1. Decision point
            │ Decision    │    - Evaluate threat score
            │ Node        │    - Check severity
            └──────┬──────┘
                   │
                   ▼
            ┌─────────────┐
            │ Slack       │ 2. Analyst prompt
            │ Message     │
            └─────────────┘

   Message Content:
   ┌──────────────────────────────────────────────┐
   │ 🚨 ALERT ENRICHMENT COMPLETE                 │
   │                                              │
   │ Case: #2 - SSH Brute Force Attempt          │
   │ IOC: 8.8.8.8                                 │
   │                                              │
   │ Enrichment Results:                          │
   │ ├─ Abuse Confidence: 0/100 (Low)            │
   │ ├─ Location: United States                  │
   │ ├─ VirusTotal: Clean                        │
   │ └─ Known CDN IP (Whitelisted)               │
   │                                              │
   │ ❓ DECISION REQUIRED:                        │
   │ Should we isolate the affected machine?     │
   │                                              │
   │ [✅ Isolate]  [❌ Ignore]  [⏱️ Investigate]  │
   └──────────────────────────────────────────────┘

                   │
         ┌─────────┴─────────┬─────────────┐
         │                   │             │
    [✅ Isolate]        [❌ Ignore]   [⏱️ Investigate]
         │                   │             │
         ▼                   ▼             ▼
    ┌─────────┐        ┌─────────┐   ┌─────────┐
    │ Flow 5  │        │ Flow 6  │   │ Update  │
    │ Active  │        │ Update  │   │ & Wait  │
    │ Response│        │ Case    │   │         │
    └─────────┘        └─────────┘   └─────────┘
```

**Status**: 🚧 Not implemented

**Implementation Requirements**:
- Slack interactive messages or slash commands
- n8n webhook to receive analyst response
- Timeout handling (auto-ignore after 30 minutes)
- State management for pending decisions

---

### Flow 5: Active Response - Endpoint Isolation (🚧 Documented, Not Implemented)

```
Analyst Response: [✅ Isolate]
         │
         ▼
┌─────────────┐
│ n8n         │ 1. Trigger active response
│ HTTP Request│    - Method: POST
└──────┬──────┘    - Endpoint: Wazuh API
       │           - Body: {"command": "firewall-drop", "agent_id": "001"}
       ▼
┌─────────────┐
│ Wazuh       │ 2. Execute active response
│ Manager     │    - Locate agent 001
└──────┬──────┘    - Queue command: firewall-drop
       │ Secure agent connection
       ▼
┌─────────────┐
│ Windows 10  │ 3. Agent receives command
│ Agent       │    - Execute: active-response.sh
└──────┬──────┘
       │ Script execution
       ▼
┌─────────────────────────────────────────────┐
│ Firewall Rules Applied:                      │
│                                              │
│ netsh advfirewall set allprofiles state on   │
│ netsh advfirewall firewall add rule          │
│   name="SOC_Block_All_Inbound"              │
│   dir=in action=block                        │
│ netsh advfirewall firewall add rule          │
│   name="SOC_Block_All_Outbound"             │
│   dir=out action=block                       │
│                                              │
│ Exceptions:                                  │
│ - Allow Wazuh agent (port 1514)             │
│ - Allow management IPs only                  │
└──────────────┬──────────────────────────────┘
               │
               ▼
        ┌─────────────┐
        │ Windows 10  │ 4. Machine isolated
        │ Status:     │    - No internet access
        │ ISOLATED    │    - No lateral movement
        └──────┬──────┘    - Wazuh agent still connected
               │
               ▼
        ┌─────────────┐
        │ n8n         │ 5. Verify isolation
        │ Poll Status │    - Check agent connectivity
        └──────┬──────┘    - Verify firewall rules
               │
               ├──────────────────┬──────────────────┐
               │                  │                  │
               ▼                  ▼                  ▼
        ┌─────────┐        ┌─────────┐        ┌─────────┐
        │ Slack   │        │ Email   │        │ TheHive │
        │ Notify  │        │ (Planned│        │ Update  │
        └─────────┘        └─────────┘        └─────────┘

Slack Notification:
┌────────────────────────────────────────────┐
│ ✅ ENDPOINT ISOLATION SUCCESSFUL            │
│                                            │
│ Machine: 10.2.15.50 (Windows 10)           │
│ Agent: 001                                 │
│ Time: 2026-01-16 14:30:00                  │
│                                            │
│ Actions Taken:                             │
│ ├─ Firewall: All traffic blocked           │
│ ├─ Exceptions: Wazuh agent only            │
│ └─ Status: ISOLATED                        │
│                                            │
│ Next Steps:                                │
│ - Forensic analysis required               │
│ - Review incident timeline                 │
│ - Prepare remediation plan                 │
│                                            │
│ 🔗 TheHive Case: [Link]                    │
│ 🔧 Restore Access: /restore-machine 001    │
└────────────────────────────────────────────┘
```

**Status**: 🚧 Configuration prepared, not tested

**Active Response Script** (`/var/ossec/active-response/bin/firewall-drop.sh`):
```bash
#!/bin/bash
# Wazuh Active Response Script
# Purpose: Isolate compromised endpoint

ACTION=$1
USER=$2
IP=$3

if [ "$ACTION" = "add" ]; then
    # Block all inbound traffic
    netsh advfirewall firewall add rule name="SOC_Block_All_Inbound" \
        dir=in action=block
    
    # Block all outbound traffic
    netsh advfirewall firewall add rule name="SOC_Block_All_Outbound" \
        dir=out action=block
    
    # Allow Wazuh agent
    netsh advfirewall firewall add rule name="SOC_Allow_Wazuh" \
        dir=out action=allow protocol=TCP remoteport=1514
        
    echo "Machine isolated successfully"
fi

if [ "$ACTION" = "delete" ]; then
    # Remove isolation rules
    netsh advfirewall firewall delete rule name="SOC_Block_All_Inbound"
    netsh advfirewall firewall delete rule name="SOC_Block_All_Outbound"
    netsh advfirewall firewall delete rule name="SOC_Allow_Wazuh"
    
    echo "Machine restored successfully"
fi
```

---

### Flow 6: Update Case Without Isolation (🚧 Documented, Not Implemented)

```
Analyst Response: [❌ Ignore] or [⏱️ Investigate]
         │
         ▼
┌─────────────┐
│ n8n         │ 1. Update TheHive case
│ HTTP PATCH  │    - Status: "InProgress"
└──────┬──────┘    - Custom field: decision="no_isolation"
       │           - Add comment: Analyst rationale
       ▼
┌─────────────┐
│ TheHive     │ 2. Case updated
│ Case #2     │
└──────┬──────┘
       │
       ├──────────────────┬──────────────────┐
       │                  │                  │
       ▼                  ▼                  ▼
┌─────────┐        ┌─────────┐        ┌─────────┐
│ Slack   │        │ Email   │        │ Task    │
│ Notify  │        │ (Planned│        │ Created │
└─────────┘        └─────────┘        └─────────┘

Slack Notification:
┌────────────────────────────────────────────┐
│ ⚠️ DECISION: NO ISOLATION                  │
│                                            │
│ Case: #2 - SSH Brute Force Attempt        │
│ Machine: 10.2.15.50 (Windows 10)           │
│                                            │
│ Analyst Decision:                          │
│ "Low confidence malicious activity.        │
│  Whitelisted CDN IP. Likely false positive.│
│  Monitoring continues."                    │
│                                            │
│ Actions Required:                          │
│ ├─ Continue monitoring for 24h             │
│ ├─ Review similar alerts                   │
│ └─ Update detection rules if needed        │
│                                            │
│ 🔗 TheHive Case: [Link]                    │
└────────────────────────────────────────────┘
```

**Status**: 🚧 Not implemented

---

## 📡 API Endpoints & Authentication

### Wazuh Manager API

```
Base URL: http://10.2.15.20:55000
Authentication: JWT token

Endpoints Used:
- POST /security/user/authenticate
  → Get JWT token
  
- GET /agents
  → List all agents
  
- PUT /active-response (Planned)
  → Trigger active response
  Body: {
    "command": "firewall-drop",
    "arguments": [],
    "agent_id": ["001"]
  }
```

### TheHive API

```
Base URL: http://10.2.15.30:9000/api
Authentication: Bearer token (API key)

Endpoints Used:
- POST /v1/alert
  → Create new alert
  Headers: Authorization: Bearer [API_KEY]
  Body: {title, description, severity, tags, observables}
  
- POST /v1/case
  → Create case from alert
  
- PATCH /v1/case/{caseId} (Planned)
  → Update case status
  Body: {status, customFields}
  
- POST /v1/case/{caseId}/task (Planned)
  → Create task for analyst
```

### Cortex API

```
Base URL: http://10.2.15.30:9001/api
Authentication: Bearer token (API key)

Endpoints Used:
- POST /analyzer/{analyzerId}/run
  → Run analyzer on observable
  Body: {data, dataType, tlp}
  
- GET /job/{jobId}
  → Get job status and results
  
- GET /job/{jobId}/report
  → Get full analysis report
```

### n8n Webhooks

```
Base URL: http://10.2.15.40:5678

Webhook Endpoints:
- POST /webhook/wazuh-alert
  → Receive Wazuh alerts
  
- POST /webhook/analyst-response (Planned)
  → Receive analyst decision from Slack
  
- POST /webhook/security-onion (Planned)
  → Receive Security Onion alerts
```

### Slack Incoming Webhooks

```
URL: https://hooks.slack.com/services/[WORKSPACE]/[CHANNEL]/[TOKEN]
Method: POST
Authentication: Embedded in webhook URL

Body Format:
{
  "text": "Alert summary",
  "attachments": [{
    "color": "danger",
    "fields": [
      {"title": "Source IP", "value": "8.8.8.8"},
      {"title": "Level", "value": "5"}
    ]
  }]
}
```

---

## 🔐 Security Considerations

### Network Isolation

```
Lab Environment Security:
├─ VirtualBox NAT Network (isolated from production)
├─ No bridged networking to host
├─ Firewall rules on host (if needed)
└─ No external access except threat intel APIs
```

### Credential Management

```
Security Practices:
├─ Unique passwords per service (20+ characters)
├─ API keys rotated regularly
├─ No credentials in Git repository
├─ .gitignore for sensitive files
└─ Environment variables for secrets
```

### Testing Safety

```
Safe Testing Practices:
├─ All attacks simulated (no real malware)
├─ Controlled environment only
├─ No testing on production systems
├─ Backup configurations before changes
└─ Document all test procedures
```

---

## 📊 Resource Requirements

### VM Resource Allocation

| VM | vCPU | RAM | Disk | Justification |
|----|------|-----|------|---------------|
| **Security Onion** | 4 | 16GB | 200GB | Heavy processing (ELK stack, Zeek, Suricata) |
| **Wazuh Manager** | 2 | 8GB | 100GB | Log processing, rule engine, integrations |
| **TheHive + Cortex** | 2 | 8GB | 100GB | Case management + Docker analyzers |
| **n8n** | 1 | 4GB | 50GB | Workflow engine, minimal resources |
| **Windows 10** | 2 | 4GB | 60GB | Victim endpoint, standard workstation |
| **Kali Linux** | 2 | 4GB | 50GB | Attack tools, penetration testing |
| **Total** | **13** | **44GB** | **560GB** | |

### Host Requirements

```
Minimum:
├─ CPU: 8 cores (16 threads)
├─ RAM: 48GB
├─ Disk: 600GB SSD
└─ OS: Ubuntu 20.04+

Recommended:
├─ CPU: 12+ cores
├─ RAM: 64GB
├─ Disk: 1TB NVMe SSD
└─ OS: Ubuntu 22.04
```

---

## 🔄 Data Retention & Performance

### Elasticsearch (Security Onion)

```
Retention Policy:
├─ Network logs: 7 days
├─ Alerts: 30 days
├─ PCAP: 3 days (or disk full)
└─ Indices rotated daily
```

### TheHive/Cortex

```
Storage:
├─ Cases: Indefinite (archive old)
├─ Observables: Linked to cases
├─ Cortex jobs: 90 days
└─ Attachments: Case-dependent
```

### Wazuh

```
Alert Storage:
├─ JSON alerts: /var/ossec/logs/alerts/
├─ Rotation: Daily
├─ Retention: 30 days
└─ Archived: Elasticsearch (optional)
```

---

## 📈 Performance Metrics

### Expected Throughput

```
Alert Processing:
├─ Wazuh: ~1000 EPS (Events Per Second)
├─ n8n: ~100 workflows/minute
├─ TheHive: ~50 cases/minute
└─ Cortex: ~10 enrichments/minute (API limits)
```

### Response Times

```
End-to-End Latency:
├─ Detection → Alert: < 10 seconds
├─ Alert → TheHive: < 5 seconds
├─ Enrichment: 10-30 seconds (API dependent)
└─ Notification: < 5 seconds
```

---

## 🎯 Scalability Considerations

### Horizontal Scaling Options

```
Future Expansion:
├─ Multiple Wazuh agents (100+)
├─ Wazuh cluster (manager + workers)
├─ TheHive cluster (multiple nodes)
├─ Cortex workers (distributed analyzers)
└─ n8n cluster (queue-based)
```

### Vertical Scaling

```
Resource Increase:
├─ Security Onion: Up to 32GB RAM for larger networks
├─ Wazuh: Up to 16GB RAM for more agents
└─ Elasticsearch: More disk for longer retention
```

---

## 🔧 Maintenance & Operations

### Backup Strategy

```
Critical Components:
├─ Wazuh: /var/ossec/etc/
├─ TheHive: /opt/thp/thehive/conf/ + database
├─ Cortex: /opt/cortex/conf/ + database
├─ n8n: Workflow exports (JSON)
└─ Configs: Git repository
```

### Update Procedures

```
Safe Update Process:
1. Snapshot VMs before update
2. Update one component at a time
3. Test integrations after each update
4. Rollback if issues occur
5. Document changes
```

---

**For detailed setup instructions, see [SETUP_GUIDE.md](SETUP_GUIDE.md)**  
**For current implementation status, see [PROJECT_STATUS.md](PROJECT_STATUS.md)**
