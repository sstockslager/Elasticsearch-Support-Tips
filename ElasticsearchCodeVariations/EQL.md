# EQL Walkthrough

[EQL](https://www.elastic.co/guide/en/elasticsearch/reference/current/eql.html), or _Event Query Language_, was designed as a language to view correlation events on time series data. For those familiar with other coding types, EQL reads and writes very similar to basic SQL queries, but was created specifically for threat hunting. EQL is very versatile, but it does require the @timestamp and a value from the event.category field in order to function properly. Keep in mind that for EQL, order matters, so the list of fields in the query should be _sequenced_ in the order that the user would like them to appear. 

# Coding Examples

## Summary

The section below will include examples of EQL and demonstrate the power behind the capability of correlation. The user can also use the link below for additional syntax examples. 

[EQL Syntax Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/eql-syntax.html)

### Basic EQL Search

A simple EQL search that does not involve correlation usually follows the basic format of _category_ where _condition_. As in the example below, the EQL search is looking for any file where the file size is above 1 GB and does not have the following extension. NOTE that multiple instances and be pulled together in parentheses rather than repeating the condition. EQL is also usually case sensitive, so the '~' after the function is used for case-insensitive matching. 
```
file where file.size >= 1000000000 and file.extension not in~ ("log", "exe", "msi", "conf*")
```
### Order-Specific EQL

This example shows an example of searching for unusual files generated with successful network requests. The sequence below is in line with detecting instances of the vulnerability depicted in CVE-2021-40444. 
```
sequence by host.hostname with maxspan=30s
 [network where http.request.method : "GET"]
 [network where http.response.status_code == 200]
 [file where file.extension : "html"]
 [network where http.request.method : "GET"]
 [network where http.response.status_code == 200]
 [file where file.extension : "cab"]
```
### Searching for a Process Tree

This EQL instance searches for a singular process and its sub-executions containing specific files. The correlation rule will output the entire process tree until the file execution ends. 
```
sequence by process.pid with maxspan=3m
 [process where process.name == 'explorer.exe']
 [library where stringContains(file.name, xzy132.dll)]
 until [process where event.type == 'end']
```
### Brute Force an Account

The example below shows a method of reviewing Windows Security events to discover when there is a series of multiple failed login attempts followed by a success. Here the example specifies that all events occur within 5 seconds and that there is 5 failures before the success occurs. This example also shows how to filter certain logs out of sequence that include certain user names and source ips. 
```
sequence by winlog.computer_name, source.ip with maxspan=5s
  [authentication where event.action == "logon-failed" and
    winlog.logon.type : "Network" and user.id != null and 
    source.ip != null and source.ip != "127.0.0.1" and source.ip != "::1" and 
    not user.name : ("ANONYMOUS LOGON", "-", "*$") and not user.domain == "NT AUTHORITY"] with runs=5
  [authentication where event.action == "logged-in" and
    winlog.logon.type : "Network" and
    source.ip != null and source.ip != "127.0.0.1" and source.ip != "::1" and
    not user.name : ("ANONYMOUS LOGON", "-", "*$") and not user.domain == "NT AUTHORITY"]
```
### Piping With EQL

Pipes are not usually used in EQL, but they can be helpful in hunting for a series of the same result. For instance, tail will return a matching number of events or sequences, starting with the most recent by time. 
```
network where source.ip == '95.97.128.12'
| tail 10
```