# EQ|QL Walkthrough

[Elasticsearch Query Language](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql.html) allows the user to build upon regular event searching to perform more complex actions, such as statistical analysis, aggregation, and even generate Visualizations. 

There are a few things to keep in mind for first time users:

* ES|QL does have an auto-complete feature to help users complete error-proof commands.

* ES|QL does not work on flattened or nested field types. 

* ES|QL is similar to KQL where initial results are in a Summary view, much like the default view in Discover. Adding a KEEP statement will control the output of what fields are shown.

* The user may notice the following command when they first enter the ES|QL page: "FROM logs-* | LIMIT 10". Breaking this down, this will show the last 10 logs that were ingested into logs-*. While 10 is the default, without this statement explicitly added, the default output, or _upper limit_, is limited to 50 thousand documents and 1 thousand documents for many STAT commands. FROM will specify the data that is to be queried. Any recognized index pattern is valid for the FROM statement. Multiple index patterns can also be specified in a comma-separated list. 

# Coding Examples

## Summary

This section will provide some commonly used examples and statements for ES|QL. The user can also use the link below for syntax options. 

[Elastic ES|QL Syntax Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-language.html)

### Cross-Cluster Query

For environments with multiple clusters, ES|QL allows for the ability to search between different clusters. 
```
FROM _cluster.name_:logs-endpoint.* | LIMIT 10
```
### KEEP Statements

As mentioned above, the default output is similar to Discover, where the documents are in a Summary View. Using the KEEP statement controls what fields are shown after execution. The command below outputs the number of unique users per host, excluding any that are under 5. Then outputs the top 20 results, sorted by the hosts with the most users, showing the source ip, host name, and the number of unique users in a table. 
```
FROM logs-*
| STATS uniqueusers = COUNT(user.name) BY source.ip, host.name
| EVAL userresult = CASE(uniqueusers >= 5, "true")
| SORT uniqueusers DESC
| KEEP source.ip, host.name, uniqueusers
| LIMIT 20
```
### STATS Statements

Other than WHERE statements, STATS (_statistics_) are perhaps the most utilized. A key point to keep in mind is that any value that the user wants in the output __must__ be in the statement. The example below outputs a table of the top 25 highest cpu-consuming processes, with the number formatted back into a percent. NOTE the BUCKET function is used here to control the time range of documents that will be used to calculate the STAT output, rather than determine the value as a whole. This works very similar to how Machine Learning performs its statistical analysis in 15 minute bucket intervals. 
```
FROM metrics-system*
| WHERE process.name IS NOT NULL
| STATS maxcpu = MAX(process.cpu.pct) BY process.name, BUCKET(@timestamp, 1 day)
| EVAL maxcpu = ROUND(maxcpu*100,2)
| SORT maxcpu DESC | WHERE maxcpu IS NOT NULL
| KEEP process.name, maxcpu
| LIMIT 25
```
### MV_EXPAND Statements

Some fields are ingested into Elastic as a multi-field, which means that multiple values are allowed to be within the same field within the same document. This is commonly seen in fields such as process.args or related.ip, where a user would expect there to be multiple parts. However, several ES|QL statements will return null values if a multi-field is used, so MV_EXPAND is used to separate the values into their own column. 

The example below shows the top 100 process commands used by the admin account, sorted by the least common instances, with the process.args field split apart into individual values. NOTE that the user could use "WHERE user.name:"admin"" rather than an additional AND statement, but that feature is currently in technical preview as of version 8.17. 
```
FROM logs-endpoint.events.process*
| WHERE user.name IS NOT NULL
AND user.name=="admin"
| MV_EXPAND process.args
| STATS indvargs = COUNT(process.args) BY process.args 
| SORT indvargs ASC
| KEEP process.args, indvargs
| LIMIT 100
```
### RLIKE Statements

RLIKE is short for _regular expression_, and is how REGEX can be used within an ES|QL command. NOTE that the engine behind the commands are run in Lucene, so this might impact syntax. Here the ES|QL script is reviewing web browser information for Okta logs, specifically searching for a vulnerable version. 
```
FROM logs-* 
| WHERE user_agent.original IS NOT NULL 
AND event.module=="okta"
AND user_agent.original RLIKE `""".*Chrome/128\.[0-9]+\.[0-9]+\.[0-9]+.*"""`
| LIMIT 1000
| KEEP user_agent.original, related.ip, related.user
```
### DISSECT / GROK Statements

GROK Statements are useful in instances where the values that are being searched for are parsed into a separate field or are otherwise not parsed at all. In this example, the user is taking a raw email log, pulling out the email address and action, then enriching the email address with information from the company's AD server, before outputting everything to a table. 
```
FROM logs-*
| GROK event.original `"""(?<email_address>\b[A-Za-z0-9._%+-]+@[A-Za-z0-9._]+\.[A-Za-z]{2,}\b)"""`
| GROK event.original `"""\"title\":\"(?<email.action>[^\"]+)""""`
| WHERE email.action=="Email reported by user as malware or phish"
| ENRICH company_ad ON email_address WITH full_name, _id, department, manager
| KEEP email_address, full_name, _id, department, manager
```
This example uses the DISSECT pattern to pull out data that is mapped together within the same field. 
```
FROM logs-aws.cloudtrail*
| DISSECT aws.cloudtrail.user_identity.arn """arn:%{aws.domain}:%{aws.token.service}::%{cloud.id}:%{aws.user.type}/%{aws.user.name}/%{aws.bucket}"""
| KEEP aws.cloudtrail.user_security.arn, aws.user.name
```
NOTE that a GROK pattern uses regular expressions as opposed to a DISSECT, which will search on a delimiter-based pattern matching. With this in mind, GROK is generally more resource intensive on returning results and processing queries. Which is why when querying an index or via Logstash using ES|QL, it is recommended to use the DISSECT filter over GROK wherever possible. Additionally, it is important to be as specific as possible when indicating the index that is being queried to ensure efficient data retrieval. 

### ENRICH Statements

Enrichment is when data from a separate source is joined to another to provide further contextual information relevant to the task. Here, data is ingested from Azure SignIn logs via an enrichment Transform to link a username to an associated authenticated workstation from Cisco ISE switch logs. NOTE the DISSECT is used to remove the domain from the workstation name, stored in the user.name field in this case and "azure.signinlogs-userenrich-final" is the enrichment policy in question.

```
FROM logs-cisco_ise.log*
| WHERE 'cisco_ise.log.category.name=="CISE_Passed_Authentication"
| WHERE 'cisco_ise.log.auth.policy.matched.rule' IN ("WiFi_Policy","Wrkstn_Policy","IOT_Policy")
| DISSECT user.name """%{azure.signinlogs.properties.device_detail.display.name}.ad.company.com"""
| ENRICH azure.signinlogs-userenrich-final WITH 'azure.signinlogs.properties.user_display_name' 
| LIMIT 2500

```

### Unusual ES|QL Detection Queries and Situations

- The query below is looking for instances where the value in one field matches the value in another. This can very easily be replicated for any sort of data matching. 
```
FROM logs-system.security-windows
| WHERE event.code=="4728" AND winlog.event_data.SubjectUserSid==winlog.event_data.MemberSid
| LIMIT 10
```