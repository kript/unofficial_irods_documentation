# Initial Deployment

<!-- TODO write an introductory paragraph. -->

## Prerequisites

Before making decisions about the iRODS deployment, its system requirements should be well understood. In particular, at least the following information should be gathered.

* Customer
  * performance benchmarks for the system, including the data transfer rate, processing times, and response times for user queries
  * availability requirement of the system - does any of the infrastructure need to be HA in some way?
  * security protocols needed to protect the data, e.g., encryption, access controls, and compliance with regulatory standards (GDPR, HIPAA, etc.)
  * an estimate of how many users (people and services) will have iRODS accounts
* Storage
  * an estimate of how much storage will be needed
  * how and where the data will be stored, i.e, local or remote, multiple copies, etc
  * an estimate of how many metadata values (AVUs) will be stored
* Network
  * an estimate of how many concurrent connections will need to be supported
  * network requirements, including bandwidth, latency, and connectivity, to support the expected data traffic
* Monitoring tools for real-time tracking of system performance and automated reporting for regular assessments
* Maintenance procedures for the system, including regular updates and support structures for users
* Disaster Recovery
  * data back up strategy, e.g., frequency of back ups, methods for data recovery in case of data loss, etc.
  * comprehensive disaster recovery plan to ensure business continuity in case of major disruptions
* Future expansion plan, system scaling for increased data, more users, and/or additional functionalities

## Determining DBMS configuration

iRODS hosts its catalog DB in a relational DBMS. Here is some advice on choosing a DBMS technology, determining what performance specs are required, and configuring it.

### Decide which DBMS to use

PostgreSQL is the DBMS primarily used by RENCI when developing iRODS and performance tuning it. Unless there is a reason to use MySQL or Oracle, like your organization has a significant infrastructure built using one of them, you should use PostgreSQL.

### Determine specs for DBMS host(s)

If the iRODS installation will have a dedicated DBMS, it is important to estimate the required number of cores, how much memory and storage will be needed, and storage and network performance. It is still important to gather these estimates if the installation will share a DBMS so that the impact on the DBMS can be understood.

!!! note
    For a small zone, the DBMS can be co-hosted with the catalog service provider. In this case, a small zone doesn't have a lot of data objects, lots of metadata attached to its data object, nor computationally intensive iRODS rules that would compete with the DBMS for memory and CPU.

#### Cores/Hyperthreads

The number of cores required depends on the number of concurrent connections. iRODS doesn't have connection pooling, so each connection creates a new database connection. This means that the number of concurrent database connections will be the same as the number of concurrent connections to the iRODS catalog provider. This is not necessarily the same as the number of concurrent user connections, since deferred rule executions also connect to iRODS creating database connections. A good estimate of the expected number of concurrent database connections is the sum of the expected number of concurrent user connections and the configured maximum number of concurrent rule engine processes (`advanced_settings.maximum_number_of_concurrent_rule_engine_server_processes` in `/etc/irods/server_config.json` on the catalog provider).

Sanger has observed that 2 cores can serve over 100 concurrent DB connections and 64 cores can handle at least 2500.

#### Memory

The amount of memory required depends largely on the number of replicas, permissions, and AVU metadata values stored in the database. The entire database needs to be held in memory, or the DBMS will end up having to load records from disk as needed, which will dramatically degrade query performance.

CyVerse has observed that the total size of a PostgreSQL ICAT DB will approximately be `size(data) + size(avu) + size(acl)`, where `size(data) ~ (1000 B) * num_data_obj * num_repl_per_obj`, `size(avu) ~ (200  B) * num_data_obj * num_avu_per_obj`, and `size(acl) ~ (200 B) * num_data_obj * num_perm_per_obj`.

#### Storage

If there is insufficient memory to hold all the entire database, then the DBMS will often be accessing database files, so IOPS is critical. If all the data can be held in memory, IOPS is still important but less so. The blog post [Understanding Postgres IOPS](https://www.crunchydata.com/blog/understanding-postgres-iops) explains why in more detail. IOPS is weakly correlated with the number of concurrent database connections, so expected number of connections can be used to estimate required IOPS.

During a one hour observation period, CyVerse had between 70 and 130 concurrent DB connections (90 on average). The DBMS was performing 140 IOPS on average with bursts of up to 3000.

The storage will need to be able to hold the entire database as well write-ahead logs and DBMS bookkeeping information. The database files tend to grow larger over time due to rows in the middle of the file becoming invalidated due to deletes and updates. (Eventually these rows will be overwritten.) This growth will need to be accounted for when estimating how much storage will be needed.

CyVerse's database is approximately 900 GiB in size, but PostgreSQL is using 1400 GiB of storage.

#### Network

Since the iRODS catalog provider will be making lots a requests to the DBMS with nearly all of these requests and their responses only requiring a small amount of data being transferred, the latency of the network connection between the catalog provider and the DBMS should be as small as reasonably possible.

Network throughput is not that important. Over a one day period, CyVerse observed their catalog database had an average throughput of 4 MiB/s with a peak of 11 MiB/s.

Institutional firewall appliances are known for increasing network jitter. They also have a habit of severing connections that are idle for too long. Some iRODS queries can take a few minutes to execute on a large database. To reduce the likelihood of broken database connections causing problems for users, it is recommended to not place the DBMS and the iRODS catalog provider on opposite sides of an institutional firewall.

## Determining iRODS configuration

<!-- TODO write this section -->

### Deployment topology

<!-- TODO write this section -->

#### Deciding whether to colocate catalog provider and DBMS

In general, it is better to have the catalog provider on a separate server that the DBMS, but there are cases where it makes sense to collocate them as long as the DBMS is on a single host and is dedicated to iRODS.

1. The iRODS zone will be small and lightly used. In other words, the zone won't have a lot of data objects and there won't be more than a few concurrent connections.
1. The catalog provider will not be storing data object replicas locally, and the system is expected to be moderately used -- maybe less than 10 concurrent connections.

#### Deciding if catalog provider should be a resource server.

<!-- TODO write this section -->

* How busy will the iRODS zone be?

  If there will be lots of concurrent connections to the iRODS zone, it would better to offload the storage management responsibilities to a separate iRODS catalog consumer

* Will there be a lot of computationally intensive rule logic?

  If the rule logic will consume a lot of memory on the catalog provider, this will mean less memory for the buffer cache used during file transfers, it might be better to offload storage management to a catalog consumer/resource server.

* Will data be physically stored on the iRODS server or stored remotely, i.e., will it be store on a disk directly attached to the server or somewhere else, like a NAS device or in the cloud?

  If there will be multiple resource servers, in general, it is recommended to have a dedicated catalog provider that does not act as a resource server. This acts as a separation of concerns and allows for the catalog provider to be optimized for concurrency and more interactive network sessions, and the resource servers to be optimized for data transfer.

#### Deciding how many catalog provider hosts to have.

<!-- TODO write this section -->

It is non-trivial to run multiple catalog providers. Unless there is a requirement, such as HA, that demands multiple catalog providers, you should start with one, and scale out if it cannot handle the load. CyVerse has a single catalog provider that can sustain 100+ concurrent connections. They have had spikes of over 300 connections that stress the system unacceptably, so they are planning to move to multiple catalog providers in the future.


### Determine specs for hosts

<!-- TODO write this section -->

Usage of all the below should be monitored over time so that additional capacity can be added when needed

* Catalog provider(s)

  For a catalog provider that is on a separate host from the DBMS and isn't a resource server, the host will primarily act as a network switch yard. It will forward catalog queries and updates to the DBMS and data transfer requests to the appropriate resource server. For small files, data transfers will be routed through the catalog provider. The catalog provider will be the primary executor of any rule logic. This means that catalog provider performance will primarily be bound by concurrency, so it should have lots of cores. Also, its network interface should be tuned to minimize latency. The provider probably won't need a lot of memory or storage unless custom rule logic requires it.  The CyVerse catalog provider has 48 cores. It has 252 GiB of memory. It's way over-provisioned, It only uses ~10 GiB of memory, and even holding nearly 2 years of iRODS log files, it uses less than 400 GiB of disk.

* Resource server(s)

  If the resource server isn't also a catalog provider, it will primarily act as a DTN. [ESnet](https://fasterdata.es.net/DTN/) provides good advice on configuring a DTN.

  * File system

    A resource server that stores its replicas on DAS should have at least 1 GiB of memory for every 1 TiB of storage. (There is a reference for this somewhere. I just need to look it up.) This is to ensure that there is sufficient memory for the file system cache.

  * Cacheless S3

    A resource server that hosts a cacheless S3 resource should have storage available, since this resource temporarily stores its files locally before transferring to S3. The amount of storage required will depend on the volume of data in flight between the server and S3. A way to get an initial estimate would be to estimate S, the average size of the file that will be stored in S3, and estimate N, the expected number of concurrent requests for data on that resource at peak busy times. An initial storage size estimate would be S*N.

## Deploying and configuring iRODS Zone

<!-- TODO write this section -->

### Unattended installation (See _Unattended Install_ on the [Installation - iRODS Docs](https://docs.irods.org/4.3.1/getting_started/installation/))

<!-- TODO write this section -->

Deploying using a JSON file rather than scripting the responses to the setup questions is generally good practice because;

* It provides a way to version control the setup, which in turn allows CI style checks for linting, templating and verification.
* It is easier to integrate with deployment tools such as cloud-init, ansible, terraform and so on.

### Ansible or other provisioning tool

<!-- TODO write this section -->

Using configuration management with your iRODS systems is recommended to prevent changes made on one server not be applied to others on the zone. Ansible is used within the community

### Version locking/pinning

At this time, iRODS versions are not fully backwards compatible, even between point releases. To prevent an automatic update of system packages from breaking your iRODS zone, it is recommended that the installed version of iRODS be locked/pinned.

### Controlling when iRODS starts

The start order of iRODS processes is important. If they are started in the wrong order, some of them will fail to start. The DBMS must be started before the iRODS catalog service providers, and the catalog service providers must be started before the catalog resource consumers. To prevent the services from accidentally be started in the wrong order when the processes are spread over multiple hosts, it is not recommended to have them start automatically when the host is started. Instead, manually start them using a provisioning tool to ensure the processes are brought up in the correct order.

### DNS usage

<!-- TODO write this section -->

iRODs makes _enormous_ numbers of DNS calls. In 4.2.9 and later it can do a better job of caching, but it is recommended that your connection to the DNS system is low latency and/or that all the servers in the zone, including database servers, have entries in the /etc/hosts or /etc/irods/hosts files to prevent it reaching out to the DNS. Issues seem from slow or missed DNS lookups have been SYS_HEADER_READ_LEN errors where inter-server connections could not be established.

### Firewalls

Institutional firewalls often sever connections that have been idle, i.e., had no packet traffic, for a little while. CyVerse has observed this time to be as short as five minutes. When iRODS uses parallel transfer, the primary connection will be idle during the transfer. If the transfer takes long enough, a firewall may sever the primary connection, causing the transfer to fail. CyVerse has observed It is recommended to set the TCP keep-alive time to something like 2 minutes.

## Advice on choosing iRODS clients

<!-- TODO write this section -->

Start with the ones published by RENCI: iCommands and MetaLNX. If they don't meet all of your user's needs, add in Cyberduck. If there are still use cases that aren't supported, branch out to other clients like Baton, GoCommands, etc.
