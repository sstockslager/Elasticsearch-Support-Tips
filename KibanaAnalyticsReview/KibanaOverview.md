# Kibana Analytics

## Summary

This document will provide a guide to utilizing the [Analytics solution](https://www.elastic.co/guide/en/kibana/current/introduction.html#visualize-and-analyze) within [Kibana](https://www.elastic.co/guide/en/kibana/current/introduction.html). By the end of this guide, the viewer will be able to effectively search through ingested data to locate relevant information for the task at hand. 

NOTE that while some definitions for terms will be provided, this guide does assume some prior experience with Elasticsearch and familiarity with its components. Technical experience and understanding of technical jargon is also assumed. 

## Analytics Features

While there are several menu sections, which are also known as solutions, available in [Kibana](https://www.elastic.co/guide/en/kibana/current/introduction.html), this document will focus on the [Kibana Analytics Solution](https://www.elastic.co/guide/en/kibana/current/introduction.html#visualize-and-analyze). In the top left section of the screen, you will the menu selection, or "hamburger". The Analytics solution is usually the first on the list. The pages, which as also known as features, of this solution are used to examine data already ingested into Elasticsearch. There are several features within Analytics, including:

* __Discover__ - Search for individual documents, which usually represent parsed log events. 
* __Dashboards__ - Review multiple visualizations, including graphs, at once.
* __Canvas__ - Take your Dashboards and further personalize them in a more artistic or personalized rendering. 
* __Maps__ - The [Elastic Maps Service (EMS)](https://www.elastic.co/guide/en/kibana/current/maps-connect-to-ems.html) will allow the user to visualize the geolocational data ingested from network-based sources. 
* __Machine Learning__ - Actualize, run calculations, and predict trends on your ingested data. 
* __Graph__ - Summarizing relationships between various data points in a graphical format.
* __Visualize Library__ - Location of individual Lenses not generated within a Dashboard, such as stand-alone tables or pie charts. 

## Discover

### Summary

Raw logs that are brought into Elasticsearch from various sources such as a TCP/UDP connection, Syslog, S3 buckets, etc., are typically handled by Logstash, which is a server-side application that processes and forwards log data. Once brought in, the data is parsed through to map to structured fields, commonly known as Elastic Common Schema, or ECS. Once the logs are parsed, they are stored into an Elastic index and can be searched and analyzed using Elastic's Discover feature. 

### Functionality

A single log, or _document_, such as a log event, is displayed in its entirety in Discover. This can be an extremely large number of documents; and impossible to review document by document. Every function of Discover is designed to help reduce the document count until only the documents relevant to the inputted query are displayed.  

### Data Views

In the top left corner, there is the Data View selector. Data views allow the user to carefully select specific indices to query in Discover; excluding those not currently of importance to the task in mind. There are a series of preconfigured data views that can be used, including one that has all potential data views selected. However, to save processing time, it is recommended that the user select a data view as close to the data stream in mind as possible. If there is not a data view present that fits the use case for the user's query, then the option to create their own data view is available. For example, if the user knows they are searching for a specific file recorded from Elastic Defend's endpoint tool, then the user can specify a data view with the index "logs-endpoint.event.file-\*". NOTE that the wildcard here is required, as it will allow the user to search for all variations of that index, including ones that have rolled over, within the specified time frame. If the user does not know the data stream under which the specific file was recorded from Elastic Defend, then the user could specify a data view of "logs-endpoint.event.\*". 

A common default data view to have is "logs-\*", which holds all the ingested data types in one data view. The user can also specify two specific data streams in the same data view, which could be beneficial for finding examples of correlation between the data streams and pivot points to further an investigation. An example of this would be combining a host-based and network-based data stream together in the same data view, such as "logs-system.security-\*" AND "logs-panw.panos-\*". NOTE that to do this, when making the data view, each index pattern would need to be separated by a comma. If done correctly, the user should see a green message indicating their index pattern matched two sources. 

### Fields

Directly underneath the data view selection is the list of available fields, which are categorized into four main sections. _Popular Fields_, which will show the top field or fields that is the most present in the documents shown. _Available Fields_ will show all the fields that are used in the current data view in alphabetical order. _Empty Fields_ is also shown, but indicates the remaining list of fields that are available, but do not hold any values based on the selected data view and filters. The last one to note is _Selected Fields_, which appears at the top if a user chooses to add the field as a column in the main view. Any field can also be searched for as well if the user knows the field name. 

Also notice the icons next to every field. This indicates what kind of data is within that field. During ingestion, many of the logs when broken down locate pieces of the data that are not simple text. Elastic will note that and designate a specific [field type](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html) for that information, such as a Date, IP Address, or Number. Most of the time, the rest of the log is placed in a field with a type of Keyword. 

Any field in this section can also be clicked on, allowing the user to see a brief overview of the Top Values seen in that field. NOTE that the percentages here are an approximation taken from a sample size of the filtered documents. The user can also filter for the field present in any documents, filter for/out any of the top values shown, and add the field as a column in the main view. At the bottom of the popout, the user can also pivot to see the field statistics in the Visualize Library. 

### Filters

In the top center of the Discover page is heart of the Discover search feature. Here is there the user will primarily search and filter their data. At the top is the search bar, where a user will initially query the data view selected. Below is a horizontal bar chart indicating the rate of logs over the indicated time. This can also be hidden from view or further broken down by another field. Users can also pivot to the Visualize Library and manipulate this chart by clicking the magnifying glass off to the right. 

The last main section here is the time selection to limit the queries results to a specific time frame. By default, the selection is "Today", which only indicates the amount of time passed from 00:00 UTC for the current date. Click the calendar to view the various preset time selections, or the middle of the bar to manually input a specific timeframe. The outlook should look very similar to the time function, with the added addition of being able to configure an auto-refresh in the case that newer data will come in. 

### Search Queries

There are multiple search languages available to query documents in Analytics: 

* [KQL](/ElasticCodeVariations/KQL.md) - Kibana Query Language
* [DSL](/ElasticCodeVariations/DSL.md) - Domain-Specific Language
* [EQL](/ElasticCodeVariations/EQL.md) - Event Query Language
* [ES|QL](/ElasticCodeVariations/ESQL.md) - Elasticsearch Query Language
