# Security Alert Creation and Tuning

## Summary

Creating custom rules are the bedrock of an ever-evolving security cybersecurity posture and key to keeping relevant in an ever-changing environment. While pre-built detections provided by managed software can provide a basic amount of protection, the advanced monitoring team will build upon those pre-built rules and generate their own to protect against unwanted attacks. 

## Detection Logic Considerations

Once an Analyst has an idea of an alert in mind, there are several questions that should be asked to further develop the rule:

1. _What Rule Type would be best?_ Some alerts can be easy to generate, as they are simply looking for a specific event in the logs to occur. Therefore, a KQL query can suffice. However, while a straight KQL query is still a valid alert, sometimes a more complex rule logic is required. For example, a successful password spray attempt might require a complex ES|QL query. Brute forcing against an account can be accomplished via a Threshold rule. More intricate suspicious events might require EQL to link one event followed by another in sequence. Each rule situation can have multiple logical solutions. 

2. _What data source or sources are required?_ Understanding where the logs are coming from is just as important as what the rule is looking for. 

3. _How often will the rule search for results?_ The other half of the rule logic is the rule scheduling. How often will the rule execute to search for matching results? Most default scheduling is set to run every 5 minutes with a look-back time of 1 minute. Depending on the logic implemented, standard environment operation, and Elastic ingestion lag, this may be considered too inefficient; or on the other end, providing monitoring gaps or duplicate alerts.

4. _What fields are required for the rule to function properly?_ Once the analyst knows what data source the rule will query, finding the parsed fields that the hold the event in question is next. The easiest way to accomplish this is through Discover. With the idea of the rule in mind, use a KQL query to filter down the documents until only the required logs are present. During this process, the specific field or fields that hinge on the rule logic will become more apparent as will any potential logic issues that will arise. 

[Elastic Security Blog - Security Engineering Capabilities](https://www.elastic.co/blog/elastic-security-detection-engineering)

### Rule Syntax Best Practices

While there any many ways to go about writing out a rule's logic, not all of them will produce the expectant results. Below are some of the most common rule logic situations that an analyst will come across:

#### Wildcards

Wildcards are a powerful tool available to Discover searches, and they can be a useful ally in discovering a variety of results. However, its use should be as limited as possible, as the use of a wildcard over large datasets or high cardinality fields can have a detrimental effect on search performance. That said, targeted use of a wildcard can be extremely powerful. 

Using a wildcard to pull documents that hold a specific field is useful, but the more common a field, the larger pull on Elastic resources. The example below would be a common way to remove machine accounts out of Windows Security logs:

```
data_stream.dataset: "system.security" AND NOT user.name: "*$"
```

Wildcards can also work on fields as well, if there is a specific subset of fields within a dataset that an analyst wants to focus on. The example below would only pull documents from PaloAlto firewall logs containing URL information. However, while this technically works, there is likely a Palo field that is present for URL events that an analyst can focus on, such as "panw.panos.type: TRAFFIC". 

```
data_stream.dataset: "panw.panos" AND panw.panos.url.*: *
```

Using wildcards where they are the most beneficial can outweigh any potential strain they can cause on processing, such as reviewing file directories. NOTE here the "?" is used as a single-character wildcard. 

```
data_stream.dataset: "endpoint.events.process" AND process.working_directory: "?:\\Users\\Public\\*.exe"
```

#### Parentheses

Utilizing parentheses along with AND/OR statements are oftentimes not crucial for some queries, but they can be critical for controlling the order of query execution and ensuring valid results. Consider this Windows Security Log search:

```
data_stream.dataset: "system.security" AND user.name: "Administrator" AND event.code: "4624" OR event.code: "4672" OR event.code: "5140"
```
Using the query above will give you results for a successful logon (Code 4624) for user name Administrator, as well as every result for Codes 4672 (Special Logon) and 5140 (Network Share Object Accessed). Whether or not any 4672 or 5140 events matches with the user name does not matter here. However, using the parentheses in the query below will show Codes 4614, 4672, and 5140 for only the Administrator account. 

```
data_stream.dataset: "system.security" AND user.name: "Administrator" AND (event.code: "4624" OR event.code: "4672" OR event.code: "5140")
```

Parentheses can also be used to nest multiple values from the same field together, making the query more efficient. The example below reworks the previous query to showcase this:

```
data_stream.dataset: "system.security" AND user.name: "Administrator" AND event.code: ("4624" OR "4672" OR "5140")
```

#### Operational Considerations

Once rule logic has been generated, there are a few additional things to keep in mind from an auditing or engineering perspective:

- As an Analyst, applying an accurate severity score to the alert allows correct prioritization during triage.

- Utilizing alert suppression to reduce duplicate or repeated alerts can be beneficial to overall triage means.

- Each rule execution functions similarly to a query search in Discover; recalls the Index or Data View, then further filters down results by query execution. This provides two opportunities to reduce processing unwanted logs in the query. Specifying the Index Pattern as much as possible as opposed to using the logs-* pattern will reduce overhead. Users can also use custom Data Views instead of Index Patterns for ease of use. The more specific, the faster the results will be. For example, an Index Pattern of logs-azure.activitylogs* is better than logs-azure.*. An analyst could make a rule with the index set as logs-azure.\* and a query with "data_stream.dataset: azure.activitylogs". While this still works, it would not be as efficient as the previous option. 

- Filtering out results for the rule query can either be done as a filter itself or within the query; both are acceptable. However, filters are more efficient than queries. As a rule, filters are used on a binary Yes or No search or searching out for exact values. Filters are also cached data. A query uses a full text search where the results depend on a relevance score. 

#### Machine Learning Considerations

Machine Learning detections are a unique use case, which is make it a distinct point worthy of discussion. Since the ML job itself and the security rule act reliant, yet independent of each other, tuning resultant alerts provides multiple options. 

Alerts from a Machine Learning rule are dependent on the severity that the Job provides a specific processed event. By default, ML rules are set to alert on any event with a calculated severity of above 75 (resolves out to a High Severity Level). An easy solution to this would be to update the severity to a high number; meaning only events that the ML job considered highly anomalous would become an alert.

Alert tuning also becomes a two-fold manner. An analyst can either tune an event from firing out of the rule itself in the Security App under Rule Exceptions, OR the specific event can be filtered out of the Machine Learning Job's Datafeed in Kibana via DSL. This situation becomes a discussion about whether the event in question still warrants being a part of the job, but not seen in Security, or if the event is considered expected activity or a false positive. 

## Tuning Out Events From Alerting

Now that a rule is created and actively firing, additional tuning of the alert can be conducted. Ideally, after reviewing the general initial results in Discover during the configuration period, the analyst understand what might be considered a false positive or expected activity. Otherwise, this can be accomplished by following normal triage procedure until a false positive determination is made. 

Afterwards, tuning can be accomplished through the [Rule Exceptions](https://www.elastic.co/docs/solutions/security/detect-and-alert/rule-exceptions) tab in the Rule panel. These exceptions are events that should not fire an alert, so the NOT is implied in every exception created. If a not is used in an exception, such as "file.extension IS NOT .pdf", then the rule will only fire if the event matches that exception. Series and wildcards can also be utilized in the Rule Exceptions.

### Shared Exception Lists

Some exceptions can be applied across multiple rules. In this case, the Analyst can create a Shared Exception List to apply exceptions across multiple rules simultaneously. 

Commonly, these Shared Exception Lists are grouped with similar exceptions under one List. For example, a List can be made for field values that display administrator activity. This can be easier when large environments follow a similar naming schema for their user accounts. Another List can be made for Service Accounts, Scanning and Vulnerability Software Activity, General Software and Tool Activity, Cloud Activity, Network Activity, etc. 

NOTE that any single Rule Exception that is changed within a rule itself that is linked to a Shared Exception List will change that exception across all linked rules from that List. If that is a concern, then consider making a singular exception that is like the shared exception for that one rule. 