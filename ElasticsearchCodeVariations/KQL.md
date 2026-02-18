# Hunting Through Discover

# Summary

Now with usable data being ingested into Discover, the user can begin to hunt through that data. First, the ingested data was filtered by selecting a data view. Now, the user will further refine the document output through the query and filtering.

## KQL

All of the available fields that were created during ingestion can be categorized into two distinct sections: ECS and non-ECS. ECS stands for _Elastic Common Schema_, which is Elasticsearch's uniform field types common across all data sets. However, not all log data can be filled with an ECS field, so there are additional fields that are created that come with specific integrations to accommodate for this. These non-ECS are usually indicated with the data types name in the field itself. For example, Windows data from the Event Log could also be located in the winlog.event_data.\* fields. Also note that in some instances, certain ECS values can equate to multiple, similar logs. For instance, in the Windows Event Logs, an event.action of "added-member-to-group" can reference a winlog.event_id of 4732, 4728, or 4756. As a bonus, the Windows Event Codes that are stored in the winlog.event_id field also copy over to the ECS field of event.code. 

The Search Bar in Discover will be how the user will search for the information they want to find. NOTE that the search bar requests that the query you use to filter for the intended documents is in [KQL](https://www.elastic.co/guide/en/kibana/current/kuery-query.html) syntax (Kibana Query Language). 

### Ask the Right Questions

When running a query through Discover, the user should ask themselves the following questions:

* What question am I asking?
* Where would the information logically be located?
* What fields might possess that information, if any?
* How can I narrow the results to find only data related to my question?

_What question am I asking?_ As an example, the user was asked to discover the employees that had failed to log into their accounts the most over the last week. 

_Where would the information logically be located?_ If the user were to find this information in Discover, the first place they would logically look would be in the Windows Security Logs. Check the data view to ensure that the best option is selected, such as logs-\* or perhaps more specific such as logs-system.security-\*. 

_What fields might possess that information, if any?_ Using the question at hand, the user can formulate a query in KQL using the Search Bar in Discover at the top center of the page. Specify the data type in use, which will filter out any others that may be in the selected Data View. This can be seen in the field "data_stream.dataset" or "event.dataset". Some logs, like ones from the Windows Event Log, possess a unique identifier for the user to locate that event type or action that was done. In this case, the Windows Event Code can be used in the search query to focus results, which is in the "event.code" field. If the code is not known, then a more generic field such as "event.outcome" can be used to satisfy the "failure" portion of the query. Expand your search results to the time frame in question. A valid query to use for the question above would be:

data_stream.dataset: "system.security" AND event.code: "4625"

NOTE that any of the ECS fields listed above all follow the same format. "event.\*" will be any ECS-field that is related to the specific document in question. "host.\*" for any recorded host-based information, and for network-based, look for "network.\*", "source.\*", or "destination.\*". ECS fields will sometimes also nest certain values together so that searches can be formed to focus on that value itself, regardless of when in the timeline the event occurred. These are known as _related fields_. As an example, for events that involve two separate users, such as an admin unlocking a users account, both of those usernames will appear in the "related.user" field. 

_How can I narrow the results to find only data related to my question?_ Now that a query has been generated, determine if the results satisfy the question. Was the query that was used too broad or too specific? If searching for account failures using the Event Code 4625 was accurate, check the results by opening one of the documents below in the bottom center of the Discover page. There may be other events that were recorded by this event code that do not satisfy the question, such as failures from service and machine accounts. 

Keep in mind that depending on how the query is phrased may alert the results. Searching Windows Security Logs using event code 4625 will ONLY yield that event code in Discover. However, searching for event.outcome: "failure" is more broad and will consist of more than one event code that is considered a failure. 

Results that do not fit within the constraint of what the user is asking can be filtered out either by adding "AND NOT" to the KQL query. Or if a specific field is discovered in a document during a search, hovering over that value on the right side of the document will give the option to filter for that value or exclude it from the results. Additionally, hovering over the field name on the left side will give the option to filter for any document where that field exists or to add that field as a column in the center of the Discover page. 

[Elastic Search Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-with-elasticsearch.html)

### Save your Discover View

After the user's work is completed, they have the option to preserve their Discover page if they wish. Often in the course of investigations, a user locates certain fields and column configurations that derives the most benefit for that particular query or report. In the top right, there is a save button that will allow the user to save their search with a title, description, and even Tag for later recall. Once saved, any query in the search bar, filters, or added columns will be saved to be later brought back. To find the Saved Search again, click Open in the top right corner and search for the title. Many Integrations also come with their own set of Saved Searches to guide the user in locating specific subsets of data in the logs. 

If the user wishes to restart their search with a clean slate of Discover, without any filters or KQL query, using the plus (+) icon at the top right corner will reset the Discover view back to non-filtered state.

NOTE that Tags are uniform throughout ElasticSearch, and are a quick way to locate closely-related objects. A Tag can be added to any custom Object in the Security, Observability, or Analytics stack. A good recommendation to easily locate any custom-made content is to add a uniform Tag to the Object for easier search, such as a Tag named "Operations" or "\[CompanyName\]". 

### Additional Considerations

* Elasticsearch can be reconfigured from default settings to change the default landing page when a user logs into Elastic. There is always a homepage that indicates which Stacks, such as Analytics, Security, Observability, or Search, is available. This is usually the landing page, but this can be updated for the user to log into Discover or a Dashboard instead. 

* While non-ECS fields are available to help parse out the logs, not everything is given a field. Anything missed by the ingest pipeline will be left in a keyword field called "message". 

* If a user is not sure what fields or values are available in the selected data view, begin typing in the KQL search bar, and the most relevant results will begin to appear in __alphabetical order__. For example, if a user typed in event.dataset, then the results will offer any data sets that are available. This can be a good way to see at a glance the variety of options that can be made, as well as aid the user in typing the query in a format that will guarantee results. The search bar will also keep a history of the user's previous related search attempts during the course of the user's Elastic login session. 

* Wildcards are a powerful tool when searching with fields. They provide the user with the opportunity to determine if a field exists in any document, such as user.name: \*. Or the wildcard can be used uncover any variation of the value within a field, much like user.name: "svc_\*". In certain cases, a wildcard can even be utilized to locate a value in a variety of similar fields, such as url.\* : "google.com". However, keep in mind that the use of a wildcard is very resource intensive and can result in slower return times in a user's queries. 

* It does not matter if filtering for a field value is done in the KQL search bar or by adding a filter. Provided it is the same value, the result will be the same amount of documents. 

* Some fields may show that there is a "field conflict", which has a tendency of impacting Discover searches and creating Visualizations. A conflict usually occurs when that field is in multiple indices, but is stored as a different value type between each index. As an example, a Cisco datastream may save an IP address in the source.ip field as an IP field type, but the Winlog datastream may save its source.ip as a keyword. Fixing this will require some dynamic mapping or re-indexing, but a quick solution can be to be as specific as possible when naming the index in the data view. 

* Using a field is not required in the search bar. The user can simply type in a text value and Kibana will return any document where that text is seen. However, this is resource intensive and it is recommended that quotations are used. 
