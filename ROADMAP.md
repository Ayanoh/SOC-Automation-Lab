# 🗺️ Project Roadmap

This document outlines the planned implementation of features that are currently documented but not yet implemented.

---

## 📊 Implementation Status Overview

```
✅ Phase 1: Core Detection Pipeline          [100% Complete]
✅ Phase 2: SOAR Orchestration               [100% Complete]
✅ Phase 3: Threat Intelligence              [100% Complete]
✅ Phase 4: Basic Notifications              [85% Complete]
✅ Phase 5: Advanced Automation              [60% Complete]
🚧 Phase 6: Network Monitoring Integration   [0% Complete]
🚧 Phase 7: Active Response                  [0% Complete]
```

---

## 🎯 Priority 1: Critical Features (Next 2-3 weeks)

### 1. Fix n8n JSONPath Expressions
**Status**: 🔴 High Priority  
**Estimated Time**: 30 minutes  
**Dependencies**: None

**Current Issue**:
- Observables sometimes created with "N/A" values
- JSONPath expressions not always matching Wazuh alert structure

**Tasks**:
- [ ] Review all JSONPath expressions in "Edit Fields" node
- [ ] Change `$json.sourceIP` → `$json.data.srcip`
- [ ] Change `$json.sourceUser` → `$json.data.srcuser`
- [ ] Add fallback values with `|| "N/A"` syntax
- [ ] Test with multiple alert types (SSH, malware, login)
- [ ] Document correct field mappings

**Success Criteria**:
- All observables created with correct values
- No "N/A" values unless field genuinely missing

---

### 2. Automate Cortex Enrichment from n8n
**Status**: 🟡 Medium Priority  
**Estimated Time**: 2-3 hours  
**Dependencies**: Fix #1

**Current State**:
- Cortex analyzers work perfectly when triggered manually
- n8n workflow creates TheHive cases successfully
- Missing: Automatic analyzer trigger after case creation

**Implementation Plan**:

**Step 1**: Extract Case/Alert ID from TheHive response (15 min)
```javascript
// In n8n HTTP Request node response
{
  "id": "~40988816",  // This is what we need
  "caseId": 2
}

// Add node: Extract Alert ID
const alertId = $json.id;
```

**Step 2**: Create Cortex API calls (1 hour)
```javascript
// For each analyzer, trigger via Cortex API
POST http://10.2.15.30:9001/api/analyzer/{analyzerId}/run
Headers:
  Authorization: Bearer [CORTEX_API_KEY]
Body:
{
  "data": "8.8.8.8",           // IP from observable
  "dataType": "ip",
  "tlp": 2,                    // TLP:AMBER
  "message": "Enrichment from n8n"
}

Analyzers to trigger:
- VirusTotal_GetReport_3_1
- AbuseIPDB_1_1
- MaxMind_GeoIP_4_0
```

**Step 3**: Poll for job completion (45 min)
```javascript
// Wait for analyzer jobs to complete
GET http://10.2.15.30:9001/api/job/{jobId}

// Check status every 5 seconds
if (status === "Success") {
  // Get results
  GET http://10.2.15.30:9001/api/job/{jobId}/report
}
```

**Step 4**: Update TheHive with results (30 min)
```javascript
// Add tags to observable based on results
PATCH http://10.2.15.30:9000/api/v1/case/{caseId}
Body:
{
  "customFields": {
    "virusTotalScore": 0,
    "abuseConfidence": 0,
    "geoLocation": "United States"
  }
}
```

**Tasks**:
- [ ] Create new n8n workflow: "Cortex Automation"
- [ ] Add HTTP Request nodes for each analyzer
- [ ] Implement job status polling with timeout
- [ ] Parse analyzer results (JSON)
- [ ] Update TheHive case with enrichment data
- [ ] Add error handling for API failures
- [ ] Test with known malicious IPs
- [ ] Document workflow configuration

**Success Criteria**:
- Analyzers triggered automatically within 10 seconds of case creation
- Results visible in TheHive within 60 seconds
- No manual intervention required

---

### 3. Implement Analyst Decision Prompt
**Status**: 🟡 Medium Priority  
**Estimated Time**: 2-3 hours  
**Dependencies**: Automate Cortex Enrichment (#2)

**Architecture**:
```
Cortex Enrichment Complete
    ↓
n8n sends Slack message with buttons
    ↓
Analyst clicks [Isolate] or [Ignore]
    ↓
Slack webhook → n8n
    ↓
Conditional logic executes appropriate action
```

**Implementation Plan**:

**Step 1**: Create Slack Interactive Message (1 hour)
```json
// Slack Block Kit format
{
  "text": "🚨 Alert Enrichment Complete",
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Case #2: SSH Brute Force*\n\nIOC: 8.8.8.8\nAbuse Score: 0/100\nLocation: United States"
      }
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": {"type": "plain_text", "text": "✅ Isolate Machine"},
          "style": "danger",
          "value": "isolate_001",
          "action_id": "isolate"
        },
        {
          "type": "button",
          "text": {"type": "plain_text", "text": "❌ Ignore Alert"},
          "value": "ignore_001",
          "action_id": "ignore"
        }
      ]
    }
  ]
}
```

**Step 2**: Configure Slack App for Interactivity (30 min)
- Create Slack App in workspace
- Enable Interactive Components
- Set Request URL: `http://[n8n-url]/webhook/slack-response`
- Install app to workspace
- Get OAuth token

**Step 3**: Create n8n Response Webhook (1 hour)
```javascript
// New n8n workflow: Slack Response Handler
Webhook Trigger → Switch Node → Action Nodes

// Switch based on action_id
if (action_id === "isolate") {
  → Trigger Active Response
} else if (action_id === "ignore") {
  → Update Case & Notify
}
```

**Step 4**: Add timeout logic (30 min)
```javascript
// If no response in 30 minutes → auto-ignore
Wait Node (30 minutes)
  → If no analyst response
  → Auto-ignore alert
  → Notify: "No analyst response, alert auto-ignored"
```

**Tasks**:
- [ ] Create Slack App with interactive components
- [ ] Design Slack message template with buttons
- [ ] Create n8n webhook to receive button clicks
- [ ] Implement switch logic for decision routing
- [ ] Add timeout with auto-ignore
- [ ] Test button interactions
- [ ] Document Slack app configuration

**Success Criteria**:
- Analyst receives interactive Slack message
- Buttons correctly trigger workflows
- Timeout works if no response

---

## 🎯 Priority 2: Important Features (Month 2)

### 4. Configure Wazuh Active Response
**Status**: 🟠 High Priority (Requires Testing)  
**Estimated Time**: 3-4 hours  
**Dependencies**: Decision Prompt (#3)

**Implementation Plan**:

**Step 1**: Create Active Response Script (1 hour)
```bash
# /var/ossec/active-response/bin/firewall-drop.sh
#!/bin/bash

ACTION=$1
USER=$2
IP=$3

LOG_FILE="/var/ossec/logs/active-responses.log"

if [ "$ACTION" = "add" ]; then
    # Log action
    echo "[$(date)] Isolating machine from $IP" >> $LOG_FILE
    
    # Block all inbound
    netsh advfirewall firewall add rule \
        name="SOC_Block_Inbound" \
        dir=in action=block
    
    # Block all outbound (except Wazuh)
    netsh advfirewall firewall add rule \
        name="SOC_Block_Outbound" \
        dir=out action=block
    
    # Allow Wazuh agent
    netsh advfirewall firewall add rule \
        name="SOC_Allow_Wazuh" \
        dir=out action=allow protocol=TCP remoteport=1514 \
        remoteip=10.2.15.20
    
    echo "[$(date)] Machine isolated successfully" >> $LOG_FILE
fi

if [ "$ACTION" = "delete" ]; then
    # Restore connectivity
    netsh advfirewall firewall delete rule name="SOC_Block_Inbound"
    netsh advfirewall firewall delete rule name="SOC_Block_Outbound"
    netsh advfirewall firewall delete rule name="SOC_Allow_Wazuh"
    
    echo "[$(date)] Machine connectivity restored" >> $LOG_FILE
fi
```

**Step 2**: Configure Wazuh Manager (30 min)
```xml
<!-- /var/ossec/etc/ossec.conf -->
<command>
  <name>firewall-drop</name>
  <executable>firewall-drop.sh</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>

<active-response>
  <disabled>no</disabled>
  <command>firewall-drop</command>
  <location>local</location>
  <timeout>0</timeout> <!-- Permanent until manual restore -->
</active-response>
```

**Step 3**: Test in Isolated Environment (1 hour)
- Trigger response manually: `/var/ossec/bin/agent_control -b 192.168.1.100 -f firewall-drop -u 001`
- Verify firewall rules applied
- Test that Wazuh agent still communicates
- Test restore functionality
- Document rollback procedure

**Step 4**: Integrate with n8n (1 hour)
```javascript
// n8n HTTP Request to Wazuh API
POST http://10.2.15.20:55000/active-response
Headers:
  Authorization: Bearer [JWT_TOKEN]
Body:
{
  "command": "firewall-drop",
  "arguments": [],
  "alert": {
    "agent": {"id": "001"}
  }
}
```

**Tasks**:
- [ ] Write firewall-drop.sh script
- [ ] Configure ossec.conf with active response
- [ ] Test script manually on Windows agent
- [ ] Verify Wazuh agent stays connected
- [ ] Create n8n integration
- [ ] Test full workflow: Decision → Isolation
- [ ] Create restore procedure
- [ ] Document safety precautions

**Success Criteria**:
- Analyst decision triggers isolation within 10 seconds
- Windows endpoint fully isolated
- Wazuh agent maintains connection
- Restore procedure works reliably

---

### 5. Security Onion Integration
**Status**: 🟢 Lower Priority  
**Estimated Time**: 3-4 hours  
**Dependencies**: None (parallel with above)

**Implementation Plan**:

**Step 1**: Configure Security Onion Alerts (1 hour)
- Enable Suricata custom rules
- Configure Zeek scripts for specific detections
- Set alert thresholds
- Verify Elasticsearch indexing

**Step 2**: Create n8n Elasticsearch Query (1.5 hours)
```javascript
// n8n Schedule Trigger (every 5 minutes)
Schedule Trigger → HTTP Request → Parse Results → TheHive

// Query Security Onion Elasticsearch
GET https://10.2.15.10:9200/logstash-*/_search
Body:
{
  "query": {
    "bool": {
      "must": [
        {"range": {"@timestamp": {"gte": "now-5m"}}},
        {"match": {"event_type": "alert"}},
        {"range": {"alert.severity": {"gte": 2}}}
      ]
    }
  },
  "sort": [{"@timestamp": "desc"}]
}
```

**Step 3**: Parse and Transform (1 hour)
```javascript
// Extract relevant fields
alerts.forEach(alert => {
  const newCase = {
    title: `Network Alert: ${alert.alert.signature}`,
    description: `Source: ${alert.src_ip}\nDest: ${alert.dest_ip}\nRule: ${alert.alert.signature_id}`,
    severity: mapSeverity(alert.alert.severity),
    tags: ["security-onion", "network", alert.proto],
    observables: [
      {dataType: "ip", data: alert.src_ip},
      {dataType: "ip", data: alert.dest_ip}
    ]
  };
  // Send to TheHive
});
```

**Step 4**: Deduplication Logic (30 min)
- Check if alert already exists in TheHive
- Use signature_id + src_ip + timestamp for uniqueness
- Update existing case instead of creating duplicate

**Tasks**:
- [ ] Configure Security Onion alert rules
- [ ] Create n8n scheduled workflow
- [ ] Implement Elasticsearch query
- [ ] Parse alert data
- [ ] Add deduplication logic
- [ ] Test with simulated network attacks
- [ ] Document Security Onion configuration

**Success Criteria**:
- Network alerts appear in TheHive within 5 minutes
- No duplicate cases created
- Proper correlation with Wazuh events

---

### 6. Email Notifications
**Status**: 🟢 Lower Priority  
**Estimated Time**: 1 hour  
**Dependencies**: None

**Implementation Plan**:

**Step 1**: Configure SMTP (20 min)
```javascript
// n8n Email node configuration
SMTP Server: smtp.gmail.com
Port: 587
Security: STARTTLS
Username: oussama.elmaskaoui@gmail.com
Password: [App-specific password]
```

**Step 2**: Create Email Template (20 min)
```html
Subject: [SOC Alert] {{alert.title}}

Body:
<!DOCTYPE html>
<html>
<body>
  <h2>🚨 SOC Alert Detected</h2>
  
  <table>
    <tr><td><b>Alert:</b></td><td>{{alert.title}}</td></tr>
    <tr><td><b>Severity:</b></td><td>{{alert.severity}}</td></tr>
    <tr><td><b>Source IP:</b></td><td>{{alert.sourceIP}}</td></tr>
    <tr><td><b>Time:</b></td><td>{{alert.timestamp}}</td></tr>
  </table>
  
  <p><a href="{{thehiveURL}}">View in TheHive</a></p>
</body>
</html>
```

**Step 3**: Add to n8n Workflow (20 min)
- Add Email node parallel to Slack notification
- Configure recipients (yourself + team)
- Set priority based on alert severity
- Test delivery

**Tasks**:
- [ ] Generate Gmail app-specific password
- [ ] Configure n8n Email node
- [ ] Create HTML email template
- [ ] Add to main workflow
- [ ] Test email delivery
- [ ] Add multiple recipients

**Success Criteria**:
- Emails delivered within 10 seconds
- HTML formatting displays correctly
- Links to TheHive work

---

## 🎯 Priority 3: Enhancements (Month 3+)

### 7. Additional Detection Rules
- Malware execution detection
- Privilege escalation attempts
- Lateral movement (SMB, RDP)
- Data exfiltration patterns
- Credential dumping (Mimikatz)

### 8. Dashboard & Metrics
- Kibana dashboard for SOC metrics
- Alert volume over time
- MTTR (Mean Time To Respond)
- Top alert sources
- Analyst activity tracking

### 9. Automated Response Playbooks
- Port scan → Block IP
- Malware detected → Isolate + dump memory
- Brute force → IP blacklist
- Data exfil → Kill process + block domain

### 10. Threat Hunting Capabilities
- Scheduled queries for IOCs
- YARA rule integration
- Sigma rule conversion
- Custom threat intelligence feeds

---

## 🔄 Continuous Improvement

### Regular Maintenance
- Weekly rule tuning
- Monthly analyzer updates
- Quarterly architecture review
- Continuous documentation updates

### Community Contributions
- Share learnings on GitHub
- Blog posts about challenges solved
- Contribute back to open-source tools
- Mentor others building similar labs

---

**For current status, see [PROJECT_STATUS.md](PROJECT_STATUS.md)**  
**For implementation details, see component docs in [docs/](docs/)**
