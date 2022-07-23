# Database tuning

## Indexes

It can be tempting to add indexes for lots of the tables, but beware, there is database overhead to maintain an index.

e.g., run as PostgreSQL user with write access to the database; `create index idx_data_main9 on R_DATA_MAIN (data_checksum);`

For example, if you are writing lots of checksums (eg you have msiDataObjChksum in an upload rule), you might _not_ want to add an index as the default and it can be added later by a DBA et al.

## No of Connections

As iRODS does not do connection pooling, every connection to iRODS (including admin tasks) creates a new conection to the database, so you need to be sure that your database is configured for a number of connections relative to the amount of traffic you are expecting (and possibly a healthy overhead to be sure).
