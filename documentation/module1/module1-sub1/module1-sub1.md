---
title: Module 1 Sub 1
description: Module 1 Sub 1 Description
---

# Module 1 Sub 1

Prosciutto ground round drumstick rump pork chop turducken tongue tail turkey porchetta beef ribs ribeye chislic filet mignon shoulder. Pork belly beef ribs burgdoggen, kevin meatloaf turducken tri-tip pork chop t-bone. Filet mignon andouille beef ribs, chicken boudin spare ribs porchetta. Landjaeger doner sausage cow beef, pig ground round jowl pork belly bacon hamburger turkey buffalo ham hock venison.

# Chapter 5: Database Changelog (Triggers) Connector 

When a database object is configured as a publisher, triggers are installed on the object and document all changes to a log table. This object name has the syntax `\<table name\>\_LOG`. In the log table, two predefined column names are required: `RLICHANGEID` and `RLICHANGETYPE`. `RLICHANGEID` uniquely identifies one row in the change log table, and `RLICHANGETYPE` identifies the operation (insert, update, delete, abort). The database connector queries the log table to check for changes based on the polling interval.

The `RLI_CON` user (configurable) is the default owner of the log table. This special user can also prevent a constant loop of changes when the database objects have been configured as both a publisher and subscriber. When this user makes changes to the database objects, the connector knows to ignore the change.

## Database Connector Failover

Database connectors can leverage a failover backend that has been configured for the data source. When you configure a data source for your backend database, indicate a failover server. An example is shown below.

>[!note]
>For SQL Server backends, indicating the failover server in the data source is not required. You only need to indicate the primary server in the data source and the connector can automatically detect the failover/mirrored server.**

![An image showing where to select a Failover Server for the Backend Database](media/image15.png)

Figure 5. 1: Configuring a Failover Server for the Backend Database

If a connection cannot be made to the primary server, and [max retries is exhausted](02-configuring-connector-types-and-properties.md#max-retries-on-connection-error), the connector connects to the failover server.

>[!important]
>The backend servers must be configured for replication/mirroring. Please check the vendor documentation for assistance with configuring replication for your backends.

For SQL Server backends, make sure the database connector user exists on the mirrored server. User logins created on a database server are at the instance level and are not automatically propagated to mirrored instances (because mirroring works at the database level).

The method to fix the problem of the database connector user not existing on the mirrored server is to create a login in the mirrored server using the same SID of the user which already exists in the principal server.

To find the user's sid, use the query below on the principal server:

`Select sid from dbo.sysusers where name='rli_con';`

To create the login on the mirrored server, use the query below (with
the sid retrieved from the principal server):

`Create login rli_con with password ='password'  
sid=0xC82A450E586CCC4CADDD802E9D9CA404;`

>[!important]
>In the case where the principal/primary SQL Server is not available, the connector fails over to the mirrored server. Once the principal server is back online, the connector continues to read from the mirror (it essentially becomes the new principal server) until the mirror is not available, in which case it reverts to the original primary server.

## Configuration

To detect changes using Changelog (triggers), set the connector type in the pipeline configuration by clicking the **Capture** component. In the Core Properties section, select DB Changelog from the drop-down list.

![An image showing the drop-down list for Connector Type with DB Changelog selected, in the Core Properties section of Configure Pipeline](media/image16.png)

Figure 5. 2: DB Changelog Connector Type

After the DB Changelog connector type has been selected and configured, you can configure the properties in the Core Properties, Advanced Properties, and [Event Content](02-configuring-connector-types-and-properties.md#event-contents) sections at the bottom. For properties common to all connectors, see [Chapter](02-configuring-connector-types-and-properties.md#common-properties-for-all-connectors) 2.

### Log Table User

Enter the user name for the connector's dedicated credentials for connecting to the log table. If you do not have the user name, contact your DBA to get the information to use.

### Log Table User Password

The password for the user configured in the Log Table User property. If you do not have the password, contact your DBA to get the credentials.

### Table Name

The name of the database table where changes to database entries are logged. You can use an existing table by entering the name in this property. Or enter the name of the table to create using the RadiantOne configuration scripts. Be sure to use the proper syntax for your database vendor (e.g. `\<USER\>.\<TABLE\>\_LOG`).

### Apply Configuration to Database

When you save the connector type configuration, you are prompted to execute the scripts on the database server. An example is shown below.

![An image showing a prompt to execute DB Changelog configure scripts with "No" and "OK" buttons](media/image17.png)

Figure 5. 3: Option to Execute Database Scripts

If you use choose "OK" to execute the DB Changelog scripts, `SELECT`, `INSERT`, `UPDATE`, and `DELETE` access are granted for PUBLIC to the LOG table. Check with your database administrator if you need to restrict access rights on the log table. You can choose the "NO" option and the generated script can be reviewed, modified if needed, and run on the database by a DBA. The location of the generated scripts is: `\<RLI_HOME\>/work/sql/\<pipelineId\>`

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

If the SQL Filter syntax entered in the property is not correct, an error occurs. The connector waits for the length of time specified in the [Retry Interval on Error](02-configuring-connector-types-and-properties.md#retry-interval-on-error) and then tries to get the changed entries in the database again. After the maximum number of retries (indicated in the [Max Retries on Error](02-configuring-connector-types-and-properties.md#max-retries-on-error) property) is exhausted, if the SQL syntax is still invalid, the connector stops. You must either remove or correct the SQL filter before restarting the connector. It is recommended that you set the connector [Log Level](02-configuring-connector-types-and-properties.md#log-level) to DEBUG and check the connector log for the SQL query ("Executing query:") that is generated to ensure the value entered in the SQL Filter property is translated properly. The capture connector log is located at `\<RLI_HOME\>\logs\sync_agents\\\<pipelineId\>\connector.log` on the RadiantOne node where the sync agent is running. Run `\<RLI_HOME\>/bin/monitoring.bat` (`.sh` on Linux) `-d pipeline` to locate your sync process and the value of the `captureHostname` propertyId value indicates the machine where the connector.log is located.

If the connector should process all changed entries, do not enter a SQL filter.

### Force Sequential Counters

This property accepts a value of `true` or `false` and dictates how the connector treats entries it picks up from the LOG table that have non-sequential change IDs. The default is `true` meaning that if the connector detects a non-sequential change ID for an entry in the LOG table, it behaves as if there is an error (non-connection error) and the retry logic based on the Max Retries on Error and Retry Interval on Error properties takes effect. Sometimes rows in the log table are not written in the order of the change ID, and if the connector doesn't wait for the entries to have sequential IDs, some changes could be missed. The connector waits for the length of time specified in the [Retry Interval on Error]
