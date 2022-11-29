# Automatic-Creation-of-Indices-in-PostgreSQL
Applications perform poorly when transactions are executed over relations with no
indices. Operations like joining two large relations, which ideally takes seconds,
may take hours without proper indices. Initially, database administrators create the
necessary indices on relations after investigating potential data access patterns,
but occasionally in the real world, access patterns can differ, resulting in slow
response times because there is no index on the attribute the query is attempting
to access. To close this gap, the database should create indices on frequently
accessed attributes on the fly if the index was not already present. Thus
unexpected data access patterns doesn’t slow down the performance of the
application.

## Project Overview
The project has three components.
1. Gathering the information required for index creation. We must keep track
of the most used attributes of a relation, the information will be used to
decide if an index should be created over an attribute.
2. Creating a daemon process that starts when the server starts and
terminates cleanly when the server stops. This can be done when ```pg_ctl
start``` and ```pg_ctl stop``` are invoked.
3. The daemon process should periodically scan the information gathered in
Step 1 and create the index when some conditions are met.

## Storing the attribute usage count
We track the number of sequential scans performed on a relation. The sequential
scan must have a qualifier, i.e., tuples are filtered based on some condition. The
attribute involved in the qualifier, the relation name, and the usage count is stored
in a relation called ```pg_att_use_count```.
* Obtaining the relation name

  The relation name can be obtained from the ```SeqScanState``` structure
initialized in ```ExecInitSeqScan```.
* Obtaining the ordinal position of the attribute

  The ordinal position or the column number of the attribute can be found in
the ```qual``` node of the query plan.
* Getting the attribute name

  Given the relation name and the ordinal position, we can use the
```information_schema.columns``` view to get the attribute name.
* Updating the usage count

  The use count is updated in the pg_att_use_count relation using the relation
and the attribute name.



## Using ```pg_ctl``` to execute and terminate the daemon process
* Executing the daemon process

When the server is started using ```pg_ctl start```, a child process is forked, and a
daemon process called ```index_creator``` is executed in a child process. This daemon
will periodically scan the ```pg_att_use_count``` relation and create the required
indices.
* Terminating the daemon process

When the server is stopped using the ```pg_ctl stop```, a SIGTERM signal is sent
from pg_ctl to the ```index_creator``` daemon. The daemon process closes the
database connection, frees any resources acquired and exits when the SIGTERM
signal is received.

## The Daemon Process
The daemon process connects to the database and scans the pg_att_use_count
relations. For each tuple (representing the use count of an attribute in a relation),
the estimated size of the target relation is found via pg_classes, if the estimated
size times the use count is greater than some threshold, an index is created over
the attribute.
After the scan is completed, the daemon process goes to sleep for some
fixed duration, and the whole process is repeated.

## Requirements
* Postgres 14.5
* ```libpq-fe``` library (sudo apt-get install libpq-dev)
