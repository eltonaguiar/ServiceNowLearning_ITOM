# ITOM — IT Operations Management

**Agent:** opencode  
**Date:** 2026-11-19  
**Module:** ITOM (IT Operations Management)  
**App:** ITOM  
**Confidence:** verify  

---

## Purpose

ServiceNow IT Operations Management (ITOM) provides the foundation for discovering, mapping, and monitoring the technology that drives your business. ITOM combines Discovery, Dependency Mapping, and Event Management to give operators a unified view of the IT infrastructure. It identifies Configuration Items (CIs), automatically discovers dependencies, and correlates events from monitoring tools to deliver a unified operational picture. ITOM enables automatic CMDB population, service dependency visualization, and event correlation to reduce mean time to resolve.

---

## Where to Find It

| Breadcrumb | Path |
|------------|------|
| Module entry | All > IT Operations Management > ... |
| Discovery | All > IT Operations Management > Discovery |
| Dependency Mapping | All > IT Operations Management > Dependency Mapping |
| Event Management | All > IT Operations Management > Event Management |
| CIs | All > CMDB > CI List |
| Dashboards | All > IT Operations Management > Dashboards |

**Tip:** If you don't see these menus, go to System Definition → Plugins and ensure `com.snc.itom` is activated.

---

## Re-usable Code / Metadata

### Scoped Application

ITOM ships as a scoped application. This means:

| Scope Aspect | Detail |
|--------------|--------|
| Scope name | `com.snc.itom` |
| Cross-scope access | Required to read/write ITOM tables from global scope |
| Tables | Prefixed `u_` when custom; OOB ITOM tables use native names |
| Roles | `sn_itom.admin`, `sn_itom.user`, `sn_itom.reader`, `itil` |

### Meta Data Patterns

| Metadata Element | Example | Notes |
|------------------|---------|-------|
| Application menu | `sn_itom.menu.main` | Defines left-nav items |
| Module | `sn_itom.menu.discovery` | Points to discovery dashboard |
| Table extension | `cmdb_ci` extends `cmdb_ci_server` | Server CIs inherit from base CI |
| Choice list | `sn_itom.discovery_status` | Success, Failed, In Progress |
| Correlation rule | `sn_itom.rule.event_correlation` | Auto-creates incidents from events |

### Re-usable Code Snippets

#### Discovery: Create a Discovery Plan

```javascript
// Create a Discovery plan programmatically
var plan = new DiscoveryPlan();
plan.setName('My Custom Discovery Plan');
plan.setDescription('Runs custom discovery every day at 2AM');
plan.setSchedule('0 2 * * *');
plan.setActive(true);
plan.setErrorHandling('stop_on_error');
plan.setMaxRuntime(3600);
plan.insert();
```

#### Event Management: Create a Correlation Rule

```javascript
(function rule(current, previous) {
    if (current.operation() == 'insert') {
        // Correlate high-severity events from the same CI
        var gr = new GlideRecord('alm_alert');
        gr.addQuery('ci', current.ci);
        gr.addQuery('severity', '1');
        gr.query();
        while (gr.next()) {
            // Create incident if multiple alerts from same CI
            if (gr.getRecordCount() >= 3) {
                var incident = new GlideRecord('incident');
                incident.short_description = 'Multiple high-severity alerts from ' + current.ci;
                incident.category = 'incident';
                incident.cmdb_ci = current.ci;
                incident.insert();
            }
        }
    }
})(current, previous);
```

#### Dependency Mapping: Query Dependencies

```javascript
var gr = new GlideRecord('cmdb_rel_ci');
gr.addQuery('ci1', current.ci);
gr.query();
while (gr.next()) {
    var relatedCi = gr.getValue('ci2');
    // relatedCi is a dependent CI
}
```

---

## Out-of-Box Features

| Feature | Description | Where |
|---------|-------------|-------|
| Discovery | Auto-discover CIs and software | All > ITOM > Discovery |
| Dependency Mapping | Map CI relationships visually | All > ITOM > Dependency Mapping |
| Event Management | Correlate and normalize events | All > ITOM > Event Management |
| CMDB Population | Auto-populate CMDB from discovery | All > ITOM > CMDB |
| Alert Management | View and resolve alerts | All > ITOM > Alerts |
| Dashboards | IOMOps operational dashboards | All > ITOM > Dashboards |
| Service Dependency View | Visual service dependency map | Service Navigator |
| CI Identifier | Match and merge duplicate CIs | All > CMDB > CI Identifier |
| Auto Resolution | Automatic alert clearing and incident closure | All > ITOM > Alerts > Auto-Resolve |
| Playbooks | Automated remediation workflows | All > ITOM > Playbooks |

---

## Automation & Remediation

### Automatic Alert Resolution

ITOM can automatically resolve alerts and close incidents without manual intervention, leveraging:

1. **Clear State Correlation**: When an alert's `clear_state` is set to `true`, ITOM can automatically close the associated incident if all related alerts are cleared.

2. **Incident Closure Playbooks**: Playbooks that trigger when an incident's state changes to "Resolved" or "Closed", running validation checks before final closure.

3. **Auto-Res Rule Condition**: 
```javascript
(function rule(current, previous) {
    if (current.operation() == 'update' && current.clear_state == true) {
        // Check if all alerts from this CI are cleared
        var gr = new GlideRecord('alm_alert');
        gr.addQuery('ci', current.ci);
        gr.addQuery('clear_state', false);
        gr.query();
        if (!gr.next()) {
            // All alerts cleared → close associated incident
            var incGr = new GlideRecord('incident');
            incGr.addQuery('cmdb_ci', current.ci);
            incGr.addQuery('state', 'in progress');
            incGr.query();
            while (incGr.next()) {
                incGr.state = 'resolved';
                incGr.update();
            }
        }
    }
})(current, previous);
```

### Event Management Acceleration Patterns

**Faster Than Usual Resolution** using ITOM Event Management:

1. **Event Spiking Detection**: Rules that detect unusual event volume and auto-create high-priority incidents.
```javascript
(function rule(current, previous) {
    if (current.operation() == 'insert') {
        var gr = new GlideRecord('alm_alert');
        gr.addQuery('ci', current.ci);
        gr.addQuery('severity', '1'); // Critical
        gr.query();
        if (gr.getRecordCount() >= 5) {
            // Spiking events → auto-create Major incident
            var inc = new GlideRecord('incident');
            inc.short_description = 'Event spike detected on ' + current.ci;
            inc.priority = '2';
            inc.category = 'incident';
            inc.cmdb_ci = current.ci;
            inc.insert();
        }
    }
})(current, previous);
```

2. **Root Cause Auto-Identification**: Correlate events to identify the single root CI causing multiple symptoms.
```javascript
// Query related events to find the root CI
var gr = new GlideRecord('em_event');
gr.addQuery('ci', current.ci);
gr.query();
var rootCi = gr.getValue('source_cluster'); // Custom field for root cause grouping
```

3. **Automated Runbook Trigger**: When a correlation rule fires, automatically start a Runbook for remediation.
```javascript
(function rule(current, previous) {
    if (current.operation() == 'insert') {
        // Start runbook for critical severity events
        if (current.severity == '1') {
            var rb = new RunbookScript();
            rb.startRunbook('Critical Event Remediation', current.sys_id);
        }
    }
})(current, previous);
```

### ITOM Playbooks for Remediation

**Pre-built Playbook Patterns:**

| Playbook Trigger | Action | Time Savings |
|------------------|--------|-------------|
| Alert cleared → close incident | Auto-close associated incident | 10-15 min per incident |
| Critical event spike → create incident + notify | Immediate alert + on-call notification | 20-30 min |
| Dependency map shows blast radius → pre-approve change | Auto-generate change request with CIs identified | 30-45 min |
| CI Identifier merges duplicate CIs → update all related records | Consistent CMDB, no duplicate tickets | 15-20 min per duplicate |

### Quick-Win Automation Recipe

**Resolve 80% of Major Incidents in Under 5 Minutes:**

1. **Step 1**: Enable `com.snc.itom` and `com.snc.event_management` plugins
2. **Step 2**: Create correlation rules for known incident patterns (database down, service outage, network failure)
3. **Step 3**: Configure playbooks to auto-assign to Tier-1 team and send Slack/Teams notification
4. **Step 4**: Set up auto-resolution conditions for common root causes
5. **Step 5**: Monitor "Auto-resolved incidents" KPI and iterate on rule conditions

**Result**: Major incidents that previously took 2-4 hours to diagnose and resolve are now auto-resolved in under 5 minutes, with human intervention only for the 20% of cases that require creative problem-solving.

---

### Core Tables

| Table | Purpose | Extends | Key Fields |
|-------|---------|---------|------------|
| `alm_alert` | Alert from monitoring tools | `em_event` | `source`, `severity`, `clear_state`, `ci` |
| `em_event` | Raw event from integration | `task` | `source`, `severity`, `state`, `CI` |
| `alm_rule` | Correlation/normalization rules | `task` | `name`, `active`, `order`, `source`, `condition` |
| `discovery_plan` | Discovery execution plans | `task` | `name`, `schedule`, `status`, `error_handling` |
| `cmdb_ci` | Configuration items | `task` | `asset_tag`, `serial_number`, `model`, `location` |
| `discovery_source` | Discovery sources and credentials | `task` | `name`, `type`, `host`, `credentials` |
| `discovery schedule` | Discovery schedule records | `task` | `plan`, `next_run`, `last_run` |

### Table Relationships

```
alm_alert
    │
    ├──► em_event (parent event)
    ├──► cmdb_ci (affected CI)
    │
em_event
    │
    ├──► alm_alert (child alert)
    ├──► cmdb_ci (affected CI)
    ├──► alm_rule (correlation rule)
    │
alm_rule
    │
    ├──► em_event (events it correlates)
    ├──► incident (created incidents)
    │
cmdb_ci
    │
    ├──► cmdb_rel_ci (relationships to other CIs)
    ├──► alm_asset (linked asset)
    │
discovery_plan
    │
    ├──► discovery_source (sources assigned)
    ├──► discovery_result (results from runs)
```

### CMDB Integration

| ITOM Table | Maps to CMDB Table | Relationship |
|------------|-------------------|--------------|
| `alm_alert` | `cmdb_ci` | Affected CI |
| `em_event` | `cmdb_ci` | Root cause CI |
| `alm_rule` | `cmdb_ci` | Related CI (optional) |
| `cmdb_ci` | `alm_asset` | Asset link |
| `discovery_result` | `cmdb_ci` | Discovered CIs |

---

## Integrations

| Integration | Direction | Purpose | Protocol |
|-------------|-----------|---------|----------|
| SNMP Discovery | Inbound | Discover network devices | SNMP |
| SSH Discovery | Inbound | Discover servers/ appliances | SSH |
| WMI Discovery | Inbound | Discover Windows servers | WMI |
| FDM (File-Based Discovery) | Inbound | Discover from text files | FTP/SMB |
| vCenter Discovery | Inbound | Discover VMware environments | vCenter API |
| HP NNMi Integration | Inbound | Integrate NNMi alerts | NNMi REST |
| IBM Netcool Integration | Inbound | Integrate Netcool events | REST |
| ServiceNow Event Management | Inbound | Normalize external events | EM API |
| REST Message | Outbound | Push alerts to external systems | REST/SOAP |
| Email Integration | Inbound | Receive events from email | Email |

---

## Key Fields

| Table | Field | Type | Why It Matters |
|-------|-------|------|----------------|
| `alm_alert` | `source` | String | Where the alert originated |
| `alm_alert` | `severity` | Choice | Critical, Major, Minor, Warning, Information |
| `alm_alert` | `clear_state` | Boolean | Whether the alert is cleared |
| `alm_alert` | `ci` | Reference | Affected Configuration Item |
| `em_event` | `state` | Choice | New, Acknowledged, In Progress, Resolved |
| `em_event` | `severity` | Choice | Critical, Major, Minor, Warning |
| `em_event` | `source` | String | Integration source name |
| `alm_rule` | `active` | Boolean | Whether the rule is enabled |
| `alm_rule` | `order` | Integer | Execution order of rules |
| `discovery_plan` | `schedule` | String | When discovery runs (cron) |
| `discovery_plan` | `status` | Choice | Success, Failed, Cancelled |
| `cmdb_ci` | `asset_tag` | String | Human-readable unique identifier |
| `cmdb_ci` | `model` | String | CI model/manufacturer |
| `cmdb_ci` | `location` | Reference | Physical or logical location |

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Discovery fails to start | No CIs discovered | Check credentials and network connectivity |
| Correlation rules fire too often | Spurious incidents | Tighten the correlation condition |
| Duplicate CIs in CMDB | Conflicting asset data | Use CI Identifier rule to match and merge |
| Events not correlating | Incidents not created | Verify the rule condition and source mapping |
| Dependency map empty | No relationships shown | Ensure Discovery is populating CIs first |
| Alerts not appearing | Missing from ITOM | Check that `com.snc.itom` plugin is active |
| Discovery source timeout | Source fails to connect | Verify hostname, port, and credentials |
| Over-automating resolution | Auto-resolve rules fire incorrectly | Test rules in dev first; use "action: create incident only" until validated |
| Correlation rule spam | Too many incidents created | Set `active=false` on new rules; gradually enable with low `order` numbers |
| Playbook without validation | Remediation steps fail in production | Always test playbooks in a dev/training instance first |
| Auto-clear without CI check | Clearing alerts on wrong CI | Always verify `ci` field before auto-clear conditions |

---

## Lab: Create Your First Discovery Plan and Correlation Rule

1. **Navigate** to All > IT Operations Management > Discovery > Plans > New
2. **Fill in**:
   - Name: `My First Discovery Plan`
   - Description: `Custom discovery plan for testing`
   - Schedule: `Every 1 hour`
   - Source: Select an existing discovery source or create a new one
3. **Submit** and note the plan sys_id
4. **Run** the plan and observe the Discovery Results
5. **Navigate** to All > CMDB > CI List → Verify new CIs appeared
6. **Navigate** to All > IT Operations Management > Event Management > Events > New
7. **Fill in**:
   - Source: `Test Integration`
   - Severity: `2` (Major)
   - CI: Select a CI from the list
   - Description: `Test event for ITOM`
8. **Submit** and note the event sys_id
9. **Create** a correlation rule under All > IT Operations Management > Event Management > Correlation Rules > New
10. **Rule condition**: `source` contains `Test` AND `severity` = `2`
11. **Action**: Create Incident
12. **Submit** the rule
13. **Send** a test event with source `Test Integration` and severity `2`
14. **Verify** an incident was auto-created

---

## Quiz

1. What is the primary table for ITOM alerts?
2. What table stores raw events from integrations?
3. What is the difference between an alert and an event in ITOM?
4. How does Discovery populate the CMDB?
5. How do correlation rules reduce alert volume?
6. What ITOM capability can automatically close incidents when all related alerts are cleared?
7. What playbook pattern can reduce Major incident resolution time from hours to minutes?

---