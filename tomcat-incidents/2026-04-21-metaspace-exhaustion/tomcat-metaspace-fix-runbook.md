# CRITICAL: Tomcat 9.0.117 Metaspace Exhaustion Fix
**Host:** cb-apib-proddc-rpms-02  
**Status:** Production impact - 147k Full GCs in 2 days, 86-second pauses  
**Root Cause:** Metaspace filled to 94% (492MB/512MB), Full GC every 0.4 seconds

---

## PHASE 1: IMMEDIATE STABILIZATION (5-10 min)

### Step 1: Create systemd override to increase MaxMetaspaceSize
```bash
mkdir -p /etc/systemd/system/tomcat9.service.d/
cat > /etc/systemd/system/tomcat9.service.d/override.conf << 'EOF'
[Service]
Environment="CATALINA_OPTS=-XX:MetaspaceSize=512m -XX:MaxMetaspaceSize=1536m"
EOF

# Verify syntax
cat /etc/systemd/system/tomcat9.service.d/override.conf
```

### Step 2: Reload and restart Tomcat
```bash
systemctl daemon-reload
systemctl restart tomcat9.service

# Wait 30 seconds for startup
sleep 30

# Verify Tomcat is running
systemctl status tomcat9.service
```

### Step 3: Monitor GC behavior in real-time
```bash
# Run this for 60 seconds - watch FGC column (should STOP incrementing)
jstat -gc $(pgrep -f 'java.*tomcat' | head -1) 1000 60 | head -20

# EXPECTED: FGC count should stabilize (not increment every second)
# If still incrementing rapidly → move to Phase 2
```

### Step 4: Check system stability
```bash
# Monitor application availability
curl -s http://localhost:8080/ | head -5

# Check for 5xx errors in Tomcat logs
tail -50 /opt/tomcat9/logs/catalina.out | grep -i "exception\|error" | tail -10
```

**If stabilized, STOP here and monitor for 2 hours. If still bad, proceed to Phase 2.**

---

## PHASE 2: IDENTIFY ROOT CAUSE (10-15 min)

### Step 5: Capture detailed metrics BEFORE any other changes
```bash
# Save current state for analysis
mkdir -p /tmp/tomcat-diagnostics
date > /tmp/tomcat-diagnostics/baseline.txt
ps aux | grep tomcat >> /tmp/tomcat-diagnostics/baseline.txt
jstat -gc $(pgrep -f 'java.*tomcat' | head -1) >> /tmp/tomcat-diagnostics/baseline.txt
systemctl status tomcat9.service >> /tmp/tomcat-diagnostics/baseline.txt

echo "=== Baseline captured in /tmp/tomcat-diagnostics/ ==="
```

### Step 6: Check if this is a Tomcat 9.0.117-specific regression
```bash
# Query: What was the version BEFORE upgrade?
echo "CHECK VAPT ticket/work order for pre-upgrade version"
echo "Likely: 9.0.108 or 9.0.111"
echo "If pre-upgrade was 9.0.108, we have strong evidence 9.0.117 is the culprit"

# Check Tomcat binary timestamp
ls -la /opt/tomcat9/bin/tomcat-juli.jar | awk '{print $6, $7, $8}'
stat /opt/tomcat9/RELEASE-NOTES | grep -i modify
```

### Step 7: Check for backup of pre-VAPT Tomcat
```bash
# Look for old versions
ls -la /opt/ | grep tomcat

# If backup exists at /opt/tomcat9.old or similar:
echo "BACKUP FOUND - Rollback is possible"
# Proceed to Phase 3A (Rollback)

# If no backup:
echo "NO BACKUP - Must proceed with class loading investigation"
# Proceed to Phase 3B (Deep Dive)
```

---

## PHASE 3A: ROLLBACK PATH (if pre-VAPT version exists)

### Option: Revert to pre-VAPT Tomcat
```bash
# ONLY if you have /opt/tomcat9.old or similar backup

systemctl stop tomcat9.service
sleep 5

# Backup current broken version
mv /opt/tomcat9 /opt/tomcat9-9.0.117-broken

# Restore previous version
cp -r /opt/tomcat9.old /opt/tomcat9

# Verify permissions
chown -R tomcat_admin:tomcat /opt/tomcat9

# Start with old JVM args
systemctl start tomcat9.service
sleep 30

# Monitor GC
jstat -gc $(pgrep -f 'java.*tomcat' | head -1) 1000 10

# EXPECTED: FGC count stabilizes immediately
```

**If FGC stabilizes after rollback → Keep 9.0.117 in /opt/tomcat9-9.0.117-broken for root cause analysis later.**

---

## PHASE 3B: DEEP ROOT CAUSE ANALYSIS (if no rollback option)

### Step 8: Enable class loading tracing
```bash
# Add detailed logging to catch class leaks
cat > /etc/systemd/system/tomcat9.service.d/override.conf << 'EOF'
[Service]
Environment="CATALINA_OPTS=-XX:MetaspaceSize=512m -XX:MaxMetaspaceSize=1536m -XX:+TraceClassLoading -XX:+TraceClassUnloading -XX:+PrintClassHistogram"
EOF

systemctl daemon-reload
systemctl restart tomcat9.service

# Monitor for 5 minutes
sleep 300

# Check what's being loaded
journalctl -u tomcat9 -n 1000 | grep "TraceClassLoading" | tail -20
```

### Step 9: Suspect #1 - Old Log4j classes (most likely)
```bash
# Count log4j JARs across all webapps
find /opt/tomcat9/webapps -name "log4j*.jar" | wc -l
# RESULT: 26 JARs (13 webapps × 2 JARs each) - MAJOR RED FLAG

# List all log4j versions
find /opt/tomcat9/webapps -name "log4j*.jar" -exec ls -la {} \; | sort -u

# If all are 2.17.2 → Upgrade to 2.25.3
# Create upgrade script (see Phase 4)
```

### Step 10: Suspect #2 - Security manager overhead in 9.0.117
```bash
# Check if security manager is active
grep -i "security" /opt/tomcat9/conf/catalina.policy

# Temporarily disable to test
# Edit: /opt/tomcat9/bin/catalina.sh
# Find: JAVA_OPTS="$JAVA_OPTS -Djava.security.manager..."
# Comment it out with: # JAVA_OPTS="$JAVA_OPTS -Djava.security.manager..."

systemctl restart tomcat9.service
jstat -gc $(pgrep -f 'java.*tomcat' | head -1) 1000 10

# If FGC stops, security manager overhead is culprit
# Re-enable and investigate security policy file
```

---

## PHASE 4: PROPER FIX - UPDATE LOG4J (likely permanent solution)

### Step 11: Backup current webapps
```bash
cd /opt/tomcat9/webapps
tar czf /tmp/webapps-backup-$(date +%Y%m%d-%H%M%S).tar.gz .
echo "Backup created: /tmp/webapps-backup-*.tar.gz"
```

### Step 12: Remove old Log4j 2.17.2 JARs from all webapps
```bash
# REMOVE from all webapps
find /opt/tomcat9/webapps -name "log4j-api-2.17.2.jar" -delete
find /opt/tomcat9/webapps -name "log4j-to-slf4j-2.17.2.jar" -delete

# Verify removal
find /opt/tomcat9/webapps -name "log4j*.jar" -ls
# EXPECTED: No results, all removed

# Fix permissions if needed
chown -R tomcat_admin:tomcat /opt/tomcat9/webapps
```

### Step 13: Download Log4j 2.25.3 (patched versions)
```bash
cd /tmp
wget -q https://archive.apache.org/dist/logging/log4j/2.25.3/apache-log4j-2.25.3-bin.zip
unzip -q apache-log4j-2.25.3-bin.zip

ls apache-log4j-2.25.3-bin/log4j-api-2.25.3.jar
# Should exist

cd /tmp/apache-log4j-2.25.3-bin
cp log4j-api-2.25.3.jar /tmp/
cp log4j-to-slf4j-2.25.3.jar /tmp/

cd /tmp
rm -rf apache-log4j-2.25.3-bin apache-log4j-2.25.3-bin.zip
```

### Step 14: Install updated Log4j into shared classloader
```bash
# Option 1: Put in Tomcat shared lib (RECOMMENDED for consistency)
cp /tmp/log4j-api-2.25.3.jar /opt/tomcat9/lib/
cp /tmp/log4j-to-slf4j-2.25.3.jar /opt/tomcat9/lib/

# Fix permissions
chown tomcat_admin:tomcat /opt/tomcat9/lib/log4j-*.jar

# Verify
ls -la /opt/tomcat9/lib/log4j-*.jar
```

### Step 15: Restart Tomcat
```bash
systemctl restart tomcat9.service
sleep 30

# Monitor GC behavior
jstat -gc $(pgrep -f 'java.*tomcat' | head -1) 1000 10

# EXPECTED: FGC count stabilizes, Metaspace usage drops below 50%
```

### Step 16: Verify all webapps are functional
```bash
# List deployed webapps
ls /opt/tomcat9/webapps/

# Test 2-3 critical ones
curl -s http://localhost:8080/feedback-api/ | head -2
curl -s http://localhost:8080/ifsc-service/ | head -2
curl -s http://localhost:8080/imps-api/ | head -2

# Check Tomcat logs for errors
tail -100 /opt/tomcat9/logs/catalina.out | grep -i "exception\|error" | tail -5
```

---

## PHASE 5: MONITOR & DOCUMENT (ongoing)

### Step 17: Long-term monitoring
```bash
# Run for 24 hours minimum
watch -n 60 'echo "=== $(date) ===" >> /tmp/gc-monitor.log && \
  jstat -gc $(pgrep -f "java.*tomcat" | head -1) >> /tmp/gc-monitor.log && \
  tail -3 /tmp/gc-monitor.log'

# Expected trend: FGC count should increase SLOWLY (< 1 per minute, not per second)
```

### Step 18: Document findings for handoff
```bash
cat > /tmp/tomcat-incident-summary.txt << 'EOF'
INCIDENT: Tomcat 9.0.117 Metaspace Exhaustion on cb-apib-proddc-rpms-02
DATE: 2026-04-20 to 2026-04-21
IMPACT: 147,296 Full GCs in 2 days, 86-second pause times

ROOT CAUSE:
[ ] Tomcat 9.0.117 class loading regression
[ ] Log4j 2.17.2 classloader pollution (26 old JARs across webapps)
[ ] Security manager overhead in new version
[ ] Webapp-specific class leak post-upgrade

FIX APPLIED:
[ ] Increased MaxMetaspaceSize from 512MB to 1536MB
[ ] Updated Log4j from 2.17.2 to 2.25.3
[ ] Rolled back Tomcat from 9.0.117 to 9.0.111
[ ] Other: ___________

RESULT:
[ ] Stabilized - FGC rate dropped to normal levels
[ ] Pending - Monitoring for 24 hours

NEXT STEPS FOR TCS HANDOFF:
- Monitor /tmp/gc-monitor.log for 24 hours
- If FGC rate remains < 1/min, issue resolved
- If FGC persists, escalate to Apache Tomcat security list with jstat dump
EOF

cat /tmp/tomcat-incident-summary.txt
```

---

## QUICK REFERENCE - COMMAND TO RUN RIGHT NOW

```bash
# Execute this 3-liner immediately
mkdir -p /etc/systemd/system/tomcat9.service.d/ && \
echo -e '[Service]\nEnvironment="CATALINA_OPTS=-XX:MetaspaceSize=512m -XX:MaxMetaspaceSize=1536m"' > /etc/systemd/system/tomcat9.service.d/override.conf && \
systemctl daemon-reload && systemctl restart tomcat9.service && sleep 30 && jstat -gc $(pgrep -f 'java.*tomcat' | head -1) 1000 10
```

**Then check:** FGC column should NOT increment in every row. If it does, proceed to Phase 3B.
