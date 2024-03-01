---
title: PostgreSQL HA on Docker with pg_auto_failover
description: Create a highly available infrastructure for databases and applications, We'll focus on setting up HA PostgreSQL on Docker, using the pg_auto_failover.
date: 2023-06-28T15:40:00.000Z
last_updated: 2024-02-27
tags:
    - Docker
    - HA
    - Postgresql
categories:
    - Docker
    - Database
draft: true
preview: /img/banners/pg_autofailover_banner.jpg
image: /img/banners/pg_autofailover_banner.jpg
---

##  PostgreSQL HA on Docker with pg_auto_failover

We are going to discuss how we set up high availability across our network. This will be split up to several parts which touch various components that I personally run in my networks. 

All the commands here are run on Ubuntu 22.04 server minimal and use the default values. All machines need to have docker running, though you can also just run parts of it dockerised and others on physical machines.
<!-- FM:Snippet:Start data:{"id":"Affiliate warning","fields":[]} -->
> Some links may be affiliate links that keep this site running.
{: .prompt-info}
<!-- FM:Snippet:End -->

### Introduction
Some of my selfhosting is done for business purposes, and to offer services according to my own motto of "service excellence" I decided it is time to build a highly available infrastructure for the databases and applications that serve the business.
The solution has been build with HAProxy, Cloudflare, Netbird, PostgreSQL, repmgr, MariaDB and Galera Cluster.
<!-- FM:Snippet:Start data:{"id":"VPS Links","fields":[]} -->
<div class="alert alert-info" style="background-color: lavender; border: none; color: black;">
  <!--<i class="fas fa-info-circle"></i><br>--> I host majority of my cloud instances on <a href="https://cloud.hosthatch.com/a/3785" target=_blank>HostHatch VPS (Virtual Private Server) Instance</a> (In Asia) for a steal. Some of the other hosts I use are <a href="https://my.racknerd.com/aff.php?aff=9825" target=_blank>RackNerd</a> (US) and <a href="https://clients.webhorizon.net/?affid=27" target=_blank>WebHorizon</a> (Asia+Europe) VPS, and decided that it is time to move away from Linode - which is a Great service, but I am looking to reduce the billing on instances. For comparison, I save more than 50% on <a href="https://cloud.hosthatch.com/a/3785" target=_blank>HostHatch</a> compared to <a href="https://www.linode.com/lp/refer/?r=1ba3d23e62b7803c1d317de14a21a9f96a9a1b97" target=_blank>Linode</a> ($3.33 compared to $8) - Don't get me wrong, if this was an extremely (like REALLY) critical application, I would keep it on <a href="https://www.linode.com/lp/refer/?r=1ba3d23e62b7803c1d317de14a21a9f96a9a1b97" target=_blank>Linode</a>.
</div>
<!-- FM:Snippet:End -->

Today I'm going to cover setting up HA PostgreSQL on Docker with the help of pg_auto_failvoer - it is a postgres extension that works on monitoring the database and redistributing the master/replica roles as well as initiating the processes to bring replicas up to day. The docker image that is used has PostgreSQL together with the extentions necessary.

You can also find more information at the [pg_auto_failover docs][pg_auto_failover].
I preferred the image by livindocs as it seems more maintained and updated compared to the citusdata one - for the docker image documentation, check out the [docker hub repository][livingdocs_postgres_docker]

### Install NetBird
Learn here how to selfhost Netbird, otherwise you can install tailscale or other preferred overlay networks.
Enable UFW to allow connections only from Netbird interface, once you run the last command sudo ufw enable tou may get dropped off if you are not connected to the Netbird network. You will have to reconnect again, but this time to the Netbird IP of the machine, so make sure to note or retrieve your Netbird IP beforehand.

```shell
sudo ufw allow in on wt0
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw route allow from 100.64.0.0/10 to any
sudo ufw enable
```
{: .nolineno }

### Initialise monitor

You're going to need to have a "monitor", that machine is going to check the status and sync all of your postgresql instances together. I haven't managed to figure out how to make it listen and use the hostname of the host it is currently on right now though and so you will be mapping the network mode to "host" in the docker file and commands.

For those that use k8s and docker swarm, it should work easier as you can just reference internal hostnames and use the internal resolver services to reach the various containers/pods.

> If you are not using swarm or k8s, or your name resolution is not configured properly, use the `/etc/hosts` to populate a mapping of IP addresses and hostnames.
{: .prompt-info }

We are going to create our docker directory and change to the correc priviliges just in case.
<!-- FM:Snippet:Start data:{"id":"Shell code block","fields":[]} -->
```shell
sudo mkdir /docker/postgresql/db -p
sudo chown 1000:1000 /docker/postgresql/db
cd /docker/postgresql
```
{: .nolineno }
<!-- FM:Snippet:End -->

This has created a directory in the path `/docker/postgresql/db` and this is where the monitor is going to set up it's own separate database where the state of the cluster is going to be kept. Let's switch to that director.
We need to first initialize the monitor and pull the container, to do so we need to run the following command. Don't worry where you run it as long as you replace the folders to the correct values it should be fine.

<!-- FM:Snippet:Start data:{"id":"Shell code block","fields":[]} -->
```shell
docker run --rm \
  --network host \
  -e PGDATA=/var/lib/postgresql/15/data \
  -e PG_AUTOCTL_DEBUG=1 \
  -e XDG_CONFIG_HOME=/var/lib/postgresql/.xdg_config \
  -e XDG_DATA_HOME=/var/lib/postgresql/.xdg_data \
  -v /docker/postgresql/db:/var/lib/postgresql \
  --name monitor \
  livingdocs/postgres:15.3 \
  pg_autoctl create monitor --ssl-self-signed --skip-pg-hba --run
```
{: .nolineno }
<!-- FM:Snippet:End -->

!["Initialise pg_auto_failover monitor"](/img/posts/postgres/pg_auto_failover_-_initializing_monitor-min.gif)
_Initialise pg_auto_failover monitor_

Once the container and the data set is initialized you can stop the container and remove it, we do not have a need for it anymore. We now need to edit the `pg_hba.conf` file to allow access from other machines in our network.
```shell
user@monitor:/docker$ nano /docker/postgresql/db/15/data/pg_hba.conf 
```
{: .nolineno }

Append to the end of the document the following lines:
```text
hostssl pg_auto_failover autoctl_node   100.64.0.0/10           trust
```
{: .nolineno }
{: file="/docker/postgresql/db/15/data/pg_hba.conf"}

* `hostssl` specifies that the connection to your PostgreSQL database should be made over SSL, a secure method of communication that encrypts the data.
* `pg_auto_failover` is a service used to automate the failover procedure. Failover is a backup operational mode in which the functions of a system are assumed by secondary system components when the primary system becomes unavailable.
* `autoctl_node` is a specific user who is given control of the automated failover process.
* `100.64.0.0/10` represents a range of Netbird IP addresses. These are the devices that are permitted to connect to your PostgreSQL database.
* `trust` means that the system trusts all the incoming connections from the specified IP range and doesn't require a password for them.

The entire line is essentially allowing secure, password-free connections to your database's failover system from a specific range of IP addresses.

### Set up a compose file for monitor
Once we are done with initializing the monitor, lets set up a docker-compose file that will run the monitor and restart when needed.

```yaml
version: "3.9"
services:
  monitor:
    image: livingdocs/postgres:15.3
    network_mode: host
    restart: unless-stopped
    environment:
      PGDATA: /var/lib/postgresql/15/data
      XDG_CONFIG_HOME: /var/lib/postgresql/.xdg_config
      XDG_DATA_HOME: /var/lib/postgresql/.xdg_data
    command: pg_autoctl run
    volumes:
      - /docker/postgresql/db:/var/lib/postgresql/
```
{: file="/docker/postgresql/docker-compose.yaml"}

Bring it up with `docker compose up -d` and you can check the logs with `docker compose logs -f monitor`

![Starting a postgresql monitor instance](/img/posts/postgres/pg_auto_failover_-_starting_monitor-min.gif)
_Starting a postgresql monitor instance_

### Initialise main database containers

> The first database that you are going to initalize is going to be the master and all the configuration is going to be copied from it to the replicas.
{: .prompt-info }

>You don't have to initalize an empty container with an empty database, you can map the docker to an existing database to make it highly available and automated failover, just note:
- When you initalize replicas the folders will be purged.
- You set the permissions correctly.
- You edited the pg_hba.conf file correctly.
{: .prompt-tip}

This needs to be run on every machine you'd like to have a replica on, please keep that in mind. You can run multiple databases there is no issue with that - that is explained at the end.
To initialize the database we are going to run the command below.

> Remember to replace in the docker command the following: `PG_AUTOCTL_NODE_NAME` - Name of your db node
`PG_AUTOCTL_CANDIDATE_PRIORITY`- Priority of database
`PG_AUTOCTL_MONITOR` - Connection string to your monitor. You can find it in the log of the monitor docker container
{: .prompt-warning}

```yaml
docker run --rm\
  --network host \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=password \
  -e PGDATA=/var/lib/postgresql/15/data \
  -e PG_AUTOCTL_NODE_NAME=SG-DB-1 \
  -e PG_AUTOCTL_CANDIDATE_PRIORITY=100 \
  -e XDG_CONFIG_HOME=/var/lib/postgresql/.xdg_config \
  -e XDG_DATA_HOME=/var/lib/postgresql/.xdg_data \
  -e PG_AUTO_CTL_DEBUG=1 \
  -e PG_AUTOCTL_MONITOR="postgres://autoctl_node@TOK-Monitor:5432/pg_auto_failover?sslmode=require" \
  -v /docker/postgresql/db:/var/lib/postgresql \
  --name postgresql_ha_main \
  livingdocs/postgres:15.3 \
  pg_autoctl create postgres --ssl-self-signed --skip-pg-hba --username postgres --formation default --run
```
{. file="/docker/postgresql/docker-compose.yaml}

![Initialise database docker run command](/img/posts/postgres/pg_auto_failover_-_Intializing_database-min.gif)
_Initialise database docker run command_

To make sure it works, you will see at the end of the log on the primary:
```text
pg_autoctl service is running, current state is "single"
```
{: .nolineno }

The monitor will show you the following:
```text
INFO New state for node 1 "database name" (databasename:5432): single -> single
```
{: .nolineno }

You can stop the container as we need to do another small change here. you need to edit the `pg_hba.conf` file once more, this time it is for the machines that are going to connect to the primary for replication. After you edit the file, append the following lines at the end:

```text
hostssl "postgres" "postgres" 			100.64.0.0/10 trust
hostssl all "pgautofailover_monitor" 	100.64.0.0/10 trust
hostssl replication "pgautofailover_replicator" 100.64.0.0/10 trust
hostssl "postgres" 	"pgautofailover_replicator" 100.64.0.0/10 trust
host	all		all		100.64.0.0/10 md5
```
{: .nolineno }

### Starting up the postresql container

Let's create the docker compose file to run the container, you'll need to replicate the file configuration with the appropriate changes as those are mentioned above.
```yaml
version: "3.9"
services:
  postgresql_ha_default:
    image: livingdocs/postgres:15.3
    network_mode: host
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      PGDATA: /var/lib/postgresql/15/data
      PG_AUTOCTL_NODE_NAME: hk-db-1
      PG_AUTOCTL_CANDIDATE_PRIORITY: 100
      XDG_CONFIG_HOME: /var/lib/postgresql/.xdg_config
      XDG_DATA_HOME: /var/lib/postgresql/.xdg_data
      PG_AUTO_CTL_DEBUG: 1
      PG_AUTOCTL_MONITOR: "postgres://autoctl_node@SG-Gateway:5432/pg_auto_failover?sslmode=require"
    command:  pg_autoctl run
    volumes:
      - /docker/postgresql/db:/var/lib/postgresql/
```
{: file="/docker/postgresql/docker-compose.yaml"}

We are going to bring this up now:
```shell
docker compose up -d && docker compose logs -f
```
{: .nolineno }

![Bringing up the database](/img/posts/postgres/pg_auto_failover_-_Bringing_up_database-min.gif)
_Bringing up the database_

> If at any point you get the error:<br>
`FATAL Failed to find a local IP address, please provide --hostname.`
<br>That means that your name resolution is not set up properly.
{: .prompt-danger }

Now continue running this section on all the machines you are looking to add as replicas, you can check the monitor log to see how those are added in. Make sure that your replica stabilizes before you terminate the initial docker initialization command. you will usually see a line that says `Transition complete: current state is now "secondary"`

### Verifying
You can monitor the logs of the monitor container and bring down your primary, you will see that after a bunch of checks, your primary will be demoted and a new one will be brought up instead of it.

![pg_auto_failover terminal failover example](/img/posts/postgres/pg_auto_failover_-_Failover_example-min.gif)
_pg_auto_failover terminal failover example_

And we are done! Hooray!

> It is important to remember that only the primary aceepts write queries in this configuration and so you will need to know who your primary is first before connecting to it. The standbys will only allow read queries.
{: .prompt-tip}

### Extra: Creating formations

You can have multiple clusters managed by the monitor. That is done with the use of formations. If you check the initialization of the database, there is a variable named --formation, it is set to default which is the initial formation you will have.

You will need to create more formations first on the monitor before you can actually add those in the initialization.
You can find more information about it at the [documentation page][pg_autoctl formations].

[pg_auto_failover]: https://github.com/hapostgres/pg_auto_failover?ref=selfhosted.club
[livingdocs_postgres_docker]: https://hub.docker.com/r/livingdocs/postgres?ref=selfhosted.club
[pg_autoctl formations]: https://pg-auto-failover.readthedocs.io/en/main/ref/pg_autoctl_create_formation.html?ref=selfhosted.club
