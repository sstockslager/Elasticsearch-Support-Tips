# Security Rule Creation

## Summary

The Elastic Security Stack hold hundreds of detections to help Analysts set up and configure a SIEM space for any given environment. Depending on the needs of that specific instance, some complex alerting may be required. This page is meant to give a brief overview in the various alert rule types that are available. 

## Rule Types

### KQL

[Kibana Query Language](/ElasticCodeVariations/KQL.md), or KQL, is the most basic version of alert makeup. It is considered a one-for-one match between the query itself and the matching documents. Essentially, any query that is generated in the Discover page can be transferred to the Security Stack to be alerted on. Some examples of a KQL detection are shown below:
```
data_stream.dataset: "system.security" AND event.code: "4625" AND user.name: "administrator"

event.category: "network" AND NOT (source.ip: ("10.0.0.0/8" OR "192.168.0.0/12") AND (event.outcome: "failure" OR event.type: "denied"))

user.id : "S-1-5-18" and event.action: "start" AND NOT process.args: "C:\windows\temp\nessus_*.txt"
```

### EQL

[Event Query Language](/ElasticCodeVariations/EQL.md), or EQL, is a separate language designed with correlation and event order in mind. This makes it particularly powerful when it comes to threat hunting. The example below is an EQL rule that is looking for a user that ran all three executions within a span of 10 minutes. 
```
sequence by user.name with maxspan=10m 
[process where process.name == "cmdl.exe"]
[process where process.name == "whoami.exe"]
[process where process.name == "findstr.exe"]
```

### ES|QL

[Elasticsearch Query Language](/ElasticCodeVariations/ESQL.md), while like SQL, is newer addition to Kibana designed to expand upon KQL to provide more advanced statistical analysis and aggregation. In addition to a more optimized search performance, the expanded command structure allowed for more in-depth log analysis to uncover potential security risks. The example below returns when the destination IP address is not local and the number of bytes being sent over NetFlow is over 1000. 
```
FROM logs-netflow.log*
| KEEP source.ip, destination.ip, source.bytes, destination.bytes
| EVAL high_bytes = source.bytes > 1000
| EVAL local_ip = cidr_match(destination.ip, "10.0.0.0/8")
| WHERE high_bytes and not local_ip
```

### Machine Learning

Machine Learning is a feature in Elastic that can review large amounts of data and generate models to identify patterns or anomalies. All Security alerts made with ML are a part of the Anomaly Detection portion of Machine Learning. Within the Security Stack, the Machine Learning rules consists of two parts: the ML Jobs in Kibana (which is made up of the ML Job query and datafeed) and the Security Rule. Once the jobs are configured and running, the alert can be configured to fire on events reported by the ML job that are above a certain severity rating. 

NOTE that any job that is made for the Security Stack needs to be tagged "Security" in the Machine Learning section under Kibana. Without the Security tag, the job will not be available in the Security Stack to make a rule. 

### Threshold

Threshold rules are an expansion upon a KQL rule. But where a regular KQL rule looks for a one-to-one result (one document match to the provided KQL query), a Threshold rule looks for a series of similar documents within a certain time frame for a match to be made. There are three main parts of a Threshold rule to keep in mind: 

1. Query - this is the KQL query to match all related documents that could be a part of the rule. 
2. Grouping - indicate the field or fields will be uniform across all matching documents. 
3. Cardinality - this section is optional but can be used if there is a field that should have a specific number of distinct values (i.e. uniqueness). 

### New Terms

As the name implies, this alert looks for a differentiating value in a certain timeframe. That is, the alert will fire within the matching query whenever it sees a value on a selected field that is not a part of normal operating status (or high cardinality). The query will only review matching documents within a specific timeframe rather than ALL matching documents, which depending on the alert configuration, may be a limitation. Like a Threshold rule, there are two main parts:

1. Fields - the field or fields that is reviewed for new values. NOTE that multiple fields can be chosen, but each unique combination of values are reviewed separately. So, the more fields that are selected, the more unique combinations are available. And utilizing fields that normally have a high number of distinct values may also influence alert results.

2. History Window Size - the time range that is searched for new values. For correct configuration, this value must be larger than the scheduled look back time. 

### Indicator Match

For environments that ingest IoC (Indicators of Compromise) values from a Threat Intel feed, the ability to generate an alert from these values is available. The basic makeup of this rule is to line up the raw ingested data from the environment against the Intel feed and search for matching values. The design of this rule has several parts that must be set up correctly:

1. Index Pattern - the index or data view of the parsed log data that is being ingested from Elastic Agents.
2. Query - the KQL query used to further filter down the index.
3. Indicator Index Pattern - when threat intel feeds are ingested, they are stored in a hidden index starting with '.items-*'. Otherwise, calling up the intel feed follows the same rules as the index pattern for Elastic Agent data. 
4. Indicator Query - if required, a KQL query to reduce the number of results brought up from the Intel feed. 
5. Indicator Mapping - link the field or fields that is being searched for from the Elastic indices to be compared. A field type must also be selected; either keyword, text, or IP.   

## Additional Setup and Activation

### Configuration and Settings

After setting up the rule logic, there are multiple different settings that can be configured for the rule. While many of them are not required, some would be beneficial to take the time to fill out:

- __Alert Suppression__ - if a rule is expected to fire in a series of alerts over the same events, adding alert suppression using the field with that repeated value will group those alerts together. For example, if there is a series of 10 alerts with user.name: "Administrator", then the rule can be suppressed from 10 alerts to 1; either for that one rule execution or over a specific time period. 

- __Required Fields__ - a list of fields that are required to be present for the rule to be run. Additionally, these can also help the user reviewing the alert to quickly check the fields that are considered important to the alert. 

- __Name__ - the rule title.

- __Description__ - while not required, it is useful to write out what the rule is looking for the user to understand.

- __Severity__ - Noting the order of importance for triage when this alert fires. This can be entered by the rule author or if the alert is based on external security alerts that come equipped with its own severity level, then __Severity Override__ can be used to indicate the field where that external severity is located.

[Elastic Rules Severity Override](https://www.elastic.co/docs/solutions/security/detect-and-alert/create-detection-rule)

- __Tags__ - While tagging the alert with specific notations can be useful for later recall, some Tags that came with the pre-built Elastic Rules are still helpful for later grouping of similar alerts. Each Tag is case-specific, and there are several pre-built ones that are useful to follow on custom rules. Any "Rule Type: *" tag will group the alert based on the type of alert (i.e. KQL, Machine Learning, Threshold, etc.). "Data Source: *" to group together rules that focus on a specific Integration, such as Windows or AWS. "OS: *" for a specific operating system. "Tactic: *" to recall how the alert is mapped to the MITRE ATT&CK Framework.

- __Reference URLs__ - if the rule logic or inspiration originated from a news article, security blog post, or CVE notification, then a hyperlink to the webpage can be posted here.

- __False Positive Examples__ - to aid in triage, examples of expected potential false positives can be described here.

- __MITRE ATT&CK__ - correct selections of related MITRE ATT&CK Tactics and Techniques to the rule logic will help fill in MITRE ATT&CK Coverage Dashboard in the Security Stack. This dashboard will enable the user to view attack coverage on enabled alerts and potential detection gaps. 

[MITRE ATT&CK Coverage Dashboard](https://www.elastic.co/docs/solutions/security/detect-and-alert/mitre-attandckr-coverage)

- __Investigation Guide__ - Use this section to aid the reviewing analyst in providing an appropriate step-by-step triage guide. What to look for, information about the potential security issue such as why this alert is a potential risk, how to determine if this alert is considered a true positive, and what remediation steps should be taken. 

- __Author__ - while most pre-built alerts have an author of _Elastic_, the user can input their own name here to later track all rules that were generated by them. Rule authors can be searched for later in Discover or the Alerts Dashboard using the field "kibana.alert.rule.author".

- __Building Block__ - Several pre-built alerts are configured to be set as "Building Block", which prevents any alerts that fire from this rule from being initially seen in the Alerts Dashboard. These pre-built rules are meant to form the backbone of other alerts that look when there is a series of build blocks that fire within a certain time frame. Many of the building block rules are also very generic and intentionally noisy and usually require tuning specific to the cluster's environment. In this case, it also makes the building block ability the perfect place to test custom alerts for noise without interrupting the security posture of a security team. 

[Elastic Building Block Documentation](https://www.elastic.co/docs/solutions/security/detect-and-alert/about-building-block-rules)

- __Actions__ - One of the last parts of any rule before enabling it is deciding if any additional Actions are required if an alert fires. Many of these actions use a Connector to link to an external software such as Teams, ServiceNow, or Jira for additional notification purposes. But there are options for additional actions to be taken such as sending an email, executing a command via a Webhook, or processing the alert through a SOAR platform via Tines. 

[Elastic Security Connectors](https://www.elastic.co/docs/reference/search-connectors/es-connectors-security)

### Detection Scheduling Considerations

Once the rule itself is fully set up, one of the last parts that needs to be taken into consideration is the rule scheduling. This is the setting that determines when the rule is run. There are two main parts:

1. The rule's _Run Time_, which is both how often the rule runs and the amount of time the rule will look at to detect matching alerts.

2. The _look-back time_ adds additional time to the Run Time on the chance that additional time is required to not miss alerts. 

There could also be other situations that should be considered that may not be apparent:

- Depending on the environment, some data can be ingested from sources across more than one time zone. An easier resolution to this is to set up all the aggregation servers that are ingesting into Elastic to report as UTC rather than the local time zone. If this cannot be accomplished, then the look-back time can be changed to accommodate for this.

- Depending on ingestion lag, a look-back time should be increased to accommodate for any potential delay. Using event.ingested rather than the @timestamp field can also alleviate those concerns. 