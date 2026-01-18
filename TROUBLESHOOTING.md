# 🐛 Troubleshooting Guide

This document contains all major technical issues encountered during the project development, along with their root causes and solutions. These represent real debugging experiences and problem-solving methodology.

---

## 🔴 Critical Issues Resolved

### Issue #1: Cortex Analyzers Failing with "Worker Not Found"

**Symptom**:
```
All Cortex analyzers showing status: Failure
Error in Cortex logs: "Worker not found"
Docker containers starting but jobs failing immediately
No results returned to TheHive
```

**Initial Investigation** (Day 1 - 2 hours):
- Checked Cortex logs: `/var/log/cortex/application.log`
- Verified Docker containers were running: `docker ps`
- Confirmed analyzers were enabled in Cortex interface
- Tested Docker manually: Containers worked when run directly

**Root Cause Discovery** (Day 2 - 1 hour):
```bash
# Tested Docker volume mounting
docker run --rm -v /tmp/test:/job alpine ls /job
# Result: Empty directory (should show files)

# Checked Snap confinement
snap connections docker
# Result: Docker Snap has restricted access to /tmp

# Verified the issue
ls -la /tmp/cortex-jobs/
# Result: Directory exists but Docker can't see it
```

**Root Cause**:
Docker installed via Snap has security confinement that prevents mounting volumes in `/tmp`. Cortex was configured to use `/tmp/cortex-jobs` for analyzer jobs, but Snap Docker containers couldn't access this location, resulting in empty job directories and "worker not found" errors.

**Solution** (Day 2 - 1 hour):

**Step 1**: Change job directory to accessible location
```bash
# Create new directory outside /tmp
sudo mkdir -p /home/thehive/cortex-jobs
sudo chown thehive:thehive /home/thehive/cortex-jobs
sudo chmod 750 /home/thehive/cortex-jobs
```

**Step 2**: Update Cortex configuration
```bash
# Edit /opt/cortex/conf/application.conf
sudo nano /opt/cortex/conf/application.conf

# Change these lines:
job {
  runner = [docker]
  directory = "/home/thehive/cortex-jobs"     # Changed from /tmp/cortex-jobs
  dockerDirectory = "/home/thehive/cortex-jobs"
}

docker {
  host = "unix:///var/run/docker.sock"
  autoUpdate = true
  attachStdin = true
}
```

**Step 3**: Configure Docker permissions
```bash
# Add docker group if not exists
sudo addgroup --system docker

# Add thehive user to docker group
sudo usermod -aG docker thehive

# Set docker socket permissions
sudo chown root:docker /var/run/docker.sock
sudo chmod 660 /var/run/docker.sock
```

**Step 4**: Create systemd override for Cortex
```bash
# Create override directory
sudo mkdir -p /etc/systemd/system/cortex.service.d/

# Create override file
sudo nano /etc/systemd/system/cortex.service.d/docker.conf

# Add content:
[Service]
SupplementaryGroups=docker
Environment="DOCKER_HOST=unix:///var/run/docker.sock"
```

**Step 5**: Restart services
```bash
sudo systemctl daemon-reload
sudo systemctl restart cortex
sudo docker ps  # Verify containers can start
```

**Verification**:
```bash
# Test Docker volume mounting with new location
docker run --rm -v /home/thehive/cortex-jobs:/job alpine ls -la /job
# Result: Success! Directory is accessible

# Test analyzer from Cortex interface
# Result: VirusTotal, AbuseIPDB, MaxMind all return "Success"
```

**Time to Resolve**: ~4 hours total  
**Status**: ✅ Fully Resolved

**Lessons Learned**:
- Snap confinement can cause unexpected Docker volume mounting issues
- Always verify Docker volume accessibility when using Snap-installed Docker
- Testing with simple Docker commands helps isolate permission problems
- Systemd service configuration important for proper group membership

---

### Issue #2: JSONPath Data Extraction Returning "N/A"

**Symptom**:
```
TheHive observables created with value: "N/A"
Expected: Actual IP addresses from Wazuh alerts
n8n workflow shows successful execution
No errors in logs
```

**Investigation** (1 hour):
- Examined Wazuh alert JSON structure
- Checked n8n "Edit Fields" node configuration
- Tested JSONPath expressions in n8n debugger
- Compared expected vs actual data paths

**Root Cause**:
JSONPath expressions in n8n were looking for fields that don't exist in Wazuh alert structure.

**Incorrect expressions**:
```javascript
sourceIP: {{ $json.sourceIP }}        // Field doesn't exist
sourceUser: {{ $json.sourceUser }}    // Field doesn't exist
```

**Wazuh alert actual structure**:
```json
{
  "rule": {...},
  "agent": {...},
  "data": {
    "srcip": "8.8.8.8",        // Actual field name
    "srcuser": "admin"         // Actual field name
  }
}
```

**Solution**:
```javascript
// Corrected expressions in n8n Edit Fields node
alertTitle: {{ $json.rule.description }}
alertLevel: {{ $json.rule.level }}
sourceIP: {{ $json.data.srcip || "N/A" }}       // Correct path
sourceUser: {{ $json.data.srcuser || "N/A" }}   // Correct path
agentName: {{ $json.agent.name }}
ruleId: {{ $json.rule.id }}
```

**Verification**:
- Triggered test alert from Kali → Windows
- Checked TheHive observable: IP = 8.8.8.8 ✅
- Verified all fields populated correctly

**Time to Resolve**: ~2 hours  
**Status**: ✅ Resolved

**Lessons Learned**:
- Always examine actual JSON structure before writing JSONPath
- Use n8n debugger to inspect incoming data
- Add fallback values with `|| "N/A"` for optional fields
- Document correct field mappings for future reference

---

## 🟡 Medium Issues Resolved

### Issue #3: TheHive API Authentication Failures

**Symptom**:
```
HTTP 401 Unauthorized from TheHive API
n8n HTTP Request node failing
Error: "Authentication failure"
```

**Root Cause**:
Using wrong authentication method. TheHive 4.x requires Bearer token in header, not basic auth.

**Incorrect approach**:
```javascript
Headers:
  Authorization: "thehive-api-key"  // Wrong format
```

**Correct solution**:
```javascript
Headers:
  Authorization: "Bearer thehive-api-key"  // Correct format
  Content-Type: "application/json"
```

**Time to Resolve**: 30 minutes  
**Status**: ✅ Resolved

---

### Issue #4: Python Library Version Incompatibility

**Symptom**:
```
Custom Wazuh integration script failing
Error: "AttributeError: TheHiveApi has no attribute 'create_alert'"
Python script crashes on execution
```

**Root Cause**:
Using thehive4py version 2.x with TheHive 4.x (incompatible).

**Version Requirements**:
```
TheHive 4.x → Requires thehive4py 1.8.1
TheHive 5.x → Requires thehive4py 2.x
```

**Solution**:
```bash
# Uninstall incorrect version
sudo pip3 uninstall thehive4py

# Install correct version for TheHive 4.x
sudo pip3 install thehive4py==1.8.1 --break-system-packages

# Verify installation
python3 -c "import thehive4py; print(thehive4py.__version__)"
# Output: 1.8.1
```

**Time to Resolve**: 45 minutes  
**Status**: ✅ Resolved

**Lessons Learned**:
- Always check API library version compatibility
- Read official documentation for version requirements
- Test imports before writing full integration

---

### Issue #5: Wazuh Integration Script Permission Denied

**Symptom**:
```
Wazuh logs show: "Permission denied: /var/ossec/integrations/custom-w2n8n"
Integration not executing
Alerts not forwarded to n8n
```

**Root Cause**:
Custom integration script not executable.

**Solution**:
```bash
# Set execute permissions
sudo chmod 750 /var/ossec/integrations/custom-w2n8n
sudo chown root:wazuh /var/ossec/integrations/custom-w2n8n

# Verify
ls -la /var/ossec/integrations/custom-w2n8n
# Output: -rwxr-x--- 1 root wazuh [...] custom-w2n8n

# Restart Wazuh
sudo systemctl restart wazuh-manager
```

**Time to Resolve**: 15 minutes  
**Status**: ✅ Resolved

---

### Issue #6: n8n Workflow Execution Timeout

**Symptom**:
```
n8n workflow execution timeout after 60 seconds
Cortex enrichment incomplete
Workflow fails in middle of execution
```

**Root Cause**:
Default n8n execution timeout too short for multiple API calls.

**Solution**:
```javascript
// In n8n Settings → Executions
Execution Timeout: 300 seconds (5 minutes)

// For specific nodes (HTTP Request):
Settings → Timeout: 120000 ms (2 minutes)
```

**Time to Resolve**: 20 minutes  
**Status**: ✅ Resolved

---

## 🟢 Minor Issues & Quick Fixes

### Issue #7: Slack Webhook 404 Error

**Problem**: Webhook URL returned 404  
**Cause**: Typo in webhook URL  
**Fix**: Verified URL in Slack app settings, corrected in n8n  
**Time**: 10 minutes

---

### Issue #8: VirtualBox VM Network Not Reachable

**Problem**: VMs couldn't ping each other  
**Cause**: NAT network not attached to all VMs  
**Fix**: Settings → Network → Attached to: NAT Network (SOC-Lab)  
**Time**: 15 minutes

---

### Issue #9: Wazuh Agent Not Reporting

**Problem**: Windows agent showing as "Disconnected"  
**Cause**: Windows firewall blocking port 1514  
**Fix**: 
```powershell
netsh advfirewall firewall add rule name="Wazuh" dir=out action=allow protocol=TCP remoteport=1514
```
**Time**: 20 minutes

---

### Issue #10: TheHive Web Interface Not Accessible

**Problem**: Could not access TheHive at http://10.2.15.30:9000  
**Cause**: Service not started after VM reboot  
**Fix**:
```bash
sudo systemctl start thehive
sudo systemctl enable thehive  # Auto-start on boot
```
**Time**: 5 minutes

---

## 🔧 Common Troubleshooting Procedures

### Debugging Wazuh Integration

```bash
# Check Wazuh logs for integration execution
sudo tail -f /var/ossec/logs/ossec.log | grep integration

# Test integration script manually
sudo /var/ossec/integrations/custom-w2n8n 
'{"rule":{"level":5},"data":{"srcip":"1.2.3.4"}}'

# Verify integration configuration
cat /var/ossec/etc/ossec.conf | grep -A 10 integration

# Check integration script permissions
ls -la /var/ossec/integrations/
```

### Debugging n8n Workflows

```bash
# Check n8n logs
docker logs n8n -f

# Test webhook manually
curl -X POST http://10.2.15.40:5678/webhook/wazuh-alert \
  -H "Content-Type: application/json" \
  -d '{"rule":{"level":5,"description":"Test"},"data":{"srcip":"1.2.3.4"}}'

# Export workflow for backup
# Settings → Workflows → Download

# Check n8n process status
docker ps | grep n8n
```

### Debugging TheHive/Cortex

```bash
# Check TheHive logs
sudo journalctl -u thehive -f

# Check Cortex logs
sudo journalctl -u cortex -f

# Verify Cortex-TheHive connection
curl http://localhost:9001/api/status

# Test analyzer manually
curl -X POST http://localhost:9001/api/analyzer/MaxMind_GeoIP_4_0/run \
  -H "Authorization: Bearer [API_KEY]" \
  -H "Content-Type: application/json" \
  -d '{"data":"8.8.8.8","dataType":"ip"}'
```

### Debugging Docker Containers

```bash
# List running containers
docker ps

# Check container logs
docker logs [container_name]

# Inspect container
docker inspect [container_name]

# Test volume mounting
docker run --rm -v /path/to/mount:/test alpine ls -la /test

# Check Docker socket permissions
ls -la /var/run/docker.sock
```

---

## 📊 Debugging Methodology

### Standard Troubleshooting Process

1. **Reproduce the issue** consistently
2. **Check logs** (application, system, Docker)
3. **Isolate the component** (which system is failing?)
4. **Test manually** (curl, CLI commands)
5. **Verify configuration** (syntax, permissions, paths)
6. **Search documentation** (official docs, GitHub issues)
7. **Test fix** in isolation
8. **Document solution** for future reference

### Tools Used for Debugging

```bash
# Log viewing
tail -f /var/log/[service]/application.log
journalctl -u [service] -f
docker logs [container] -f

# Network testing
ping [ip]
curl -v [url]
netstat -tulpn | grep [port]
tcpdump -i any port [port]

# Process inspection
ps aux | grep [process]
systemctl status [service]
htop

# File permissions
ls -la [file]
namei -l [path]

# JSON validation
cat file.json | jq .
echo $json | python3 -m json.tool
```

---

## 💡 Prevention Strategies

### Avoiding Common Pitfalls

1. **Version Compatibility**: Always check library/API versions before integration
2. **Permissions**: Set correct ownership and execute permissions immediately
3. **Configuration Syntax**: Use syntax validators before applying configs
4. **Backups**: Snapshot VMs before major changes
5. **Testing**: Test each component individually before integration
6. **Documentation**: Document working configurations as you go

### Testing Checklist Before Deployment

```
□ All services start automatically on boot
□ API keys valid and not expired
□ Network connectivity verified (ping all systems)
□ Firewall rules allow required traffic
□ Logs accessible and rotating properly
□ Disk space sufficient (>20% free)
□ Backups/snapshots taken
□ Documentation updated
```

---

## 🆘 Getting Help

### When Stuck

1. **Official Documentation**:
   - Wazuh: https://documentation.wazuh.com/
   - TheHive: https://docs.thehive-project.org/
   - Cortex: https://github.com/TheHive-Project/Cortex/
   - n8n: https://docs.n8n.io/

2. **Community Forums**:
   - Wazuh Google Group
   - TheHive Discord
   - n8n Community Forum

3. **GitHub Issues**:
   - Search existing issues first
   - Provide: logs, config files, steps to reproduce

4. **Stack Overflow**:
   - Tag: [wazuh], [thehive], [n8n]
   - Include minimal reproducible example

---

## 📝 Issue Reporting Template

When documenting new issues:

```markdown
## Issue: [Brief Description]

**Symptom**:
[What's broken? What error messages?]

**Environment**:
- Component: [Wazuh/TheHive/n8n/etc.]
- Version: [X.X.X]
- OS: [Ubuntu 20.04]

**Steps to Reproduce**:
1. [Step 1]
2. [Step 2]
3. [Result]

**Expected Behavior**:
[What should happen]

**Actual Behavior**:
[What actually happens]

**Logs**:
```
[Relevant log excerpts]
```

**Investigation**:
[What you've tried so far]

**Solution** (if found):
[What fixed it]

**Time to Resolve**: [X hours/days]
```

---

**For architecture details, see [ARCHITECTURE.md](ARCHITECTURE.md)**  
**For implementation status, see [PROJECT_STATUS.md](PROJECT_STATUS.md)**
