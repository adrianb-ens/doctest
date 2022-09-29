---
title: Module 2 Sub 1
description: Module 2 Sub 1 Description
---

# Module 2 Sub 1

Prosciutto ground round drumstick rump pork chop turducken tongue tail turkey porchetta beef ribs ribeye chislic filet mignon shoulder. Pork belly beef ribs burgdoggen, kevin meatloaf turducken tri-tip pork chop t-bone. Filet mignon andouille beef ribs, chicken boudin spare ribs porchetta. Landjaeger doner sausage cow beef, pig ground round jowl pork belly bacon hamburger turkey buffalo ham hock venison.

# Chapter 6: Database Timestamp Connector

For Oracle, SQL Server, MySQL, MariaDB, and Salesforce backends (using the RadiantOne JDBC driver), a timestamp-based change detection mechanism is available. To leverage this mechanism, your database table must have a column that contains a timestamp/date value associated with updates. For Salesforce, this column is LastModifiedDate. The column used in the timestamp connector must be indexed for performance.

>[!important]
>This connector type does not detect delete operations. If you have a need to detect and propagate delete operations from the database, you should choose a different connector type. However, for Salesforce backends, delete operations can be detected because a delete operation is detected when the `isActive` attribute is set to `false`.
>This connector type does not differentiate between `ADD` and `UPDATE` operations. All events are processed as `UPDATE` operations. You can customize the transformation logic to dictate how the `UPDATE` operations should be handled on the target (e.g. translated into an `INSERT`…etc.).

For each database object that is a publisher of changes, a new/changed row in the table must have a timestamp column associated with it.

## Supported Database Date Types

The timestamp connector has been validated against Oracle, SQL Server, MySQL, and MariaDB databases, and Salesforce only (when accessed using the RadiantOne Salesforce JDBC driver). The timestamp connector time stamp mode supports the following database date types:

- For Oracle DB: `TIMESTAMP`, `TIMESTAMP WITH TIME ZONE`, `TIMESTAMP WITH LOCAL TIME ZONE`, and `DATE`.

- For SQL Server: `SMALLDATETIME`, `DATETIME`, and `DATETIME2`

- For Salesforce (using RadiantOne Salesforce JDBC driver): `LastModifiedDate`

- For MySQL or MariaDB: `DATETIME` is preferable, but `TIMESTAMP` can also be used. `DATETIME`, `DATETIME(3)`, or `DATETIME(6)` can be used. `DATETIME(7)` is not supported.

## Database Connector Failover

The database connector failover functionality is described in [Chapter 5](#database-connector-failover).

## Configuration

This connector type can be used for detecting changes in Oracle, SQL Server, MySQL, and MariaDB in addition to Salesforce (when the RadiantOne JDBC driver for Salesforce is used).

To detect changes using a timestamp, set the connector type in the pipeline configuration by clicking the **Capture** component. In the Core Properties section, select DB Timestamp from the drop-down list.

![An image showing the drop-down list for Connector Type with "DB Timestamp" selected, in the Core Properties section of Configure Pipeline](media/image18.png)

Figure 6. 1: DB Timestamp Connector Type

After the DB Timestamp connector type has been selected, configure the properties in the Core Properties, Advanced Properties, and [Event Contents](02-configuring-connector-types-and-properties.md#event-contents) sections. For properties common to all connectors, see [Chapter 2](02-configuring-connector-types-and-properties.md#common-properties-for-all-connectors).

### Timestamp Column

In the Core Properties section, set the Timestamp Column. The value of this property should indicate the exact database column name in your database table that contains either a date/timestamp or a sequence number that indicates when a record has been modified. The value of this column is used by the connector to determine which rows have been modified since the last time it picked up changes from the table.

>[!note]
>For detecting changes in Salesforce, the column name should be LastModifiedDate.

If an invalid column name is configured, the connector stops.

![An image showing the Timestamp Column property value in the Core Properties section, which has been set as "CHANGETIME"](media/image19.png)

Figure 6. 2: Core Properties for DB Timestamp Connector

### Processing Delay

This property can be used if there is a need for a delay in the processing of changes by the connector. For example, if there are two or more processes that update the source table at the same time, and they take about 2 minutes each to run, the processing delay can be set at anywhere between 4-6 minutes. This delays the processing and makes sure the connector captures all changes coming from both processes.

The property value is in seconds, and the default value is `0`. If the value is set to `<=0`, it means that the property is ignored by the connector. If the value is set to be more than `0`, then only the entries between `last_cursor` and `current_time – processing_delay` are processed by the connector. If an invalid, non-numeric value is set for this property, the connector stops.

### SQL Filter

SQL filter is used as part of the request for entries captured by the timestamp connector. Only changes that match the filter are published by the connector.

A SQL filter is either a single expression or several single expressions joined by binary operators and brackets ( ). Possible binary operators are:

- `AND`

- `OR`

- `NOT`

Some examples of valid SQL Filters are:

- `ID='5'`

- `ID='5' AND NAME='ALLEN'`

- `ID='5' AND NAME='ALLEN' OR CITY='SAN FRANCISCO'`

- `ID='5' AND (NAME='ALLEN' OR CITY='SAN FRANCISCO')`

- `NAME LIKE 'AL%'`

- `NAME LIKE 'ALLE\_'`

If the SQL Filter syntax entered into the property is not correct, an error occurs. The connector waits for the length of time specified in the [Retry Interval on Error](02-configuring-connector-types-and-properties.md#retry-interval-on-error) and then tries to get the changed entries in the database again. After the maximum number of retries (indicated in the [Max Retries on Error](02-configuring-connector-types-and-properties.md#max-retries-on-error) property) is exhausted, if the SQL syntax is still invalid, the connector stops. You must either remove or correct the SQL filter before restarting the connector. It is recommended that you set the connector [Log Level](02-configuring-connector-types-and-properties.md#log-level) to DEBUG and check the connector log for the SQL query ("Executing query:") that is generated to ensure the value entered in the SQL Filter property is translated properly. The capture connector log is located at `\<RLI_HOME\>\logs\sync_agents\\\<pipelineId\>\connector.log` on the RadiantOne node where the sync agent is running. Run `\<RLI_HOME\>/bin/monitoring.bat` (`.sh` on Linux) `-d pipeline` to locate your sync process and the value of the `captureHostname` propertyId value indicates the machine where the connector.log is located.

If the connector should process all changed entries, do not enter a SQL filter.

### Force Sequential Counters

This property accepts a value of`true` or `false` and dictates how the connector treats entries it picks up from the LOG table that have non-sequential change IDs. The default is `true` meaning that if the connector detects a non-sequential change ID for an entry in the LOG table, it behaves as if there is an error (non-connection error) and the retry logic based on the Max Retries on Error and Retry Interval on Error properties takes effect. Sometimes rows in the log table are not written in the order of the change ID, and if the connector doesn't wait for the entries to have sequential IDs, some changes could be missed. The connector waits for the length of time specified in the [Retry Interval on Error](02-configuring-connector-types-and-properties.md#retry-interval-on-error) and then tries to get the changed entries in the database again. After the maximum number of retries (indicated in the [Max Retries on Error](02-configuring-connector-types-and-properties.md#max-retries-on-error) property) is exhausted, if it still detects non-sequential change IDs, the connector stops. You can [manually edit the cursor value](01-overview.md#manually-updating-connector-cursor) before restarting the connector to avoid the non-sequential number. Or you can disable the Force Sequential Counters property for the connector.

If the connector should ignore non-sequential change IDs, and process all changes immediately, set the property to `false`.
