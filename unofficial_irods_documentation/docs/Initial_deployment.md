# Initial Deployment

_Here's an outline of our initial deployment documentation.  For each decision, we should provide some heuristics and/or advice to help the reader make the decision._

## PREREQUISITE: Use the system requirements to generate the following information.__

* Decide where and how the data will be stored. (local or remote, multiple copies, etc)
* Estimate how much storage will be needed.
* Estimate home many users (people and services) will have iRODS accounts.
* Estimate how many metadata values will be stored
* Estimate how many concurrent connections there will be.
* What is the availability requirement of the system - does any of the infrastructure need to be HA in some way?
* Determine the security protocols needed to protect the data. This includes encryption, access controls, and compliance with regulatory standards (e.g., GDPR, HIPAA).
* Establish a strategy for data backup. Decide on the frequency of backups and the methods for data recovery in case of data loss.
* Define the performance benchmarks for the system, including the speed of data retrieval and upload, processing times, and response times for user queries.
* Plan for future expansion. Determine how the system will scale with increased data, more users, or additional functionalities.
* Outline the maintenance procedures for the system, including regular updates and support structures for users.
* Develop a comprehensive disaster recovery plan to ensure business continuity in case of major disruptions.
* Assess the network requirements, including bandwidth, latency, and connectivity, to support the expected data traffic.
* Implement monitoring tools for real-time tracking of system performance and automated reporting for regular assessments.

## Determining DBMS configuration

* Decide which DBMS to use. (MySQL, Oracle or PostgreSQL)

  PostgreSQL is the DBMS primarily used by RENCI when developing iRODS and performance tuning it.  Unless there is a reason to use MySQL or Oracle, like your organization has a significant infrastructure built using one of them, you should use PostgreSQL.

* Determine specs for DBMS host(s).

  * For a small zone, a small VM to host the database can be enough to start as long as you are monitoring the CPU, disk, memory and network utilization so you know when to upgrade.

  * No of cores/hyperthreads requirements according to number of connections. iRODS doesn't have connection pooling, so each connection creates a new database connection.

  * Memory requirement relates to size of metadata and records in the database - the larger the no of records, the more the database will need to hold in memory, OR will end up with a lot of disk IO

    CyVerse has observed that the total size of the database is dominated by the total number of replicas, AVUs, and permissions. The total size of a PostgreSQL ICAT will approximately be `size(data) + size(avu) + size(acl)`, where `size(data) ~ (1 kiB) * num_data_obj * num_repl_per_obj`, `size(avu) ~ (200  B) * num_data_obj * num_avu_per_obj`, and `size(acl) ~ (200 B) * num_data_obj * num_perm_per_obj`.

  * Disk requirements - if there is insufficient memory to hold all the data then the database will often be accessing the database files, so IOPS is important. If all the data can be held in memory, IOS is less important apart from speed of starting & backing up.

  * Network requirement is more impacted by latency than speed, since the database holds records rather than data.

    * N.B. PostgreSQL has a max connection limit

  * Usage of all of the above should be monitored over time so that additional capacity can be added when needed.

    * It is worth monitoring query latency as well, so trends can be seen. A simple starter might be to run a known consistent query and graph the results over time.

## Determining iRODS configuration

* Deployment topology

  * Deciding whether catalog provider should be on the same server as the DBMS

    In general, it is better to have the catalog provider on a separate server, but there are cases where it makes sense to collocate the catalog provider and the DBMS as long as the DBMS is on a single host and is dedicated to iRODS.

    * The iRODS zone will be small and lightly used. In other words, the zone won't have a lot of data objects and there won't be more than a couple of concurrent connections.
    * The catalog provider will not be storing data object replicas locally, and the system is expected to be moderately used -- maybe less than 10 (??) concurrent connections.

  * Deciding how many catalog provider hosts to have.

    It is non-trivial to run multiple catalog providers. Unless there is a requirement, such as HA, that demands multiple catalog providers, you should start with one, and scale out if it cannot handle the load. CyVerse has a single catalog provider that can sustain 100+ concurrent connections. They have had spikes of over 300 connections that stress the system unacceptably, so they are planning to move to multiple catalog providers in the future.

  * Deciding if catalog provider should be a resource server.

    * How busy will the iRODS zone be?

    If there will be lots of concurrent connections to the iRODS zone, it would better to offload the storage management responsibilities to a separate iRODS catalog consumer

    * Will there be a lot of computationally intensive rule logic?

    If the rule logic will consume a lot of memory on the catalog provider, this will mean less memory for the buffer cache used during file transfers, it might be better to offload storage management to a catalog consumer/resource server.

    * Will data be physically stored on the iRODS server or stored remotely, i.e., will it be store on a disk directly attached to the server or somewhere else, like a NAS device or in the cloud?

    If there will be multiple resource servers, in general, it is recommended to have a dedicated catalog provider that does not act as a resource server. This acts as a separation of concerns and allows for the catalog provider to be optimized for concurrency and more interactive network sessions, and the resource servers to be optimized for data transfer.

* Determine specs for hosts

  * Usage of all the below should be monitored over time so that additional capacity can be added when needed

  * Catalog provider(s)

    For a catalog provider that is on a separate host from the DBMS and isn't a resource server, the host will primarily act as a network switch yard. It will forward catalog queries and updates to the DBMS and data transfer requests to the appropriate resource server. For small files, data transfers will be routed through the catalog provider. The catalog provider will be the primary executor of any rule logic. This means that catalog provider performance will primarily be bound by concurrency, so it should have lots of cores. Also, its network interface should be tuned to minimize latency. The provider probably won't need a lot of memory or storage unless custom rule logic requires it.  The CyVerse catalog provider has 48 cores. It has 252 GiB of memory. It's way over-provisioned, It only uses ~10 GiB of memory, and even holding nearly 2 years of iRODS log files, it uses less than 400 GiB of disk.

  * Resource server(s)

    If the resource server isn't also a catalog provider, it will primarily act as a DTN. [ESnet](https://fasterdata.es.net/DTN/) provides good advice on configuring a DTN.

    * File system

      A resource server that stores its replicas on DAS should have at least 1 GiB of memory for every 1 TiB of storage. (There is a reference for this somewhere. I just need to look it up.) This is to ensure that there is sufficient memory for the file system cache.

    * Cacheless S3

      A resource server that hosts a cacheless S3 resource should have storage available, since this resource temporarily stores its files locally before transferring to S3. The amount of storage required will depend on the volume of data in flight between the server and S3. A way to get an initial estimate would be to estimate S, the average size of the file that will be stored in S3, and estimate N, the expected number of concurrent requests for data on that resource at peak busy times. An initial storage size estimate would be S*N.

## Deploying and configuring iRODS Zone

* Unattended installation (See _Unattended Install_ on the [Installation - iRODS Docs](https://docs.irods.org/4.3.1/getting_started/installation/)).

  * Deploying using a JSON file rather than scripting the responses to the setup questions is generally good practice because;

    * It provides a way to version control the setup, which in turn allows CI style checks for linting, templating and verification.
    * It is easier to integrate with deployment tools such as cloud-init, ansible, terraform and so on.

  * Ansible or other provisioning tool

    * Using configuration management with your iRODS systems is recommended to prevent changes made on one server not be applied to others on the zone. Ansible is used within the community

  * Version locking/pinning

    At this time, iRODS versions are not fully backwards compatible, even between point releases. To prevent an automatic update of system packages from breaking your iRODS zone, it is recommended that the installed version of iRODS be locked/pinned.

  * Controlling when iRODS starts

    The start order of iRODS processes is important. If they are started in the wrong order, some of them will fail to start. The DBMS must be started before the iRODS catalog service providers, and the catalog service providers must be started before the catalog resource consumers. To prevent the services from accidentally be started in the wrong order when the processes are spread over multiple hosts, it is not recommended to have them start automatically when the host is started. Instead, manually start them using a provisioning tool to ensure the processes are brought up in the correct order.

  * DNS usage

    * iRODs makes _enormous_ numbers of DNS calls. In 4.2.9 and later it can do a better job of caching, but it is recommended that your connection to the DNS system is low latency and/or that all the servers in the zone, including database servers, have entries in the /etc/hosts or /etc/irods/hosts files to prevent it reaching out to the DNS. Issues seem from slow or missed DNS lookups have been SYS_HEADER_READ_LEN errors where inter-server connections could not be established.

  * Firewalls

    Institutional firewalls often sever connections that have been idle, i.e., had no packet traffic, for a little while. CyVerse has observed this time to be as short as five minutes. When iRODS uses parallel transfer, the primary connection will be idle during the transfer. If the transfer takes long enough, a firewall may sever the primary connection, causing the transfer to fail. CyVerse has observed It is recommended to set the TCP keep-alive time to something like 2 minutes.

## Advice on choosing iRODS clients

Start with the ones published by RENCI: iCommands and MetaLNX. If they don't meet all of your user's needs, add in Cyberduck. If there are still use cases that aren't supported, branch out to other clients like Baton, GoCommands, etc.
