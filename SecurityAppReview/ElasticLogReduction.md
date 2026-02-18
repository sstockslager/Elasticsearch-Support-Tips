# Log Reduction Methods

Now that logs are successfully being ingested into Elastic, and parsing is complete, an additional step to take would be to review what exactly is being ingested. Likely there will be several documents that do not have to be ingested. Either the documents do not have a security use-case or there is a subset within the logs that can be whitelisted as expected activity. There are two major ways to accomplish this; reviewing the Integration Policies and Event Filtering. 

## Integration Policies

Many of the Elastic Agents on endpoints have an Integration Policy applied to them, which will streamline the process of data parsing and ingestion. Any available Integration comes with pre-built content, documentation, and default configurations for data ingestion and storage. During setup, the Integration Policy's settings were configured to read and pull in any logs that fit the criteria that matched the settings. 

For instance, the System Integration is used to pull in logs from the Windows Event Collector. If that Integration has the Security section activated, the policy is applied to the Agent on the endpoint to pull logs matching Windows Security Events. The sections themselves in each Integration are normally configured to pull specific events, but they can be updated afterwards to pull new events or remove pre-configured ones. Other Integrations may simply ask the file path where log files themselves are stored, or the URL that can be used to pull the logs from. 

NOTE when editing Integration Policies:

* Any value that is being looked for, such as an Event ID, can be included in a range, but each value must be in a comma-separated, or _CSV_, list.

* Each CSV list is limited to 22 separate clauses. Anymore, and the integration may ignore other clauses added after the 22nd. 

* Multiple file paths can be submitted for Integrations that require them, which can provide an opportunity to limit the amount of log files that are pulled via the Integration. 

* Every Integration Policy provides the option to preserve the original event, or a copy of the raw log, saved in the event.original field. This option can absorb more storage in the Elastic cluster, so it is best practice to only use this option when actively investigating parsing issues or log reduction methods.

[Elastic Integration Reference](https://www.elastic.co/guide/en/fleet/current/integrations.html)

### Managing Integration Policies

A small note here about Integrations. Similar to how Elasticsearch and the Security Stack continually receive version updates and patch releases; Elastic Agent Integrations often receive [periodic updates](https://www.elastic.co/docs/reference/fleet/upgrade-integration) as well. Common practice recommends keeping close to the latest version of Elastic software, barring specific use cases. Usually, these updates are to introduce a new feature or provide additional parsing capabilities, but occasionally will provide minute changes to the ingestion that can affect overall log storage levels. Larger updates may also alter log formats, which can in turn generate conflicts or duplicate logs. Some new features within the source data and Integration may also cause it to collect more detailed log events or events that were previously disabled, resulting in higher log volume. All of these should be kept in mind before updating an Integration, which is why reviewing the release notes is important before deploying updates. 

## Determine What Events to Remove

Before actively removing log events from ingestion into Elasticsearch, there are several considerations that should be examined. 

First, it should be noted that there is a difference between what CAN being ingested into Elastic and what SHOULD be ingested. 

## Filtering out Event Logs

Event Filtering is a useful tool in removing endpoint events that the user does not want to monitor in Elastic. Where an Integration Policy _specifies_ the events the user wishes to ingest, the Event Filter capability will _block_ specific events from being stored in Elastic. Any Integration settings are stored in the Stack Management part of Elastic, but the Event Filter capability is stored in the Security Stack. 

NOTE that the Event Filter option is primarily to be used for endpoints running Elastic Defend. Other endpoints events being captured outside of the Defend Integration may not be filtered out with the same level of granularity.

[Optimize Elastic Defend Reference](https://www.elastic.co/guide/en/security/current/endpoint-artifacts.html)

### Event Filters

The Event Filter section is the main are that the Elastic user would use to remove endpoint events that does not need to be stored into Elastic. 

To create a policy, go to the Security Stack, and under Manage, choose Event Filters. Choose the field and values that are to be excluded. This will work similarly to how logs are filtered out of detections. Once the filter is chosen, select which policy this filter should be applied to. 

NOTE that by default, any applied event filter is recognized globally across ALL hosts. However, the option is available to apply to filter to specific Integration Policies instead. Any events that are being filtered are still being monitored on the endpoint itself, and the Defend policy can still detect and take an action on events but is not being written to the Elastic cluster. Event Filters that are applied will not remove any previously recorded documents that match the filter.

[Event Filters Reference](https://www.elastic.co/guide/en/security/current/event-filters.html)

### Trusted Applications

The Trusted Application section was made to whitelist processes that Elastic does not need to monitor. This would be useful for certain software, such as endpoint monitoring and antivirus tools, where Elastic does not need to report on. To use this section, whole file paths can be excluded, or certain processes by file name or hash value. Applications can also be removed from whitelisting the certificate itself. 

Keep in mind that doing this will not generate any events for the application in question, so no alerts will be made for the application as well, creating an intentional blind spot. Users will need to accept this risk before applying the filter. As applications can also comprise of multiple running processes and files, determining how to fully whitelist an application from Elastic can result in some trial and error before fully removing it from ingestion. 

[Trusted Applications Reference](https://www.elastic.co/guide/en/security/current/trusted-apps-ov.html)
