# chef-postgresql-cluster

This Chef cookbook provides recipes for managing groups of
servers with interdependent PostgreSQL clusters set up
as masters and replication slaves.

# Cluster configuration

## `postgresql_cluster_master`

**should fail if access is not permitted all configured slaves**

* `hostname` - the hostname to which to deploy the master. If this does
  not match the hostname of the node, this directive is ignored
* `version` - (required) the version of PostgreSQL to install
* `cluster` - (required) the name of the cluster

```ruby
postgresql_cluster_master "foo.example.com" do
  version "9.4"
  cluster "my_cluster"
  pg_hba: ...
  conf: ...
end
```

## `postgresql_cluster_slave`

* `hostname` - the hostname to which to deploy the master. If this does
  not match the hostname of the node, this directive is ignored
* `cluster` - (required) the name of the cluster
* `version` - (required) the version of PostgreSQL to install. Fails unless
  there is a previously deployed master with the same name and
  version
* `allow_connections`: `true`*|`false`

```ruby
postgresql_cluster_slave "bar.example.com" do
  version "9.4"
  cluster "my_cluster"
end
```

# User

Users are ignored unless deploying to a master.
User resources fail if a `postgresql_cluster_master` resource with
the same name has not been declared.

* `username`
* `version` - (required) the version of PostgreSQL to install
* `cluster` - (required) the name of the cluster

```ruby
postgresql_cluster_user "bob" do
  version "9.4"
  cluster "my_cluster"
end
```

# Database

* `database`
* `version` - (required) the version of PostgreSQL to install
* `cluster` - (required) the name of the cluster
* `owner` - Default: "postrges"
* `encoding`
* `template`

```ruby
postgresql_cluster_database "my_database" do
  version "9.4"
  cluster "my_cluster"
  owner   "bob"
end
```

# Creating a master

* notices that the cluster is to be created,
* creates the cluster,
* starts the cluster.

# Creating a slave

* notices that the cluster is to be created,
* fails if the master is not available,
* initiates replication,
* starts the cluster.

# Failover of a cluster

* change master's action to `down` or change it's directive to
  `postgresql_cluster_slave`,
* change the chosen slave's directive to `postgresql_cluster_master`, and
  **move the resource before any other resource in the same cluster**
* deploy to the slave which is being promoted to master,
* deploy to other slaves,
* deploy to the master, if it is accessible.

# What the cookbook does

on an old slave when switching to `master`:

* notices the cluster exists and is acting as replication slave,
* applies any configuration changes and restarts if necessary,
* touches the failover flag file to halt recovery and switch to being
  master.

on the old master when switching to `down`:

* stops the cluster,
* applies any configuration changes.

on the old master when switching to `slave`:

* notices the cluster exists and is acting as a master,
* stops the cluster,
* configures the cluster to be a slave,
* initiates replication,
* starts the cluster.

# Similar cookbooks

* https://github.com/express42-cookbooks/postgresql
  * Doesn't handle 9.4 yet,
  * Always does shared memory expansion?
  * Each cluster is set up separately. No management of failover.
* https://github.com/Lispython/pg_cluster-cookbook
