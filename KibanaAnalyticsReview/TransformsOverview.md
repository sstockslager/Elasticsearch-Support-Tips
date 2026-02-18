# What are Transforms?

A [Transform](https://www.elastic.co/guide/en/elasticsearch/reference/current/transform-overview.html) is at its basic level a method to calculate a metric against an index. This can be useful when a user needs to pre-calculate a metric against a field or set of fields. As an example, Dashboards and Visualizations generate metrics based on their configurations at the time they are opened. Whereas a Transform has already completed the calculation for the metric, making the process that much more efficient. 

There are two types of Transforms available: Pivot and Latest. A _Pivot Transform_ allows for the transform job to take the current data and turn it into _entity-specific data_, or data focused around a specific entity, such as a host name. A _Latest Transform_ copies the most recent documents into a new index, and group that data around one or more fields. 

## Setting up a Transform

The user can locate the Transforms page in Stack Management under the Data section. Several managed Transforms may already exist depending on the integrations that are installed in the user's Elastic environment. 

Creating a Transform is very similar to setting up a Machine Learning job. First choose the Source, or the data to query against, from the list of Data Views and Saved Searches. Afterwards, the main setup page appears to configure the Transform itself. Choose the Transform type, Pivot or Latest, and begin to filter the Source. If the user needs additional help, there is a table of source documents directly below the filter bar. There is a preconfigured table of 10 columns of the first fields available in alphabetical order. The option is available to toggle off the preset columns and choose relevant fields for the Transform instead. This does not affect the Transform as a whole. 

The next step after generating the filter for the Source is to frame the query itself under the Transform Configuration. There are two main parts here; Group By and Aggregation. Group By should be any related fields to the Transforms function, while Aggregation is the calculation itself that the user intends to do. NOTE that certain Group By fields have the option to expand the interval in which that field is grouped during the selected timeframe. Depending on the value, this can still have a direct effect on the values determined from the Aggregation function. For example, if the @timestamp field is one of the Group By fields, and the interval 2 min while the timeframe is 10 min, there is a possibility that the Aggregation will result in 5 separate values for the field that is being aggregated. 

The last step is to fill in the Transform details, such as the name and description. There also is the option to run the Transform in continuous mode and update the retention policy here, as well as indicate the index the Transform results are stored. 

## Advanced Transform Generation

While most Transform creation and management can be handled in the main Transform section under Stack Management. In some instances, Transforms can be generated and reviewed via API under DevTools. 

### Create a Transform via API

Making a Transform from scratch without the UI provided in Stack Management still follows the same path as the user would follow within the UI. Below is an example of a Pivot Transform via DevTools that calculates the difference between two timestamps via added script:
```
PUT _transform/servicenow_analyst_ticket_metrics-mttc
{
    "source": {
        "index": "logs-servicenow-*",
        "query": {
            "bool": {
                "filter": [
                    {
                        "term": {
                            "event.category": {
                                "value": "incident"
                            }
                        }
                    },
                    {
                        "exists": {
                            "field": "servicenow.incident.closed.timestamp"
                        }
                    }
                ]
            }
        }
    },
    "pivot": {
        "group_by": {
            "analyst": {
                "terms": {
                    "field": "servicenow.incident.assigned.user",
                    "field": "servicenow.incident.id"
                }
            }
        },
        "aggregations": {
            "ticket_opened": {
                "min": {
                    "field": "servicenow.incident.opened.timestamp"
                }
            },
            "ticket_closed": {
                "max": {
                    "field": "servicenow.incident.closed.timestamp"
                }
            },
            "time_diff": {
                "bucket_script": {
                    "buckets_path": {
                        "open": "ticket_opened.value",
                        "closed": "ticket_closed.value"
                    },
                    "script": "params.closed - params.open"
                }
            }
        }
    },
    "description": "Calculate the Mean Time to Close (MTTC) based on the time it took for a Security Analyst to be assigned to an Incident and the time the Incident was resolved or closed out.",
    "settings": {
        "num_failure_retries": 30,
        "timeout": "90s"
    }
    "dest": {
        "index":"servicenow_transform_pivot_calculate_mmtc",
        "pipeline": "add_analyst_mttc"
    },
    "frequency": "30m",
    "sync": {
        "time": {
            "field": "@timestamp",
            "delay": "240s"
        }
    },
    "retention_policy": {
        "time": {
            "field": "@timestamp",
            "max_age": "90d"
        }
    }
}
```

NOTE that after the Transform is created, it will not be running. The user will have to go back into DevTools and start the Transform manually with the "_start" command. Once the Transform is generated in DevTools, there is a series of validation checks that is run against the scripted Transform to ensure that it will run properly, but the "_defer_validation" function can be added to the script to skip those checks. Additionally, this specific example where Painless is utilized is a unique use case, and can potentially result in slower search speeds within the Transform.

### Manage the Transform in API

Any user with the correct permissions can use DevTools to pull Transforms statistics and related information via DevTools:
```
GET _transform/logs-ded.pivot_transform-default-2.2.0/_stats
```
It is also possible to request the statistics from more than one Transform at the same time, or from the last Transforms that were processed:
```
GET _transform/ml_hostriskscore_latest_transform_default,ml_userriskscore_latest_transform_default/_stats
```
```
GET _transform/_all/_stats
```
The stats functions will return several points of data about the Transform, including the current state, health, attributes, and metrics about any processed documents completed by the Transform. This will include the number of processed pages, or batches of documents that matched the Transform, as well as the number of processed documents, the number of search failures, amount of time spent indexing results, and the amount of documents deleted from the destination index. NOTE if the response to the API script is a 404 error code, check the inputted command first for errors. 

In addition to _stats, there are many other functions that are available for Transform management. "_update" can be used to change configurations for preexisting Transforms. Some Transforms that are pre-built and Managed by Elastic and can be upgraded using "_upgrade" to a more recent version. Any Transform that is being changed should be stopped first using "_stop" or "_stop?force=true" if issues occurred initially, then restarted with "_start". There are multiple subfunctions that can be used in conjunction with these commands, all of which are available to review under the [Elastic Transform APIs](https://www.elastic.co/guide/en/elasticsearch/reference/current/transform-apis.html) 

## Additional Considerations

* The Transform is NOT making changes to the original index selected. Rather, the results are being copied to a new index. If a destination index is not called out during setup, an index is made using the name of the Transform.

* One of the main concerns with Transforms is not if they can be implemented, but if they should be. At its heart, a Transform will generate a metric based on the information that it has been provided. Using this, a Transform should be formed only as strict and as necessary as it should be. Check this page for guidance on [Elastic Transform Usage](https://www.elastic.co/guide/en/elasticsearch/reference/current/transform-usage.html). Also review this page for additional notation on confirmed [Transform Limitation](https://www.elastic.co/guide/en/elasticsearch/reference/current/transform-limitations.html).

* When making a Transform, know what is to be accomplished beforehand, what dataset that requires, and if that data is available. A Transform is only as capable as the data that it is fed. 

* Filters should be used if possible, they will make your Transform as efficient as possible to save on processing time. If a Transform needs a specific field, make a filter to focus only on documents where that field exists. 

* Before creating the Transform, use the "preview results" function to make sure that the output is what is expected. 

* Initial Transform setup can take awhile to fully activate depending on the size of the dataset and the complexity of the query. 

* Much like Elasticsearch's ILM policy, a Transform also has a retention policy that can be manipulated for how long the resultant documents that are created by the Transform are held before being deleted. 
