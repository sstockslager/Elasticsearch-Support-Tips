# Kibana's Machine Learning Walkthrough

[Machine Learning](https://www.elastic.co/what-is/machine-learning), commonly referred to as _ML_, is considered a branch of AI that uses complex algorithms to map out large datasets, identify patterns and outliers, and eventually make predictions on new data. 

ML is considered a type of _data frame analytics_, which runs continuous calculations to identify unusual points in data and how closely they are related to other points of data around it. Elastic does this by separating the ML capabilities into two main functions: the Datafeed and the Job.

The datafeed describes the ingested datasets that the ML algorithms will be run against. Any matching datasets in the datafeed is broken up into segments called _buckets_ which is what the ML job will be run against.

The job refers to the actual calculations that are being performed to identify the patterns and outliers in the datafeed. The calculations can be performed on both the individual buckets as well as the datafeed as a whole for predictive analysis. 

## Setting Up a Job

The [Machine Learning](https://www.elastic.co/guide/en/machine-learning/current/machine-learning-intro.html) section in the Analytics stack provides an Overview page when the user first enters the section of Kibana. The Overview page will describe the current Node memory usage and any hits from ML jobs organized by Group ID. ML jobs are very resource intensive, so Elastic designates a separate section of the allocated memory allotted for initial data ingestion of the entire Elastic cluster to it. A default size for a ML node is usually around 5 GB, but depending on your environment, more memory may be required. And many jobs also require a minimal amount of memory allocated to run properly. 

Group IDs are similar to Tags, and are a useful way to organize ML jobs by type. There are two main categories that are used in Machine Learning, Anomaly Detection and Data Frame Analytics. Anomaly Detection uses time series data to uncover expected behavior in ingested data to accurately pinpoint unexpected events. This will be the most commonly used ML job type. Data Frame Analytics can perform more unique analyses and provide predictive models between data points. 

Creating a Job is a simple process. The user first selects the data in question, either via a data view or saved search. Then the user selects the ML job that is to be performed on the data. There are several pre-configured options available or the user can choose to make a custom one via a Wizard. If using the pre-built jobs, all the configuration has already been completed by Elastic, so all that is left is to choose which of the jobs to install and activate. 

If the user chooses the Wizard, the next step will be to select the time range in which to initially feed the job. This will be the range of the data view that was previously selected. NOTE that while a default range will be selected, it is recommended that around 1 to 2 weeks of data be given to the job so the results from the algorithm are more accurate and understand the user's environment better. 

Next, the user will need to choose the field(s) that the job should focus its calculations on. This will be the same page where the user can choose to ignore empty buckets and the time span interval for each bucket. NOTE that usually the default 15 minute bucket span is sufficient. 

The last step before validation is adding the job details such as a name, description, and any related group IDs. Also check the Advanced tab for additional settings to configure. In this section, take note of the default Model Memory Limit. This is an estimated amount of memory that is thought to be needed for the job, but may require to be changed later. The option to also store the job results in a dedicated index rather than the default ml index is also provided, as well as having the job ignore unavailable indices during execution. NOTE that if this job is meant to be part of a security detection for alerting purposes, that the "security" group ID is required. Otherwise, when creating the alert in the Security Stack, the job that references the results will not appear. 

## Customize and Tune

Any job that was made can be seen in the "Jobs" tab in Machine Learning. The page will give the job name, description, group IDs, the number of documents that were processed by that job, the state of the memory, job, datafeed, and the current timestamp the job is running on. Before any created ML job is changed, ensure that the job itself and the datafeed are both deactivated. 

When you edit the job, there are two main tabs to focus on: the Job Details and the Datafeed. After the job is created, the user cannot edit much of the job logic itself, but the Job Details section will allow the user to change the Model Memory and the snapshot settings. The Datafeed tab will allow the user to edit the query delay and scroll size settings as well as the datafeed query itself. This query is run against the dataview that makes up the datafeed and is used to further filter the datafeed to only documents that fit the job. The datafeed query itself is scripted in [Query DSL](/ElasticsearchCodeVariations/DSL.md) and is the key way to further filter out wanted noise or documents that the user does not want included in the job results. 

## Advanced Machine Learning Job Creation

Most Machine Learning can handled or otherwise managed from the UI in Kibana. But in some cases, ML jobs can be created or audited via DevTools in Stack Management. 

### Creating an ML Job via API

#### Estimate Job Memory

Before a user creates the ML job, there are two things that should be determined: what the job is to accomplish, and how much memory will it take. Luckily, there is a command in DevTools that will help the user [estimate how much memory](https://www.elastic.co/guide/en/elasticsearch/reference/current/ml-estimate-model-memory.html) a job will take to function properly in the environment. For instance, the command below will estimate the amount of memory for a job that monitors for a spike in the error.message field:
```
POST _ml/anomaly_detectors/_estimate_model_memory
{
  "analysis_config": {
    "bucket_span": "15m",
    "detectors": [
      {
        "function": "high_count",
        "partition_field_name": "data_stream.dataset",
        "detector_index": 0
      }
    ],
    "influencers": ["data_stream.dataset","agent.name"]
  },
  "datafeed_config": {
    "indices": ["logs-*],
    "query": {
      "bool": {
        "filter": [
          {
            "field": "error.message"
          }
        ]
      }
    }
  }
  "overall_cardinality": {
    "data_stream.dataset": 15
  },
  "max_bucket_cardinality": {
    "data_stream.dataset": 15,
    "agent.name": 2500
  }
}
```

NOTE that "max_bucket_cardinality" is meant to estimate the amount of unique values for the influencers that are provided in the job. The same for "overall_cardinality", but instead of influencers, it will be for the fields indicated in the "partition_field_name", "by_field_name", or "over_field_name". 

#### Scripting Out the Job

The below example would consist of a multi-metric anomaly job designed to monitor for low counts of documents in a dataset. NOTE that there are many subfunctions that are available when generating an ML job, all of which are available in the [Elastic Job Creation via API](https://www.elastic.co/guide/en/elasticsearch/reference/current/ml-put-job.html) page.

```
PUT _ml/anomaly_detectors/logs-ingestion-monitoring
{
  "analysis_config": {
    "bucket_span": "30m":,
    "detectors": [
      {
        "function": "low_count",
        "partition_field_name": "data_stream.dataset"
      }
    ],
    "influencers": ["data_stream.dataset", "agent.hostname"],
    "model_prune_window": "30d"
  },
  "data_description": {
    "time_field": "event.ingested",
    "time_format": "epoch_ms"
  },
  "analysis_limits": {
    "model_memory_limit": "15mb"
  },
  "model_plot_config": {
    "enabled": true,
    "annotations_enabled": true
  },
  "datafeed_config": {
    "indices": [logs-*],
    "query": {
      "bool": {
        "must": [
          {
            "must_all": {}
          }
        ]
      }
    },
    "datafeed_id": "datafeed-logs-ingestion-monitoring"
  },
  "description": "Detects low counts in the logs-* index. This job is influenced by hostname and dataset.",
  "groups": ["security", "monitoring", "custom"]
}
```

### Manage an ML Job via API

#### Metrics and Statistics 

Any user with the required ML privileges can use DevTools can pull any configuration data for any generated ML job. Below is an example of pulling that information from one or multiple ML jobs:
```
GET _ml/anomaly_detectors/v3_windows_rare_user_type10_remote_login
```
```
GET _ml/anomaly_detectors/auth_rare_user,auth_rare_source_ip_for_a_user
```

NOTE the example below will pull the configuration information for all jobs. In this case, specifically for Data Frame specific Jobs, but this can also be used for Anomaly Detection Jobs as well.
```
GET _ml/data_frame/analytics/_all
```
The user can also pull statistical data on the job or datafeed itself via DevTools:
'''
GET _ml/anomaly_detectors/ded_high_sent_bytes_destination_geo_country_iso_code/_stats
'''
'''
GET _ml/datafeeds/datafeed-ded_high_sent_bytes_destination_geo_country_iso_code
'''

To pull metrics on the ML node, such as memory, jvm heap, cluster information, and permissions, use the following command in DevTools:
```
GET _ml/memory/_stats?human
```
There are a variety of subfunctions that are available to use for Machine Learning Jobs, all of which are available to review under the [Elastic Machine Learning APIs](https://www.elastic.co/guide/en/elasticsearch/reference/current/ml-ad-apis.html).

#### Job Filters, Updates, and Maintenance

Most Jobs will automatically delete job results, forecast results, and model snapshots after they have exceeded the retention period. However, a user can manually force a purge of unused data with the "_delete_expired_data" command.
```
DELETE _ml/_delete_expired_data/high_count_network_events
```

Occasionally, the datafeed may fail for an unknown reason. Either the datafeed failed to query the index or there was an unexpected error. If this is the case, stop the job and datafeed and input the code below into DevTools to change a few settings to the datafeed. If this resolves the errors, then the failure was likely caused to the job attempting to read an index that was not available for processing. 
```
POST _ml/datafeeds/datafeed-\[_datafeed name_]/_update
    {
    "indices_options":{
      "ignore_unavailable": true
     },
     "delayed_data_check_config": {
        "enabled": false
      }
    }
```

## Additional Considerations

* For most custom Anomaly Detection jobs, choosing either the Single or Multi-Metric job type will be sufficient. There are additional [Machine Learning Job Types](https://www.elastic.co/guide/en/machine-learning/current/ml-anomaly-detection-job-types.html) available, such as Rare, Geo, and Advanced. But the most commonly used types will be the Single and Multi-Metric. 

* Some metrics within the Job's User Interface, such as the memory size of each model, are usually rounded to the nearest megabyte. In reviewing the Job via JSON or in DevTools, the user can see the storage metrics in bytes, which can allow for more exact memory allocation if space is a concern.

* In the Jobs tab, the user may notice a yellow or red triangle next to the job name. This will indicate that the job has thrown a warning or error. Hovering over the triangle will give the latest notification details. Or the user can click the drop-down into the job and move to the Job Messages tab. Usually, the message will indicate that the datafeed has missed a number of documents due to ingest latency. To fix this, try to increase the query delay or bucket span in the datafeed first before increasing the model memory allocated for the job.

* With any ML job, only the selected time frame is initially provided for the job to run on. After the job processes the documents in that time frame, the job itself will close out automatically. In order to fully automate, the job will need to be restarted with a Search End Time of "No End Time (Real-Time Search)" so the job can continuously run as new data comes in. 

* The Settings section of ML contains a useful option to filter out events that the user may not want to have included in analysis. There is also the option to filter out scheduled events, like holidays or system maintenance. 
