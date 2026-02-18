# DSL Walkthrough

As a rule, [DSL](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html), or _Domain Specific Language_, is used in several places in Elastic, including Machine Learning. But one of the more useful moments is as a filter, especially where values are not easily defined within a document. 

DSL can be broken up into two separate types; the _query_ and the _filter_. Query type is often seen when trying to generate a filter in the Analytics or Security Stack without using the KQL search bar. The most common place the filter type is seen is in places like Machine Learning, where it is used to specify the content in the datafeed. 

# Coding Examples

## Summary

This section will provide examples of some of the most commonly used queries that need DSL. Check the link below for syntax options. The user is also able to test out DSL queries in Dev Tools. Any DSL query box will also recursively check for syntax errors, otherwise known as _linting_, as the user types out the command. 

[Elastic DSL Syntax Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/regexp-syntax.html)

### Basic Query

The Query command can function similar to a basic KQL syntax query. This is useful when some functions are needed on fields that are not fully available. The example below shows a method of blending two wildcards together. 
```
{
    "query_string": {
        "query": "event.dataset: system.security AND host.name: (test* OR dev*)"
    }
}
```
### Wildcards

Forming a wildcard is an essential function when there are values within the document with varying values.
```
{
    "query": {
        "wildcard": {
            "file.name": {
                "value": "scan_*.txt"
            }
        }
    }
}
```
### Fuzzy Query

A Fuzzy query returns results are similar, but not exact, to the value in the query itself. After the value in the script, the option is available to add several parameters to control the results.

- _Prefix\_length_ - the number of characters at the beginning left unchanged
- _fuzziness_ - the variation ability allowed for matching results
- _max\_expansions_ - the number of results returned from the query
```
{
    "query": {
        "fuzzy": "
        "url.original": {
            "value": "google.com",
            "prefix_length": 2
        }
    }
}
```
### Filter

Where a query looks for documents that relatively match the syntax provided, a filter is more geared towards a true or false statement. Here is an example of filtering for any firewall document with a specific PaloAlto field present. All other documents will be ignored. 
```
{
    "query": {
        "bool": {
            "filter": [
                {
                    "term": {
                        "observer.type": "firewall"
                    }
                },
                {
                    "exists": {
                        "field": "panw.panos.ruleset"
                    }
                }
            ]
        }
    }
}
```
### Must Not

A Must Not filter translates to filtering out all values that are specified. In the example below, the use case shows how to filter out entire IP ranges by CIDR notation. NOTE that terms values are placed in brackets, and every if the query calls for multiple values for the same field, a comma is placed between each value. 
```
{
  "bool": {
    "must_not": [
      {
        "terms": {
          "source.ip": [
            "192.168.1.0/16",
            "200.201.202.0/32"
          ]
        }
      }
    ]
  }
}
```
### Must

Rather than excluding documents with certain terms, the Must filter looks only for documents that fit the criteria specified. This this example, the syntax combines using a single term search and a wildcard function together. 
```
{
  "bool": {
    "must": [
      {
        "term": {
          "event.category": "network"
        }
      },
      {
        "wildcard": {
          "user.name": {
            "value": "adm-*"
          }
        }
      }
    ]
  }
}
```